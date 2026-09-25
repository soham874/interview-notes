# Concurrent Stream Limits & Household Sharing

**The prompt:** "Disney+ lets 2 screens stream at once on Basic/Standard and 4 on Premium. Design the system that enforces it — paid sharing included." It looks small, and that's the trap. It tests distributed counting under races, what happens when devices vanish without saying goodbye, where to enforce so a modified app can't skip the check, and whether you'll trade a little correctness for availability. Related: [17 · Disney+ video streaming](17_Disney_Plus_Video_Streaming.md) (where the check runs), [30 · subscriptions & billing](30_Subscriptions_Billing.md) (where the limit comes from).

## Clarify first

**What Disney+ does today (US):** 2 simultaneous streams on Basic/Standard (with or without ads), 4 on Premium; up to 7 profiles; one paid **Extra Member** outside the household with their own login and 1 stream at a time; and TVs that seem to be outside the household see "This TV doesn't seem to be part of the Household for this account."

**Functional**

- At playback start: allow if the account has fewer active streams than its plan allows; otherwise deny and **show who's watching** (device, profile, title) with a way to stop one.
- Stopping frees the slot; a crashed or disconnected device frees it automatically within a bounded time.
- "Sign out of all devices" ends every stream.
- Extra Member: a separate login with its own 1-stream limit, billed to the main account.
- Household: detect likely out-of-household viewing and prompt or verify. How strict to be is a product decision.
- Offline playback of downloads doesn't count (it can't be counted live); downloads have their own 10-device cap.

**Non-functional**

- Adds ≤ ~10–20 ms to playback start at p99.
- **A false denial is worse than brief over-admission.** Blocking a paying family during a premiere is a support crisis; letting one extra stream slip through for a minute costs almost nothing.
- Scale: ~10M active streams at peak; starts in the tens of thousands per second (bursts ×10 at premieres).
- Playback must not stop when this system's datastore does.

## Back-of-envelope

- **Renewals:** 10M streams, one heartbeat each per 30 s → **~330K renewals/s**. This is the real load, not the starts.
- **Starts and stops:** ~5–50K/s each.
- **State:** ~100 bytes per active stream × 10M ≈ **1 GB** — a small Redis cluster, partitioned by account.

## API (internal)

```text
acquire(accountId, sessionId, limit, info) → GRANTED | DENIED(watching[])
                                             (renewing = acquiring again)
release(accountId, sessionId)
active(accountId) → watching[]
evict(accountId, sessionId)                  "stop streaming on that TV"
```

Viewers see it through the playback API in [note 17](17_Disney_Plus_Video_Streaming.md): `409 STREAM_LIMIT_REACHED` with the list of active streams, and `POST /v1/playback/sessions/{id}/stop` to stop another device.

## Data model

- **Redis, per account.** The `{…}` hash tag puts both keys in one cluster slot, so one Lua script can touch both:
  - `streams:{acct}` — a sorted set: member = sessionId, score = lease expiry in ms.
  - `streammeta:{acct}` — a hash: sessionId → `{device, profile, title, startedAt}` for the "who's watching" screen.
- **DynamoDB alternative:** one item per account — `{pk: acct, sessions: {sessionId: {exp, device, …}}, version}` — updated with a conditional write on `version` (optimistic concurrency; an account sees a few writes a minute, so conflicts are rare).

## Architecture

```text
 Player ──start──→ Playback service ──acquire──→ Stream-limit service
   │                                                    ↑       │
   ├──heartbeat, every 30 s──→ Heartbeat API ──renew────┤       ↓
   │                                                    │ Redis cluster
   └──license renewal──→ License service ──has lease?───┘ (per account)
                         no lease → no renewal → playback stops

 Signals: networks, device IDs, sign-in activity
    │
    ↓
 Household service ──→ prompts, verification codes, Extra Member offer
                       (policy decides how strict — it isn't the counter)
```

- **Start:** acquire → granted, and the session proceeds; denied → 409 with who's watching.
- **Heartbeat:** renewing is just acquiring again with the same sessionId. If the lease lapsed *and* another device took the slot meanwhile, the answer is "denied" and the player stops with a "too many devices" message.
- **Stop:** release, best effort — lease expiry is the backstop.
- **Crash:** the lease expires after ~90 s (three missed heartbeats) and the slot frees itself.
- **"Stop that TV":** evict → that device's next heartbeat or license renewal fails → it stops within one interval (a push message can stop it sooner).

## Deep dives

### 1. Leases, not counters

- `INCR` on play and `DECR` on stop leaks: a phone that dies, loses signal or has its app killed never sends the `DECR`, and the family is locked out until someone writes a cleanup job — which would itself be a lease system. So store **leases** that expire unless renewed.
- **Lease length is a trade-off:** shorter frees a crashed device's slot sooner, but costs more renewal traffic and risks dropping streams during brief network blips. 2–3× the heartbeat interval is the usual answer.
- **Idempotent by sessionId:** a retried acquire (timeouts happen) must not take a second slot. The server mints the sessionId at playback start and binds it to the device's token, so a client can't forge one or reuse someone else's.

### 2. The atomic check-and-add

Two TVs pressing Play at once with one slot left is the race the interviewer is waiting for. Redis runs a Lua script atomically, so prune + count + add happen as one step:

```lua
-- KEYS[1] = streams:{acct}      sorted set: sessionId -> lease expiry (ms)
-- KEYS[2] = streammeta:{acct}   hash: sessionId -> who/what is watching
-- ARGV[1] = sessionId, ARGV[2] = leaseMs, ARGV[3] = limit, ARGV[4] = metaJson
local t = redis.call('TIME')                                  -- one clock for every app server
local now = tonumber(t[1]) * 1000 + math.floor(tonumber(t[2]) / 1000)
local lease = tonumber(ARGV[2])

local expired = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', now)
if #expired > 0 then
  redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', now)
  redis.call('HDEL', KEYS[2], unpack(expired))
end

local held = redis.call('ZSCORE', KEYS[1], ARGV[1])           -- a renewal or a retried acquire?
if not held and redis.call('ZCARD', KEYS[1]) >= tonumber(ARGV[3]) then
  return {0, redis.call('HGETALL', KEYS[2])}                  -- denied: say who's watching
end
redis.call('ZADD', KEYS[1], now + lease, ARGV[1])
redis.call('HSET', KEYS[2], ARGV[1], ARGV[4])
redis.call('PEXPIRE', KEYS[1], lease)                         -- keys die when the newest lease does
redis.call('PEXPIRE', KEYS[2], lease)
return {1}
```

- Reading `TIME` and then writing is allowed because Redis (5.0+) replicates a script's effects, not the script itself — replicas don't re-run it at a different time.
- `PEXPIRE` with the lease length is safe because every lease has the same length: the keys expire exactly when the newest lease does, never earlier.
- Release is a two-command script: `ZREM` + `HDEL`.
- Redis replicates asynchronously, so a failover can forget the last few leases. That errs toward a few seconds of over-admission, which the requirements allow.

### 3. Enforce where a modified app can't skip it

- A modified app can ignore the 409 and replay an old manifest URL. Signed, short-lived URLs slow that down, but the real lock is **the DRM license**: streaming licenses are short-lived and renewable, and the license server renews only for sessions that still hold a lease. Stop heartbeating → lose the lease → renewal refused → playback stops when the license runs out.
- Widevine policies expose exactly these knobs (`can_renew`, `renewal_delay_seconds`, `license_duration_seconds`); PlayReady and FairPlay have equivalents.

### 4. Fail open — and say why

- If Redis is slow or down (no answer within ~20 ms), **grant** the stream, count it in a `fail_open` metric and alert. A circuit breaker stops a sick Redis from adding latency to every start.
- The abuse window is small and ends with the outage. A false denial for millions of paying viewers is a headline.
- **Multi-region:** home each account's lease set in one region (route by account, like a cell); if that region is down, the others fail open. Active-active replication that resolves conflicts last-writer-wins (the default for DynamoDB global tables) is wrong for counting — two regions could each grant the "last" slot. Strongly consistent multi-region modes exist, at a write-latency cost.

### 5. Extra Member

- An Extra Member is a separate identity under the same subscription, with their own login and a limit of 1: give them their own key (`streams:{acct:member}`) and take the limit from their entitlement.
- The main household's limit (2 or 4) is unaffected. Profiles don't matter to the count at all — the limit belongs to the subscription.

### 6. Household detection

- Disney hasn't published how it decides. Netflix documents using IP addresses, device IDs and account activity from signed-in devices, and that's the standard signal set: the networks a TV usually connects from, long-lived device history, where sign-ins happen, what travel looks like.
- Produce a **score, not a verdict**: soft prompts first, then a verification code sent to the account owner, temporary access for travel, and an Extra Member offer.
- Enforce mostly on TVs and streaming boxes — fixed devices anchor a household — and give phones and laptops leeway for travel.
- Measure false positives (verifications that turn out to be legitimate), and keep the data minimal and short-lived for privacy rules like GDPR.

### 7. "Who's watching" and remote stop

- The deny screen reads `streammeta:{acct}`: device names, profile names, maybe the title.
- Stopping a device = `evict` → its lease is gone → its next heartbeat or license renewal fails → it stops within one interval; a push can stop it immediately.
- "Sign out of all devices" = revoke refresh tokens (tracker #29) + evict every lease.

## LLD

```java
public interface StreamLeaseService {
    LeaseResult acquire(String accountId, String sessionId, int limit, StreamInfo info);   // also renews
    void release(String accountId, String sessionId);
    List<StreamInfo> active(String accountId);
}

public record StreamInfo(String sessionId, String deviceName, String profileName, String titleId,
                         Instant startedAt) {}

public sealed interface LeaseResult permits Granted, Denied {}
public record Granted() implements LeaseResult {}
public record Denied(List<StreamInfo> watching) implements LeaseResult {}
```

The Redis implementation — atomic script, tight timeout, and fail open:

```java
@Component
public class RedisStreamLeaseService implements StreamLeaseService {
    private static final Duration LEASE = Duration.ofSeconds(90);      // three missed 30 s heartbeats

    private final StringRedisTemplate redis;      // Lettuce client, command timeout ~20 ms
    private final RedisScript<List> acquireScript;   // the Lua script above
    private final CircuitBreaker breaker;         // Resilience4j
    private final Counter failOpen;

    @Override
    public LeaseResult acquire(String accountId, String sessionId, int limit, StreamInfo info) {
        List<String> keys = List.of("streams:{" + accountId + "}", "streammeta:{" + accountId + "}");
        try {
            List<?> reply = breaker.executeSupplier(() -> redis.execute(acquireScript, keys,
                    sessionId, String.valueOf(LEASE.toMillis()), String.valueOf(limit), Json.write(info)));
            return Long.valueOf(1).equals(reply.get(0)) ? new Granted() : new Denied(parseWatching(reply));
        } catch (RuntimeException storeDownOrSlow) {
            failOpen.increment();                 // alert on this rate
            return new Granted();                 // a false denial costs more than one extra stream
        }
    }
}
```

Renewing on each heartbeat — the limit comes from the token's entitlement claims, not from a call to billing:

```java
public HeartbeatResponse onHeartbeat(Heartbeat hb, Entitlement fromToken) {
    LeaseResult r = leases.acquire(hb.accountId(), hb.sessionId(), fromToken.maxStreams(), hb.info());
    return r instanceof Granted ? HeartbeatResponse.CONTINUE : HeartbeatResponse.STOP_TOO_MANY_DEVICES;
}
```

A stream's life: `REQUESTED → ACTIVE → (renewed every 30 s) → RELEASED | EXPIRED | EVICTED`.

## Failure modes

| Failure | Effect | Handling |
|---|---|---|
| Device crashes mid-show | Its slot is held until the lease runs out | 90 s lease; the deny screen lets the family stop it by hand |
| Network drop longer than the lease | Lease lost; if the slot was taken meanwhile, this stream stops | The player re-acquires on reconnect — fine if a slot is free |
| Redis failover | The last few leases are forgotten | Seconds of over-admission; acceptable |
| Redis down or slow | Streams can't be counted | Fail open, metric, alert |
| Clock skew between app servers | Leases expire early or late | The script uses Redis's own `TIME` |
| Retry storm at a premiere | Duplicate acquires | Idempotent by sessionId; client backoff with jitter |

## Trade-offs to say out loud

- **Lease length:** fast recovery after crashes vs renewal traffic and tolerance of network blips.
- **API check vs DRM enforcement:** tying licenses to leases is much stronger, but couples the license server to the lease store — so the license path must fail open too.
- **Fail open vs fail closed:** some revenue leakage vs outages for paying viewers. Fail open, and monitor it.
- **Redis vs DynamoDB:** Redis is fast and TTL-friendly but replicates asynchronously; DynamoDB is durable with conditional writes, but its default multi-region replication is last-writer-wins. Either way, home each account in one region (or pay for strongly consistent multi-region writes).
- **How strict to be about households:** revenue vs goodwill. Engineering's job is good signals and graceful prompts; the policy belongs to the product.

## Trick questions

- **"Why not INCR on play and DECR on stop?"** — Crashed devices never DECR, so slots leak until the family is locked out. Leases expire on their own.
- **"Two TVs press Play in the same millisecond with one slot left."** — The Lua script is atomic: Redis runs one, then the other, and the second finds the set full. With DynamoDB, the second conditional write fails on the version and its retry becomes a denial.
- **"A phone's battery dies; they switch to the TV and get 'too many devices'."** — That's the lease-length trade-off showing. Let them stop the phone from the deny screen, or allow a **takeover**: if the same profile starts the same title while the old session has missed a heartbeat, replace it instead of denying.
- **"The lease store dies during a premiere."** — Fail open. Say it before they ask.
- **"Can a modified app bypass the limit?"** — Not for long: license renewals require a live lease.
- **"Does a paused stream count?"** — Policy. A paused player keeps heartbeating, so it holds its slot; you could release after N minutes paused and re-acquire on resume.
- **"Do downloads count?"** — No — offline playback can't be counted live. Downloads have their own cap (10 devices) and licenses that expire.
- **"Why read the limit from token claims instead of asking billing on every heartbeat?"** — 330K requests/s against the billing database would be absurd. Claims are short-lived and refreshed when the plan changes ([note 30](30_Subscriptions_Billing.md)).
- **"They upgrade from Basic (2) to Premium (4) while at the limit."** — The entitlement change bumps a version, the token refreshes, and the next acquire uses 4.
- **"The same account streams from two regions (travel, VPN)."** — The account's leases live in its home region and both regions call it; over-admission is possible only while the regions are partitioned.

## Similar systems

| System | What changes |
|---|---|
| Netflix screens and households | The same leases; household checks use IP addresses, device IDs and account activity, with temporary codes for travel |
| Spotify | One device plays at a time per Premium account, and starting elsewhere pauses the first: a limit of 1 with **takeover** instead of denial |
| Floating software licenses (FlexLM-style) | Check out a seat, heartbeat, check it back in — the same lease pattern, decades older |
| Kindle and other ebook device limits | Counts registered devices, not live sessions — like the download cap in [note 17](17_Disney_Plus_Video_Streaming.md) |
| Per-tenant API concurrency limits | The same distributed semaphore, per customer |
| Database connection pools | A local semaphore; this note is the distributed version |

## Commonly asked

- Design "N screens at once". What happens when a device crashes without telling you?
- How do you make check-and-add atomic? Show the Redis or the DynamoDB version.
- How long should a lease be, and what are you trading?
- Where do you enforce the limit so a modified app can't bypass it?
- What does the system do if its datastore is down during a premiere?
- How would you detect sharing outside a household, and how strict would you be?

## Sources

- [How many people can watch Disney+ at once? (Disney+ Help Center)](https://help.disneyplus.com/article/disneyplus-en-ca-concurrent-streams) · [Streams per plan: 2 on Standard, 4 on Premium (Digital Trends)](https://www.digitaltrends.com/home-theater/how-many-screens-can-you-stream-disney-plus-on/)
- [Paid sharing on Disney+ (The Walt Disney Company)](https://thewaltdisneycompany.com/news/paid-sharing-disney-explainer/) · [Extra Member rules and the household prompt](https://becleverwithyourcash.com/disney-password-sharing-extra-member-rules-explained/)
