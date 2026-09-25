# Disney+ Video Streaming (On Demand)

**The prompt:** "Design Disney+" — or "design a Netflix-style streaming service", or, for streaming teams, "design the playback path / the manifest service". It's the most-asked streaming question anywhere, and prep guides list the playback path for Disney streaming roles. What it tests: splitting the problem into an **offline content pipeline** and an **online playback path**; knowing why video ships as encrypted HTTP segments from CDNs; and reasoning about time to first frame, DRM and entitlements. Related: [16 · stream limits](16_Concurrent_Stream_Limits.md), [18 · traffic spikes](18_Premiere_Traffic_Spike.md), [30 · subscriptions & billing](30_Subscriptions_Billing.md).

## Clarify first

**Functional**

- Play movies and episodes on TVs, phones, web and consoles; resume where you left off (tracker #14).
- Quality by plan (US): Basic/Standard up to 1080p; Premium adds 4K UHD, HDR and Dolby Atmos — and only on devices that can protect it.
- Multiple audio languages and subtitles.
- Downloads (Premium only in the US): up to 10 devices; content stays while subscribed and while the device reconnects at least every 30 days.
- Plan limits: 2 simultaneous streams on Basic/Standard, 4 on Premium ([note 16](16_Concurrent_Stream_Limits.md)); ads on Basic (tracker #37).
- Titles play only in licensed countries and date windows ([note 32](32_Catalog_Rights_Availability.md)).

**Non-functional — say numbers**

- Scale: Disney+ and Hulu had 195.7M subscriptions at their last report (Sept 2025). Assume ~10M concurrent streams at a normal peak; premieres go higher ([note 18](18_Premiere_Traffic_Spike.md)).
- Video start time p50 ≈ 1 s, p95 ≤ 2–3 s; rebuffering well under 1% of watch time.
- The playback path gets 99.99%; everything else degrades before it does.
- Content protection is contractual: DRM everywhere, hardware-backed DRM and HDCP for HD/4K, forensic watermarking for high-value early releases.
- Cost: bandwidth dominates, so CDN offload (the cache-hit ratio) is a first-class requirement.

**Say what's out of scope:** browse and recommendations (#26), search (#27), live ([note 19](19_Live_Sports_Streaming.md)), billing ([note 30](30_Subscriptions_Billing.md)), ad decisioning (#37).

## Back-of-envelope

- **Egress:** 10M streams × ~5 Mbps average = **50 Tbps**. No origin can serve that — CDNs do, and every 1% of cache misses is 0.5 Tbps landing on your origin.
- **Segment requests:** with 6 s segments, each stream fetches about one video and one audio segment per 6 s ≈ 0.33 req/s → **~3.3M req/s** at CDN edges.
- **Playback starts:** 10M concurrent ÷ ~45-minute average session ≈ 3,700 starts/s at a normal peak. A premiere multiplies that 10–50×, so size the playback and license services for ~100K starts/s bursts.
- **Heartbeats:** 10M streams ÷ 30 s ≈ 330K/s (renew stream leases, save progress, report quality).
- **Storage:** one codec family's ladder sums to ~20–50 Mbps, roughly 10–20 GB per hour of content. A ~100K-hour library × ~15 GB × 3 codec families ≈ **4–5 PB**, plus much bigger mezzanine masters. Storage is cheap next to egress.

## API

```text
POST /v1/playback/sessions
  { titleId, profileId,
    device: { platform, drm, securityLevel, hdcp, codecs, maxResolution, hdr },
    audioLang?, subtitleLang? }
  200 { sessionId, manifestUrl (signed, short-lived), licenseUrl,
        startPositionMs, cdns: [ { name, baseUrl } ], heartbeatIntervalSec }
  403 { reason: NOT_ENTITLED | NOT_AVAILABLE_IN_REGION | MATURITY_BLOCKED }
  409 { reason: STREAM_LIMIT_REACHED, activeStreams: [ ... ] }    (note 16)

POST   /v1/licenses/{widevine|playready|fairplay}
       DRM challenge (bytes) → license (bytes)
POST   /v1/playback/sessions/{id}/heartbeat
       { positionMs, bitrateKbps, bufferMs, cdn }
DELETE /v1/playback/sessions/{id}              stop → frees the stream slot

GET {cdn}/t/{titleId}/{renditionSet}/master.m3u8   (or manifest.mpd — signed)
GET {cdn}/t/{titleId}/{renditionSet}/v/hevc_2160p/seg_00123.m4s

POST /v1/downloads              { titleId, profileId, deviceId, quality }
                                → { downloadId, manifestUrl, licenseUrl }
POST /v1/downloads/{id}/license renew the persistent license
                                (re-checks plan and rights)
```

The playback API is a **control plane**: it returns URLs and decisions, never video bytes. The CDN is the data plane. Scaling the two separately *is* the design.

## Data model

| Entity | Key fields | Store |
|---|---|---|
| Title | titleId, type, seriesId, season/episode, durationMs, maturityRating, localized metadata | Document store + heavy caching |
| MediaAsset | titleId, version (theatrical, edited), renditionSetId, codecs, HDR formats, audio and subtitle tracks, status | Relational — it's a graph of versions and components |
| Rendition | renditionSetId, codec, resolution, bitrate, keyId, path | Relational; manifests are generated from it |
| ContentKey | keyId, titleId, tier (SD, HD, UHD, audio), wrapped key | KMS/HSM — never in an app database in the clear |
| Availability | titleId, territory, start, end, tiers, platforms, downloadAllowed | Relational source → per-territory in-memory snapshots |
| PlaybackSession | sessionId, account, profile, device, titleId, cdn, lastHeartbeat | KV with TTL (Redis/DynamoDB) |
| Download | downloadId, account, deviceId, titleId, licenseExpiry | KV; a per-account device set capped at 10 |

**Separate keys per quality tier** is the detail that impresses: a device licensed only for HD can fetch the 4K segments from the CDN, but it can't decrypt them.

## Architecture

**Offline — getting a title ready**

```text
 Studio master (IMF / ProRes)
    │  multipart upload, checksum
    ↓
 Ingest + QC ──→ Encode planner ──→ Transcode workers
                 (per-title ladder) (chunked, parallel)
                                            │
                                            ↓
                  Packager: CMAF segments + HLS/DASH manifests,
                  encrypted with per-tier keys from the KMS/HSM
                                            │
                    ┌───────────────────────┴───────────────┐
                    ↓                                       ↓
       Origin storage ──→ CDNs                    Catalog: title playable
       (pre-position hot titles)
```

**Online — pressing Play**

```text
 Player ──1──→ Gateway ──→ Playback service ─┬─→ Entitlements (plan, tier)
   ↑                                         ├─→ Availability (country, window)
   │                                         ├─→ Stream limits (note 16)
   │                                         ├─→ Bookmarks (resume point)
   │                                         └─→ CDN selector + URL signer
   │←── sessionId, signed manifest URL, license URL, ranked CDNs
   │
   ├──2──→ CDN edge ──miss──→ origin shield ──→ origin   manifest + segments
   ├──3──→ License service ──→ key store (KMS/HSM)       content keys
   └──4──→ Heartbeats + QoE events ──→ Kafka ──→ leases, bookmarks,
                                                 CDN steering, alerts
```

**From Play to the first frame**

1. The player calls `POST /playback/sessions` with the device's capabilities.
2. The playback service checks cheap, local things first (token signature, availability snapshot), then fans out in parallel: entitlements, a stream slot, the resume point. It picks the rendition set (device capabilities ∩ plan tier), ranks CDNs, signs the manifest URL and returns.
3. The player fetches the manifest from the CDN and the license from the license service **in parallel** — the key IDs are known up front, so the license doesn't wait for the manifest.
4. It starts at a conservative bitrate (or the last one that worked on this device), fetches the init segment and the first media segments, decrypts them in the device's DRM module (CDM) and renders.
5. From then on: a heartbeat every ~30 s, an ABR decision every segment, progress saved.

| Step | Rough cost | How to cut it |
|---|---|---|
| DNS + TLS to the API | 50–150 ms | Warm connections at app launch; HTTP/2 or HTTP/3 |
| Playback API | 50–150 ms | Parallel fan-out, in-memory snapshots, no synchronous database writes |
| Manifest | 30–100 ms | Edge-cached; prefetched on the title's details page |
| License | 50–200 ms | In parallel with the manifest; prefetched on the details page; regional license servers |
| First segments | 100–500 ms | Start at a lower bitrate; CDN hit; short first segment |
| Decode + render | 50–100 ms | — |

## Deep dives

### 1. Encoding and packaging

- **The ABR ladder:** each title is encoded at several resolution/bitrate points so the player can switch as bandwidth changes. **Per-title encoding** (Netflix, 2015) picks the ladder from the title's complexity — an animated film needs far fewer bits than a grainy action film for the same quality (measured with VMAF). More compute, less egress, and egress is the bigger bill.
- **Codecs per device:** H.264/AVC everywhere; HEVC for 4K/HDR on TVs; AV1 where supported. HDR10 and Dolby Vision for HDR; AAC, Dolby Digital Plus and Dolby Atmos for audio.
- **Chunked parallel encoding:** split the master at scene/GOP boundaries, encode chunks on many workers, stitch. **Keyframes must line up across every rendition at segment boundaries**, or the player can't switch bitrate mid-stream.
- **Packaging:** CMAF (fragmented MP4) segments of 2–6 s, referenced by both HLS (`.m3u8`) and DASH (`.mpd`) manifests, so segments are stored once. Audio, video and subtitles are separate tracks: one set of video segments serves every audio language.
- **Encryption:** common encryption. `cbcs` mode works for FairPlay, and for Widevine/PlayReady on modern devices; older devices only support `cenc` (AES-CTR), which can force a second encrypted copy for legacy devices.

### 2. DRM and the license service

- **Three DRMs:** Widevine (Android, Chrome, many TVs), PlayReady (Windows, Xbox, many TVs), FairPlay (Apple). The player's CDM sends a license challenge; the license server checks the session (entitled, holding a stream slot), applies policy and returns content keys encrypted for that one device.
- **Policy by device security:** Widevine L1 (keys handled in hardware) for HD/4K; L3 (software only) is capped low. 4K typically also requires HDCP 2.2 on the HDMI output.
- **License shape:** short-lived and renewable for streaming — Widevine's policy has `can_renew`, `renewal_delay_seconds` and `license_duration_seconds` — which is also how [note 16](16_Concurrent_Stream_Limits.md) stops a client that ignores the stream limit. Downloads get persistent licenses with an expiry.
- **It's P0:** stateless, regional, horizontally scaled, with wrapped keys cached in memory. If licenses fail, nothing plays.
- **Forensic watermarking** (per-session A/B segment variants chosen at the edge) traces a leak of a high-value title back to the account.

### 3. CDNs

- **Multi-CDN:** no single CDN outage takes you down; more capacity for premieres; price leverage; different CDNs win in different regions.
- **Steering per session:** rank CDNs by live quality for this country × ISP × device (rebuffer rate, errors, throughput from telemetry), then apply cost and committed-capacity targets. Return an ordered list so the player can **fail over mid-stream** (several base URLs; HLS and DASH both define content steering).
- **Cache hierarchy:** edge → regional mid-tier / origin shield → origin. The shield collapses concurrent misses, so a new release's segment is fetched from origin once per shield rather than once per edge server.
- **Signed URLs:** the edge validates an HMAC token (path prefix + expiry) on every request without calling your backend. Keep auth parameters **out of the cache key**, or every viewer's URL becomes a unique object and the hit ratio collapses.
- **Pre-positioning:** push a big release's popular renditions to the edges before it goes live.

### 4. Adaptive bitrate, on the player

- **Throughput-based:** estimate bandwidth from recent segment downloads; pick the highest rung below ~80% of it.
- **Buffer-based:** pick the rung from how full the buffer is (Netflix's BBA; BOLA in dash.js) — steadier, fewer oscillations.
- Real players blend the two, add hysteresis so they don't flap, cap resolution at the screen size and plan tier, and **prefer a lower bitrate to a rebuffer** — stalls hurt engagement far more than a softer picture.

### 5. Entitlements and availability at play time

- Check **at play time**, not when the tile was drawn: the tile may be minutes old, and the plan may have changed since.
- The rendition set is an intersection: **plan** (Basic: 1080p, no Atmos) ∩ **device** (codecs, DRM security level, HDCP) ∩ **title** (what exists).
- Availability comes from an in-memory, per-territory snapshot; windows that open at local midnight are precomputed before they open ([note 32](32_Catalog_Rights_Availability.md)).
- Country comes from IP geolocation, checked against the account's country; the profile's maturity setting filters too.

### 6. Downloads

- `POST /downloads` checks that the plan allows downloads, the title's rights allow downloading, and the account is under 10 download devices — an atomic add to a per-account device set, the same pattern as [note 16](16_Concurrent_Stream_Limits.md).
- It returns a fixed-rendition manifest (no ABR offline) and a **persistent license** with an expiry. The app renews the license whenever it's online, and each renewal re-checks the subscription and the rights.
- **Revocation is expiry:** let the subscription lapse, or lose the rights, and the next renewal is refused, so the license runs out. Signing out deletes downloads.
- Downloads use HTTP range requests, so they resume after interruptions.

### 7. Measuring quality of experience

- **Metrics:** video start time, video start failures, exits before video start, rebuffer ratio (stall time ÷ watch time), average bitrate, bitrate switches, license errors, CDN errors.
- Player beacons → collectors → Kafka → Flink (live aggregates by CDN × ISP × device × app version) → CDN steering, alerts and dashboards; raw events to the data lake (tracker #35).

## LLD

The playback service's types:

```java
public record DeviceProfile(Platform platform, Set<DrmSystem> drm, SecurityLevel securityLevel,
                            HdcpVersion hdcp, Set<Codec> codecs, Resolution maxResolution,
                            Set<HdrFormat> hdr) {}

public record PlaybackRequest(String accountId, String profileId, String deviceId, String titleId,
                              DeviceProfile device, String country,
                              Entitlement fromToken) {}   // claims from the signed access token

public sealed interface PlaybackResult permits PlaybackGranted, PlaybackDenied {}
public record PlaybackGranted(String sessionId, URI manifestUrl, URI licenseUrl,
                              long startPositionMs, List<CdnEndpoint> cdns) implements PlaybackResult {}
public record PlaybackDenied(DenyReason reason, List<StreamInfo> watching) implements PlaybackResult {}

public enum DenyReason { NOT_ENTITLED, NOT_AVAILABLE_IN_REGION, MATURITY_BLOCKED, STREAM_LIMIT_REACHED }
```

The orchestration — cheap checks first, independent lookups in parallel, and no leaked stream slot if a later step fails:

```java
@Service
public class PlaybackService {
    private final AvailabilitySnapshots availability;   // in-memory, refreshed by events
    private final EntitlementClient entitlements;
    private final StreamLeaseService leases;             // note 16
    private final BookmarkClient bookmarks;
    private final RenditionSelector renditions;
    private final CdnSelector cdnSelector;
    private final UrlSigner signer;

    public PlaybackResult start(PlaybackRequest req) {
        Availability av = availability.lookup(req.titleId(), req.country());   // no network hop
        if (!av.playableNow()) return new PlaybackDenied(NOT_AVAILABLE_IN_REGION, List.of());

        CompletableFuture<Entitlement> ent = entitlements.forAccount(req.accountId())
                .orTimeout(150, MILLISECONDS)
                .exceptionally(ex -> req.fromToken());       // slow or down: trust the signed token
        CompletableFuture<Long> resume = bookmarks.position(req.profileId(), req.titleId())
                .completeOnTimeout(0L, 100, MILLISECONDS);    // no resume point beats no playback

        Entitlement e = ent.join();
        if (!e.canWatch(av)) return new PlaybackDenied(NOT_ENTITLED, List.of());

        String sessionId = UUID.randomUUID().toString();
        LeaseResult lease = leases.acquire(req.accountId(), sessionId, e.maxStreams(), StreamInfo.of(req));
        if (lease instanceof Denied d) return new PlaybackDenied(STREAM_LIMIT_REACHED, d.watching());
        try {
            RenditionSet set = renditions.select(req.titleId(), req.device(), e.qualityCap());
            List<CdnEndpoint> cdns = cdnSelector.rank(req.country(), req.device().platform());
            URI manifest = signer.sign(cdns.get(0), set.pathPrefix(), Duration.ofMinutes(5));
            return new PlaybackGranted(sessionId, manifest, licenseUrl(req.device()), resume.join(), cdns);
        } catch (RuntimeException ex) {
            leases.release(req.accountId(), sessionId);       // don't leak the slot we just took
            throw ex;
        }
    }
}
```

Picking renditions — plan ∩ device ∩ title:

```java
public RenditionSet select(String titleId, DeviceProfile d, QualityCap cap) {
    Resolution max = Resolution.min(d.maxResolution(), cap.maxResolution());     // Basic caps at 1080p
    boolean hardwareDrm = d.securityLevel() == SecurityLevel.HARDWARE && d.hdcp().atLeast(HdcpVersion.V2_2);
    if (!hardwareDrm) max = Resolution.min(max, cap.softwareDrmCeiling());      // policy: SD or HD
    Codec codec = Stream.of(Codec.AV1, Codec.HEVC, Codec.AVC)                    // best codec it has
            .filter(d.codecs()::contains).findFirst().orElse(Codec.AVC);
    boolean hdr = cap.hdrAllowed() && max.isUhd() && !d.hdr().isEmpty();
    return catalog.renditionSet(titleId, codec, max, hdr);
}
```

Signing a manifest URL so the CDN can check it without calling back:

```java
public URI sign(CdnEndpoint cdn, String pathPrefix, Duration ttl) {
    long expires = Instant.now().plus(ttl).getEpochSecond();
    String sig = hmacSha256Hex(cdn.signingKey(), pathPrefix + ":" + expires);   // edge recomputes it
    return URI.create(cdn.baseUrl() + pathPrefix + "master.m3u8?exp=" + expires + "&sig=" + sig);
}
```

Signing the path **prefix** lets one token cover every segment under it; the CDN must exclude `exp` and `sig` from its cache key.

ABR on the player — throughput with headroom, adjusted by how full the buffer is:

```java
int chooseBitrateKbps(int[] ladderKbps, double throughputKbps, double bufferSec) {   // ascending ladder
    double budget = throughputKbps * 0.8;      // headroom for variance
    if (bufferSec < 5)  budget *= 0.5;         // close to stalling: step down hard
    if (bufferSec > 30) budget *= 1.2;         // plenty buffered: allow a step up
    int pick = ladderKbps[0];
    for (int b : ladderKbps) if (b <= budget) pick = b;
    return pick;
}
```

A session's life: `STARTING → PLAYING ⇄ PAUSED → ENDED`, or `EXPIRED` when heartbeats stop and the lease lapses ([note 16](16_Concurrent_Stream_Limits.md)). The player's own state machine is tracker #46.

## Failure modes

| Failure | What viewers see | What the design does |
|---|---|---|
| One CDN degrades in a region | Rebuffering | Steering moves new sessions away; players fail over to the next CDN mid-stream |
| License service slow | Titles don't start | Regional, pre-scaled, keys cached — it's P0 |
| Entitlement service down | Plans can't be checked | Trust the signed token's claims until they expire — fail open for paying users |
| Stream-limit store down | Streams can't be counted | Fail open ([note 16](16_Concurrent_Stream_Limits.md)) |
| Bookmark store slow | No resume point | Start from 0 or the device's local position; never block playback |
| A release hammers the origin | Slow first segments | Pre-position; shields collapse misses |
| A region goes down | API errors | Multi-region, active-active API; CDNs are global anyway |

## Trade-offs to say out loud

- **HLS vs DASH:** Apple devices need HLS; much else prefers DASH. CMAF lets both manifests point at one set of segments.
- **Segment length:** short (2 s) = faster starts and switches, more requests, slightly worse compression; long (6 s — Apple's recommended target for HLS) = efficient but slower to adapt. VOD usually sits at 4–6 s.
- **Per-title encoding:** more compute, less egress — worth it at this scale.
- **Pre-packaged vs just-in-time packaging:** storing every format costs storage; packaging at the origin on request costs compute. The CDN caches the result either way.
- **Keys per tier:** more license complexity, much better protection for 4K.
- **Multi-CDN vs your own CDN:** Netflix's Open Connect (caches inside ISPs) pays off only at enormous, steady scale.

## Trick questions

- **"Why not serve one MP4 file per title?"** — No adaptation: bandwidth drops cause stalls and bandwidth rises are wasted, and switching audio or subtitles is clumsy. Segments let the player change quality every few seconds, and each one is a small cacheable object.
- **"Why HTTP — why not UDP, WebRTC or WebSockets?"** — VOD buffers seconds ahead, so it doesn't need sub-second transport. HTTP segments are what CDNs cache, which is the whole cost model; WebRTC is for interactive sub-second video (calls) and bypasses those caches. (HTTP/3 does run over UDP via QUIC — a nice nuance.)
- **"The premiere is already on the CDN. What stops people watching early?"** — The segments are encrypted. Keys come only from the license server, which checks the availability time, and manifest URLs are only issued after the availability check. A leaked URL is ciphertext.
- **"Someone shares their signed manifest URL."** — It expires in minutes, and even a valid fetch is useless without a license, which is tied to an entitled account holding a stream slot.
- **"Why can't a hacked phone play 4K if the CDN serves the same files to everyone?"** — 4K segments are encrypted with a UHD-tier key that the license server releases only to hardware-backed DRM with HDCP 2.2. The protection lives in key policy, not in URL secrecy.
- **"The cache-hit ratio drops from 98% to 90%. So what?"** — Origin traffic goes from 2% to 10% of egress: 5× the origin load (1 → 5 Tbps at 50 Tbps). Usual causes: auth tokens in the cache key, too many rendition variants, new long-tail content, short TTLs.
- **"Cut 500 ms from startup."** — Prefetch manifest and license on the details page; warm connections at launch; start at a lower bitrate; parallelize the playback API's fan-out; read entitlements from token claims.
- **"A viewer upgrades to Premium mid-movie. Do they get 4K now?"** — Not in this session: entitlements are evaluated when the session starts, and the next one picks it up. A downgrade doesn't cut the current movie off either.
- **"Why separate audio from video segments?"** — Muxed together, every audio language × every video bitrate is a separate file: storage explodes and the cache fragments.
- **"Can you cache the manifest?"** — A VOD manifest is static, so yes, at the edge. Ad-inserted manifests for ad tiers are per session and can't be — keep them small and generate them fast (tracker #37).
- **"How many renditions?"** — Roughly 6–12 per codec. More rungs mean smoother adaptation but more encoding, storage and cache fragmentation.

## Similar systems

| System | What changes |
|---|---|
| Netflix | Its own CDN (Open Connect appliances inside ISPs) and per-shot encoding; the same control-plane / data-plane split |
| YouTube | Uploads are the hard part: ingest and transcode at huge volume, with a quick low-res version first; a long-tail catalog means lower cache-hit ratios |
| Twitch and live sports | Segments are made in real time, low-latency HLS/DASH, and every viewer wants the newest segment at once ([note 19](19_Live_Sports_Streaming.md), [note 18](18_Premiere_Traffic_Spike.md)) |
| Spotify | Audio: tiny files and no complex ladder, but the same offline-license model for downloads |
| JioHotstar (Disney+ Hotstar before the JioStar merger) | The same architecture pushed to extreme live concurrency — 59M concurrent viewers for the 2023 Cricket World Cup final |
| Zoom and video calls | Real-time: WebRTC over UDP, media servers instead of CDNs, sub-second latency — a different design entirely |

## Commonly asked

- Walk through everything between pressing Play and the first frame. Where does the time go?
- Why is video split into segments, and how does the player choose a bitrate?
- How does DRM work end to end? Why can't someone just download the segments?
- Why multi-CDN, and how do you choose a CDN for a session?
- How do you stop a title being watched before its release time if it's already on the CDN?
- Estimate peak egress. Why is the cache-hit ratio the number to watch?
- How do downloads work, and how are they revoked?
- What changes for a 4K TV, a cheap phone, a Basic-plan viewer?

## Sources

- [Disney+ plans: Basic vs Premium (Digital Trends)](https://www.digitaltrends.com/home-theater/what-is-disney-plus-plans-pricing-more/) · [Downloads on Disney+ (Disney+ Help Center)](https://help.disneyplus.com/article/disneyplus-downloads)
- [195.7M Disney+ and Hulu subscriptions in Disney's final subscriber report (AV Club)](https://www.avclub.com/disney-gained-streaming-subscribers-kimmel-q4-2025)
- [59M peak concurrent viewers for the 2023 World Cup final (Business Standard)](https://www.business-standard.com/cricket/world-cup/world-cup-ind-aus-match-records-peak-viewership-of-59-mn-on-disney-hotstar-123111900827_1.html)
