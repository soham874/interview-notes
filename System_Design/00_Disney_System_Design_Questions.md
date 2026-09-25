# Disney — System Design Interview Questions

Disney isn't one engineering org, so there's no single question list — the prompt depends on where the role sits: streaming (Disney+, Hulu, ESPN), the media supply chain behind it, ad tech, or Disney Experiences (parks, resorts, cruise, shopDisney). What candidates actually report is almost always a **classic design problem with a Disney skin** — "Continue Watching for Disney+", not "design a key-value store". So: work out the org from the job description, go deep on its questions below, and learn which classic each Disney prompt reduces to, so an unfamiliar one still feels familiar.

## Step 1 — which Disney is this?

| Org | What it builds | JD keywords | Prep first |
|---|---|---|---|
| Disney Entertainment & ESPN Product & Technology — streaming | Disney+ and Hulu (merging into one Disney+ app by the end of 2026): playback, subscriptions, identity, personalization | playback, video, subscribers, personalization | Deep dives 1, 2, 7 → Streaming & identity |
| Same org — ESPN | The ESPN app (a full streaming service since Aug 2025): live games, scores, stats, fantasy | live, sports, real-time, fantasy | Deep dives 6, 2, 3 → Sports |
| Same org — Content Platforms & Operations | The "digital media supply chain": ingest, storage, transcoding, localization and distribution of content to Disney+, Hulu, ABC and partners | media supply chain, content delivery, localization, workflow | Media supply chain |
| Disney Advertising | Ad tiers on Disney+/Hulu/ESPN, the DRAX programmatic exchange, audience segments | ads, programmatic, SSAI, measurement | Ads, then deep dive 2 |
| Disney Experiences | My Disney Experience / Disneyland apps: ticketing and gate entry, Lightning Lane, virtual queues, dining/hotel/cruise booking, shopDisney | guest, parks, resorts, ticketing, reservations, commerce | Deep dives 4, 5 → Disney Experiences |

- **Bengaluru** has been posting both media-supply-chain and Disney Experiences roles through 2026, and its backend postings ask for Java, Spring Boot, microservices, REST/GraphQL, AWS (SQS, S3, Lambda, DynamoDB), Elasticsearch and MongoDB — the `Java_SpringBoot/` track maps straight onto them.
- **Hotstar reports aren't Disney reports anymore.** Star India (Hotstar included) merged with Viacom18 into JioStar in Nov 2024 — Reliance controls it, Disney keeps ~37%. Use those write-ups as a signal of India-loop *style* (LLD + HLD + schema in one round), not as Disney's current question bank.

## Step 2 — what candidates have reported

| Prompt | Where | How solid |
|---|---|---|
| "Design a 'Continue Watching' and Recommendations service for Disney+ that works across multiple devices, supports millions of users, and remains responsive during traffic spikes like major show releases." | Disney Streaming, SWE II — 75 min with two senior engineers, one a principal | First-hand review |
| Global game leaderboard — many games, score updates, real-time Top 100, "my rank", friend boards, season-end settlement | Product Software Engineer 2 | First-hand (LeetCode Discuss) |
| BookMyShow — "LLD, HLD, Database design were expected. Focus was on handling the concurrency." | Disney+ Hotstar SDE-2 (pre-JioStar) | First-hand (Glassdoor) |
| Instagram-style photo sharing | Disney+ Hotstar SDE-2 | Candidate write-up |
| Data structures for a calculator's nested operations | Disney | Glassdoor |
| URL shortener; badge ("FOB") door-access system; LRU cache | Hulu | Glassdoor, via an aggregator |
| "Design an ESPN-like system" — player scores, number of matches, scaling and availability | ESPN | CareerCup |
| Theme-park ticketing (holiday peaks, ticket types and promos, gate access control, real-time capacity); a launch-day streaming surge; the playback path / a manifest service; e-commerce browsing and search; a "ride-state aggregator that survives a partition" | Various | Prep-site guides — plausible, not first-hand |

What the reports agree on:

- Interviewers push on non-functional requirements — p50/p95 video start time, rebuffer rate, concurrency at spikes — and on *why* you picked a store (NoSQL vs SQL comes up by name).
- They reward clarifying questions and guest-centred thinking: the leaderboard interviewer explicitly valued creativity around the player's "victory" moment. Treat "what does the guest see when this fails?" as a requirement.
- Testing comes up even in design discussions — say how you'd test and load-test what you drew.

## Step 3 — every Disney prompt is a classic in disguise

| Disney prompt | Classic it reduces to | The one hard part |
|---|---|---|
| Continue Watching / resume on another device | Write-heavy key-value store fed by a stream | Billions of heartbeat writes a day; the newest position must win even when updates arrive out of order |
| Lightning Lane, dining, BookMyShow | Ticketmaster (inventory booking) | The last unit under a 7 AM stampede — no oversell, no double booking |
| Virtual queue, merch-drop waitlist | Flash sale + waiting room | Fair, exactly-once allocation when all demand lands in one second |
| Live scores, "new episode" alerts | Pub/sub fan-out + push notifications | One event to tens of millions of devices in seconds |
| Game leaderboard | Sorted sets / top-K | Rank queries at scale, ties, season resets |
| 2–4 streams per account, by plan | Distributed semaphore with leases | Devices that die without releasing their slot |
| Premiere or live-match spike | Pre-scaling + load shedding | Traffic arrives faster than autoscaling reacts |
| Disney+ playback | Video streaming (Netflix/YouTube) | Time to first frame: auth + entitlement + manifest + DRM license + first segment |
| MyDisney login everywhere | SSO / OAuth2 identity provider | Every app depends on it — it must not take playback down with it |
| Wait times, ride status | Edge aggregation + telemetry | Park-to-cloud network partitions |
| Media ingest & transcoding | Distributed workflow / batch pipeline | Hours-long jobs that must resume, not restart, after a failure |

## Deep dives — the most likely prompts

### 1. Continue Watching (+ recommendations) for Disney+ — reported

Per AWS and Disney's own re:Invent 2020 talk, Disney+ writes bookmarks to DynamoDB global tables, ingesting billions a day through Kinesis — so "stream into a NoSQL store" isn't just the textbook answer, it's what they built.

- **Clarify:** per *profile*, not per account (up to 7 profiles); how fresh across devices (next app open, or within seconds?); when a title leaves the row (credits reached → switch to the next episode); multi-region; retention.
- **Estimate:** say 10M concurrent streams × one heartbeat per 10 s = **1M writes/s** at peak. Reads: every home-page load, plus every play (the resume point).
- **Write path:** player sends `{profileId, titleId, episodeId, positionMs, durationMs, deviceId, sessionId, ts}` every N seconds and on pause/seek/stop/background → thin ingest API → stream (Kafka/Kinesis) partitioned by `profileId` → consumer coalesces (latest per profile+title per few seconds) → upsert with a **conditional write** so an older update can't overwrite a newer one (`attribute_not_exists(updatedAt) OR updatedAt < :ts`).
- **Data model:** PK `profileId`, SK `titleId` (a series stores its current episode + position). A profile's partition is small (tens to hundreds of items), so read it whole and sort by `updatedAt` in the service, or add an index on `(profileId, updatedAt)`.
- **Read path:** bookmarks for the profile → drop finished titles → hydrate with catalog metadata (heavily cached) → filter to what's still available in the viewer's country and allowed for the profile's maturity rating → cache per profile briefly, invalidated on new writes.
- **Multi-region:** active-active with last-writer-wins is fine *here* — the only "conflict" is one profile on two devices, and the latest position winning is exactly the product behavior you want. Mention clock skew (use server receive time or a per-session sequence number).
- **Premiere spike:** the stream absorbs write bursts; lengthen the heartbeat interval via remote config; and if this row's service is struggling, the home page renders *without the row* rather than failing.
- **Recommendations:** batch-computed candidate rows per profile in a KV store, re-ranked online with fresh signals — Continue Watching is the strongest intent signal you have.
- **Follow-ups to expect:** why NoSQL over SQL (single-key access, huge write volume, no joins — you'd have to shard Postgres by profile anyway; say when SQL *would* win); what if heartbeats are lost (the next one fixes it; always send on pause/stop); how the home page stays fast during a launch.

### 2. Disney+ playback path + surviving launch day

Full notes: [17 · Disney+ video streaming](17_Disney_Plus_Video_Streaming.md) and [18 · traffic spikes](18_Premiere_Traffic_Spike.md).

- **Content prep (offline):** mezzanine → transcode to an ABR ladder (resolutions × bitrates × codecs, HDR variants) → package as CMAF with HLS and DASH manifests → encrypt with common encryption so one set of segments can serve every platform, with Widevine, PlayReady and FairPlay license servers handing out keys → origin storage → multiple CDNs.
- **Play (online):** client → playback API: validate token → entitlement (plan, country availability, a free concurrent-stream slot, profile maturity) → pick renditions the device supports and a CDN (steered by live QoE data) → return a signed manifest URL + license URL → client fetches manifest, segments and license from there.
- **The crux — time to first frame.** Every hop is on the critical path: fetch license and manifest in parallel, prefetch on the details page, keep manifests edge-cacheable, keep entitlement to token claims + one cache hit. Name the metrics: p50/p95 video start time, rebuffer ratio, start failures.
- **Spike playbook** (a premiere at a fixed time, or a big live game):
  - **Pre-scale on a forecast** — reactive CPU autoscaling lags traffic that arrives in seconds. Hotstar (Disney-owned at the time) described this for 25M concurrent viewers at re:Invent 2019: capacity added in ladders keyed to expected concurrency, plus a buffer.
  - **Protect the P0 path** — login → entitlement → manifest/license → CDN. A "panic mode" of feature flags sheds everything else (recommendations, personalization, social) and serves a cached, non-personalized home page.
  - **Stop retry storms** — jittered exponential backoff in clients, `Retry-After` from servers, circuit breakers between services.
  - **Avoid a login storm** — long-lived refresh tokens mean already-signed-in viewers never touch the identity service at kickoff.
  - **Rehearse** — load-test above forecast; spread traffic across more than one CDN.
- **Follow-ups:** what you turn off first and why; how ABR picks a bitrate; what happens when the license service is slow (it's P0 — replicate it; short-lived licenses can be cached).

### 3. Global game leaderboard — reported

- **Clarify** (the interviewer wanted these asked): real-time or near-real-time; global vs regional boards; seasons; mostly Top-100 reads or "my rank"; best score or cumulative.
- **Write path:** match result → ingest API → validate (dedupe on `matchId`, anti-cheat sanity checks) → durable results store (the source of truth) → event → ranking service → Redis sorted set per `(game, season, region)`: `ZADD lb:{game}:{season}:{region} GT <score> <playerId>` for best score (`GT` only ever raises it), `ZINCRBY` for cumulative.
- **Reads:** Top 100 → `ZREVRANGE key 0 99 WITHSCORES`, cached ~1 s since everyone asks the same thing; my rank → `ZREVRANK`; "around me" → the range rank ± 5; friends → `ZMSCORE` their ids and sort in the service (friend lists are small).
- **Ties:** pack the tiebreak into the score, e.g. `score × 2^24 + (2^24 − 1 − secondsSinceSeasonStart)`, so whoever got there first ranks higher — and check it stays under 2^53, since sorted-set scores are doubles.
- **Scale:** one sorted set on one node handles millions to tens of millions of players (O(log N) updates). Beyond that, shard by player and compute global rank as 1 + the sum of `ZCOUNT key (score +inf` across shards, merging each shard's top 100 for the Top 100. Or make the product call: exact rank for the top 10K, a percentile ("top 3%") from a score histogram for everyone else.
- **Seasons:** a new key per season; at season end switch the active season id, snapshot the old set to durable storage, and settle rewards as an idempotent batch (grant id = season + player, so a rerun can't double-grant).
- **Durability:** Redis is a derived view — rebuild it by replaying the results store.
- **The "victory" moment** (what the interviewer said they valued): push the rank change right after the match ("#57 → #42 — you just passed a friend"), friend-overtake notifications, and regional or tiered boards so most players have a board where they're near the top.

### 4. Lightning Lane (or dining) booking — Ticketmaster, every morning

Lightning Lane Multi Pass / Single Pass selections open at exactly **7 AM ET**, in the My Disney Experience app only — up to 7 days ahead (for the whole stay, up to 14 days) for Disney resort guests, 3 days ahead for everyone else. A synchronized stampede every day, and the resort-guest head start is a business rule your design has to enforce.

- **Clarify:** party booking (the whole party needs the same return window — all or nothing); per-guest limits and no overlapping windows; purchase (priced by date) vs selection; modify/cancel.
- **Inventory:** one row per `(attractionId, date, returnWindow)` with `capacity` and `booked`. Book a party of N in one conditional write — `UPDATE slot SET booked = booked + :n WHERE id = :id AND booked + :n <= capacity`, then check rows affected (or a DynamoDB conditional update / Redis Lua script). No read-then-write gap, no long-held locks.
- **Per-guest rules:** entitlement bought, ticket and park valid, selection limit, no overlap — keep a per-guest booking record updated with a conditional write/version too, plus an **idempotency key** per request so a double-tap or a client retry books once.
- **Several selections at once:** a saga — book each slot, and if a later one fails, release the earlier ones (compensation); or place short TTL holds on all of them and confirm together.
- **The 7 AM surge:** pre-scale; serve availability from cache (seconds-stale is fine — the conditional write is the authority); a virtual waiting room in front if needed; for the hottest attractions, split a slot's capacity across K sub-rows (sharded counters) and try a random one first, so writes don't serialize on one row.
- **Ties to your notes:** this is the optimistic-vs-pessimistic locking question from [`09_Database_Transactions.md`](../Java_SpringBoot/09_Database_Transactions.md). The guarded `UPDATE` is a third option worth naming: a compare-and-set on the invariant itself — no version column to retry on, and the row lock lasts one statement instead of spanning application code.
- **Follow-ups:** the ride breaks down at 2 PM — re-accommodate 3,000 bookings (bulk-convert to a flexible credit + fan-out notification); stopping bots (per-account/device rate limits, device attestation, bookings tied to valid tickets); showing availability without hammering the database.

### 5. Virtual queue (boarding groups, merch drops)

When Disney World runs a virtual queue for a ride, joins open at **7 AM** (from anywhere) and again at **1 PM** (only from inside the park); guests confirm their party beforehand, get a boarding group, and have an hour to return once it's called. Disney has reused the pattern for a merchandise shop's entry waitlist (Disney Drop Shop) and a seasonal character meet with several drops a day that you can only join on-site.

- **The crux:** all demand lands in the first second, capacity is gone in seconds, each party gets at most one spot — and it has to feel fair.
- **Before 7:00:** the app pre-confirms party and eligibility (tickets, park reservation), and the server hands back a signed, short-lived eligibility token — so the 7:00:00 request needs no calls to ticketing.
- **At 7:00:** join = `(queueId, eligibility token, idempotency key)` → dedupe every party member (`SET vq:{queue}:guest:{id} 1 NX`, atomically for the whole party via a Lua script) → position = `INCR vq:{queue}:seq` → boarding group = ⌈position ÷ group size⌉ → stop at capacity (then backup groups, then "full"). The idempotency key means a retry after a timeout returns the same group, not a second one.
- **Hotspot:** one Redis `INCR` key does on the order of 100K ops/s, which may be enough; if not, give each allocator shard an interleaved range (shard *i* hands out *i*, *i+N*, *i+2N*…) — you lose strict first-come order, but a few milliseconds is network noise anyway.
- **Name the fairness trade-off:** instead of strict first-come, accept every request in the first ~2 s into a log and assign positions by lottery — fairer to guests on slow networks and less rewarding for bots. It's a product decision, so propose it; don't impose it.
- **"Must be in the park" at 1 PM:** GPS is spoofable; a park-entry tap earlier that day is a much stronger signal — combine them.
- **Calling groups:** ops sets "called through group X" from actual ride throughput → push to those parties → the ride entrance scans band/phone and checks the group is called and inside its hour. If the ride goes down, pause calling and extend windows.

### 6. Live scores & alerts (ESPN)

"Design an ESPN-like system" is a reported prompt, and since Aug 2025 the ESPN app is also a full streaming service with live stats, multiview, fantasy and betting information.

- **Pipeline:** stats-provider feeds → normalize → Kafka keyed by `gameId` (ordering per game) → game-state service (score, clock, play-by-play, a per-game sequence number) → two delivery paths.
- **Pull (most clients):** game state as small JSON behind the CDN with a 1–2 s TTL — tens of millions of polls collapse to a handful of origin requests per game. Often cheaper and sturdier than millions of open sockets; say so.
- **Push:** WebSocket/SSE gateways subscribed to per-game topics (Redis pub/sub, NATS or Kafka) for in-app live updates; APNs/FCM for alerts to followers of a team or player.
- **Alert fan-out:** follower lists by team/player in sharded segments; fan-out workers each take a shard and batch to APNs/FCM within provider rate limits; put the score in the payload so 20M phones opening the alert don't all hit your API.
- **Correctness:** per-game sequence numbers so clients drop stale or duplicate updates; scoring changes arrive as correction events, not silent edits.
- **Follow-ups:** reaching 20M devices in under a minute; the thundering herd right after a push; SSE vs WebSocket vs polling.

### 7. Stream limits per account (concurrency & sharing)

Full note: [16 · stream limits](16_Concurrent_Stream_Limits.md).

Disney+ (US) allows 2 simultaneous streams on Basic/Standard and 4 on Premium, with up to 7 profiles; paid sharing adds one "Extra Member" (one stream at a time) and prompts TVs outside the household ("This TV doesn't seem to be part of the Household for this account").

- **Design:** a lease-based distributed semaphore. On play, atomically add `{sessionId → leaseExpiry}` to the account's active-stream set only if fewer than the plan's limit are live (Redis Lua script, or a DynamoDB conditional update on a version). The playback heartbeats from #1 renew the lease; a crashed device frees its slot when the lease expires; stop releases it explicitly.
- **Enforce it server-side:** tie DRM license renewal to holding a lease — a client can ignore an API "no", but it can't keep decrypting without a license.
- **The real question — failure mode:** if the lease store is down, **fail open** (let paying viewers watch, reconcile later) rather than closed; a few minutes of over-admission costs less than a premiere outage. Say why.
- **Follow-ups:** "sign out of all devices" (revoke refresh tokens + drop leases); multi-region (home each account's lease set in one region, or accept brief over-admission).

## More prompts, by org

### Streaming & identity

- **Subscriptions, bundles & entitlements** — plans and bundles across Disney+, Hulu and ESPN, bought on the web or through Apple/Google/Roku billing. Split **billing** (money, renewals, retries) from **entitlements** (what this account can do right now). Provider webhooks → idempotent processing (dedupe on event id) → subscription state machine (trial → active → grace → cancelled/expired) → recompute an entitlement set (`hulu_with_ads`, `espn_unlimited`, `max_streams=4`) → push invalidation so an upgrade applies immediately. Follow-ups: never double-charge (idempotency key per account per billing period); a webhook arriving before the purchase redirect; grandfathered prices (versioned plans). Full note: [30 · subscriptions & billing](30_Subscriptions_Billing.md).
- **Catalog, rights & availability** — what's playable depends on country, date window, plan and device, and merging Hulu into Disney+ combines two catalogs. A read-heavy, edge-cached metadata service + availability rules `(titleId, territory, start, end, tiers, platforms)`; precompute per-territory snapshots *before* a window opens (local midnight in each territory) rather than at T-0; the authoritative check runs at play time, not when a tile is drawn.
- **MyDisney single sign-on** — one login across Disney+, Hulu, ESPN, ABC, the parks, Disney Cruise Line and the Disney Store (since April 2024). OIDC/OAuth2 identity provider; short-lived JWT access tokens that services validate *locally* with cached signing keys (so an identity-service blip doesn't stop playback) + refresh tokens; TVs sign in with the device-code flow (RFC 8628 — code on the TV, approve on the phone); "sign out everywhere" = revoke refresh tokens; credential-stuffing defenses (per-IP/account rate limits, breached-password checks); merging legacy duplicate accounts; stricter rules for kids' data.
- **Watchlist** — per-profile CRUD at huge scale (PK `profileId`, SK `titleId`), idempotent add/remove, read-your-writes across devices. A common warm-up or add-on to #1.
- **Offline downloads** — persistent DRM licenses that expire and must be renewed online, per-account download limits, revocation when the subscription lapses; download rights can differ from streaming rights.
- **Watch party** — Disney+ ran GroupWatch until Sept 2023. Only control messages (play/pause/seek + timestamps) go over WebSockets; every viewer still streams individually from the CDN and must be entitled; clients correct drift by nudging playback rate, with clock offset estimated from ping round-trips.
- **Playback QoE telemetry** — millions of clients emit start time, rebuffers, bitrate switches, errors → edge collectors → Kafka/Kinesis → Flink (real-time alerts, CDN steering) + an OLAP store (Druid/Pinot/ClickHouse) + a data lake. Event-time windows with watermarks for late events; sample chatty events; watch dimension cardinality (device × app version × CDN × ISP × region).

### Ads (Disney+ with ads, Hulu, ESPN)

- **Server-side ad insertion + decisioning** — Disney runs its own programmatic exchange (DRAX, since 2021) and first-party audience segments. At each break (cue points for VOD, SCTE-35 markers for live) the manifest service asks the ad decision server for a pod and stitches pre-transcoded ad segments into the manifest. Crux: decisioning must fit inside the manifest's latency budget — prefetch decisions for upcoming breaks; frequency caps as per-household counters with TTLs in a fast KV store (approximate is fine); for live sports, millions hit the same break at once, so pre-decide and jitter.
- **Impression counting** — beacons → stream → dedupe on event id → aggregate. Billing depends on these numbers, so idempotency, late events and fraud filtering are the conversation.

### Sports

- **Fantasy — draft room + Sunday scoring** — shard by `leagueId`. The live draft is a stateful room (WebSocket) with a server-authoritative pick clock and auto-pick on timeout; the pick number doubles as an optimistic-concurrency version so two picks can't both land. Sunday scoring computes each real player's fantasy points once, then fans out via an inverted index (player → fantasy teams rostering them); stat corrections days later recompute idempotently from the event log.

### Media supply chain

- **Ingest & transcoding pipeline** — studios deliver huge mezzanine files → multipart upload straight to object storage via presigned URLs (bytes never touch app servers) → checksum + automated QC → a workflow engine (Step Functions/Temporal-style, every step idempotent and retryable) → split into chunks → parallel transcode on a queue-fed worker fleet (spot instances) → stitch → package + DRM → publish → "title ready" event to the catalog. Crux: jobs run for hours, so checkpoint per chunk and resume instead of restarting; priority queues let a premiere jump a back-catalog re-encode; keep versions and provenance, because masters get re-delivered.
- **Localization & delivery workflow** — title → versions → components (video, audio per language, subtitles, artwork, metadata); each destination (a Disney+ region, Hulu, a partner) has a package spec; a dependency graph gates publishing (no French launch until French subtitles pass QC); each delivery is a state machine with an audit trail and SLA alerts.
- **Media asset management & search** — relational metadata for relationships and rights, object storage for bytes, a search index (Elasticsearch/OpenSearch) for discovery; unreleased content needs signed URLs, watermarking and access audit logs.

### Disney Experiences

- **Park ticketing & gate entry** — date-based pricing, multi-day tickets, add-ons, passes with blockout dates, per-day park capacity (atomic counters, as in #4). Crux: the gate must admit guests when the cloud link is slow or down — push today's signed entitlements to gate controllers, decide locally, sync taps later, push revocations for refunds, and be explicit about what risk you accept offline.
- **Ride status & wait times** (the "ride-state aggregator that survives a partition") — an aggregator in each park collects ride status and queue counts, computes waits locally (Little's law: wait ≈ people in line ÷ riders per minute, blended with history and smoothed so the number doesn't flap), drives in-park signage locally, and forwards to the cloud store-and-forward. Each ride has exactly one writer (its park), so there's nothing to merge; per-ride sequence numbers drop stale updates; while partitioned, apps show "updated 4 min ago" rather than a confident wrong number.
- **Hotel / cruise booking** — date-range inventory (room type × night): a 5-night stay decrements 5 night-rows in one transaction, locking in date order to avoid deadlocks (the transactions notes again); a TTL hold during checkout, then a saga — hold → charge → confirm, compensate on failure.
- **Dining reservations** — the #4 flash pattern, but inventory is capacity per time slot *per table size* (a party of 6 needs a 6-top).
- **Mobile food ordering** — pickup "arrival windows" capped by kitchen capacity per window (slot booking again); order state machine (placed → paid → "I'm here" → preparing → ready); kitchen-display integration; ready notification.
- **Tap to do everything (band or phone as ticket, room key, payment)** — readers send an opaque ID that the backend maps to a guest; room doors keep working offline with credentials provisioned ahead of time; payments need online authorization with limits; a lost band is revoked and replaced fast.
- **Ride & character photos** — the photographer scans the guest at capture (photoId ↔ guestId); ride photos match on `(ride, vehicle, time window)` using the ride system's record of who sat where; processing pipeline → thumbnails → retention window and purchase entitlement.
- **shopDisney** — catalog + faceted category browsing + search (Elasticsearch), CDN-cached category pages, approximate stock on browse, reservation at checkout. Limited drops = waiting room + TTL inventory holds + per-customer limits + bot defenses, with expired holds released back to stock.

### Classics that Disney-adjacent loops have used

BookMyShow (seat-lock concurrency), URL shortener, LRU cache, rate limiter, notification system, chat, Instagram-style upload and feed, badge-based door access. Know the standard answer, then attach a Disney use: rate limiter → protecting the 7 AM booking APIs; notifications → "your boarding group has been called"; chat → the fantasy draft room.

## If the "design" round is LLD

India loops in particular often make the design round object-oriented — Hotstar's SDE-2 BookMyShow round wanted LLD, HLD and the schema. Classics: parking lot, BookMyShow seat locking, elevator, vending machine, LRU cache, rate limiter. Disney-flavored practice prompts and the patterns they exercise (see [`13_Design_Patterns_Java.md`](../Java_SpringBoot/13_Design_Patterns_Java.md)):

| Prompt | Core classes | Patterns |
|---|---|---|
| Calculator with nested operations (reported) | `Expression` → `Number`, `BinaryOp` (`Add`, `Sub`, `Mul`, `Div`); `Parser` | Composite / Interpreter |
| Video player | `Player`, `PlayerState` (Idle, Loading, Playing, Paused, Buffering, Ended, Error) | State |
| Ride queue — standby + Lightning Lane merge | `Attraction`, `Queue`, `Party`, `MergePolicy` (configurable ratio) | Strategy; Observer to notify parties |
| Subscription plans & bundles | `Plan`, `AddOn`, `Bundle`, `PriceCalculator`, `Discount` | Strategy / Decorator; versioned plans |
| Leaderboard | top-K, rank, around-me, tie-break `Comparator` | `TreeMap` / sorted structure |
| Badge door access (reported at Hulu) | `Employee`, `Badge`, `Door`, `AccessPolicy` (role, schedule), `AuditLog` | Strategy / Chain of Responsibility |
| Notification preferences | `Channel` (push/email/SMS), `Preference`, `Dispatcher` | Observer, Strategy |

## Running the round

- **Requirements first, with numbers.** Functional, then non-functional: peak concurrency, a latency target ("p95 under 200 ms for the booking API"), availability, and consistency *per operation* (booking must be strongly consistent; the availability display can be seconds stale).
- **Ask the guest-experience question** — "what should the guest see when this dependency is down?" Degrade (a stale wait time, a home page missing one row, "we'll notify you") instead of erroring. This is the product thinking the reports mention.
- **Estimate, then choose storage and defend it** from access patterns and write volume, not buzzwords — the NoSQL-vs-SQL question is coming.
- **Go deep on the crux** (Step 3's last column), then failure modes, then the peak moment — the premiere, 7 AM, kickoff.
- **Close with operability:** the SLIs you'd alert on, load testing above forecast, rollout behind feature flags and canaries, and how you'd test it.
- Reported formats: about an hour per round (one report: 75 min with two engineers); 3–5 rounds overall, with senior loops sometimes adding a second design or leadership round.

## Numbers worth having ready

- Disney+ bookmarks: billions a day, Kinesis → DynamoDB global tables (re:Invent 2020).
- Disney+ (US): 2 concurrent streams on Basic/Standard, 4 on Premium; up to 7 profiles; an Extra Member gets 1 stream. Downloads are Premium-only: up to 10 devices, reconnecting at least every 30 days.
- Lightning Lane: opens 7 AM ET — 7 days ahead for resort guests (whole stay, up to 14 days), 3 days for everyone else.
- Virtual queue: 7 AM (anywhere) and 1 PM (in park); 1 hour to return once called.
- Hotstar: 25M concurrent viewers in 2019 — the standard live-spike case study.
- Back-of-envelope: a day ≈ 10^5 s, so 1M requests/day ≈ 12/s; 10M streams × 1 heartbeat per 10 s = 1M writes/s; one Redis shard ≈ 10^5 simple ops/s.

## Commonly asked

- Walk through everything between pressing Play on Disney+ and the first frame. Where does the time go, and what would you cut?
- Two devices on one profile send progress out of order — how do you guarantee the newest position wins?
- Why a key-value store for watch progress instead of Postgres? What would make you choose Postgres?
- 2M people tap "Join" at 7:00:00 — how do you hand out boarding groups fairly, exactly once per party?
- Two guests go for the last Lightning Lane slot at the same instant. Walk through the write. Optimistic or pessimistic, and why?
- How do you enforce the per-plan stream limit (2 or 4) when a device crashes without saying goodbye — and what happens if that check's datastore is down?
- A premiere will bring 10× normal traffic at a fixed time. What do you do the week before, the hour before, and while it's melting?
- What do you turn off first when overloaded, and how do you decide?
- Send a score alert to 20M followers in under a minute.
- The park's link to the cloud drops. What keeps working, and what do guests see?
- Top 100 and "my rank" for 1M players — then for 100M. What changes?
- How does a TV sign in to MyDisney without a keyboard, and how does "sign out everywhere" work?

## Sources

Reported questions:

- [Continue Watching — Disney Streaming SWE II review (LockedIn AI)](https://www.lockedinai.com/company-details/DIS/db899eb5-5097-4163-b53b-5ace6c205058)
- [Global leaderboard — Product Software Engineer 2 (LeetCode Discuss)](https://leetcode.com/discuss/post/8501988/the-walt-disney-company-product-software-ysis/)
- [BookMyShow — Disney+ Hotstar SDE-2 (Glassdoor)](https://www.glassdoor.com/Interview/Design-BookMyShow-LLD-HLD-Database-design-were-expected-Focus-was-on-handling-the-concurrency-QTN_4642647.htm) and an [SDE-2 write-up](https://devbrainiac.com/blogs/106/disney-hotstar-sde-2-interview-experience-coding-system-design-techno-managerial/)
- [Disney SWE interview questions (Glassdoor)](https://www.glassdoor.com/Interview/Walt-Disney-Company-Software-Engineer-Interview-Questions-EI_IE717.0,19_KO20,37.htm) · [Hulu SWE interview questions (Glassdoor)](https://www.glassdoor.com/Interview/Hulu-Software-Engineer-Interview-Questions-EI_IE43242.0,4_KO5,22.htm) · [ESPN-like system (CareerCup)](https://www.careercup.com/question?id=5713351995817984)
- Prep guides, not first-hand: [Interview Query — Disney Experiences SWE](https://www.interviewquery.com/interview-guides/disney-experiences-software-engineer) · [knok — Disney SWE](https://knok.work/blog/the-walt-disney-company-software-engineer-interview.html)

Product and architecture facts:

- [How Disney+ scales globally on Amazon DynamoDB (re:Invent 2020)](https://www.youtube.com/watch?v=TCnmtSY2dFM) · [Scaling hotstar.com for 25 million concurrent viewers (re:Invent 2019)](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf)
- [Lightning Lane booking windows](https://www.undercovertourist.com/blog/disney-lightning-lane-faq/) · [Virtual queues at Walt Disney World](https://wdwprepschool.com/disney-world-virtual-queue/) · [New virtual queues, Nov 2025](https://allears.net/2025/11/17/disney-world-just-quietly-added-a-new-virtual-queue/)
- [Disney+ concurrent streams](https://help.disneyplus.com/article/disneyplus-en-ca-concurrent-streams) · [Disney+ downloads](https://help.disneyplus.com/article/disneyplus-downloads) · [Paid sharing on Disney+](https://thewaltdisneycompany.com/news/paid-sharing-disney-explainer/) · [MyDisney login](https://thewaltdisneycompany.com/news/mydisney-seamless-login-faq-what-you-need-to-know/) · [GroupWatch removed](https://whatsondisneyplus.com/disney-removes-groupwatch-feature/)
- [ESPN direct-to-consumer launch](https://thewaltdisneycompany.com/news/espns-direct-to-consumer-launch-date/) · [Hulu merging into the Disney+ app](https://www.cbsnews.com/news/hulu-disney-plus-app/) · [DRAX and Disney Compass](https://www.marketingdive.com/news/disney-compass-seeks-true-north-data-collaboration-amazon/750861/)
- [JioStar merger completed](https://www.jiostar.com/news/reliance-and-disney-announce-completion-of-transaction-to-form-joint-venture-to-bring-together-the-most-iconic-and-engaging-entertainment-brands-in-india/) · [Disney technology jobs in Bengaluru](https://www.disneycareers.com/en/location/bengaluru-jobs/391/1269750-1267701-1277333/4)
