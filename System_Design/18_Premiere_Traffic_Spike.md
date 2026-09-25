# Surviving a Premiere or Live-Event Traffic Spike

**The prompt:** "A new Marvel series drops at a fixed time, or a huge live game kicks off. Traffic goes from normal to 10–50× in minutes — sometimes seconds. How does the platform survive?" Prep guides list it as "a video streaming service that can handle a surge of concurrent viewers during a major content launch". It tests whether you know **why reactive autoscaling fails**, how to **prioritise and shed load**, how **retry storms** turn a blip into an outage, and how to plan capacity before the day. Related: [17 · Disney+ video streaming](17_Disney_Plus_Video_Streaming.md), [16 · stream limits](16_Concurrent_Stream_Limits.md).

**The reference case:** Hotstar (Disney-owned at the time) served 25M concurrent viewers in 2019 and explained how at AWS re:Invent, and Disney+ Hotstar peaked at **59M concurrent viewers** for the 2023 Cricket World Cup final.

## What "survive" means — clarify first

- Name the **P0 path**: an already-valid session → entitlement → start playback → manifest + license → CDN segments → heartbeats. Everything else may degrade: personalized rows, search, the accuracy of "continue watching", social features, detailed telemetry.
- Set event SLOs for that path, e.g. ≥ 99.5% of starts succeed, video start p95 ≤ 3 s, rebuffer ratio ≤ 1%.
- **Forecast with uncertainty:** a peak-concurrency estimate and a band around it. Build for ~1.5–2× the forecast.
- Ask which kind of spike: a **premiere** (on demand — everyone starts within minutes, then it's steady) or a **live event** (waves at kickoff, halftime and big moments, and everyone watching the same live edge)?

## Anatomy of the spike

| When | What spikes | Where it lands |
|---|---|---|
| T−30 min | App opens, home page, session refreshes, sign-ups | API gateway, identity, catalog, payments |
| T0, first 1–5 min | Everyone presses Play | Playback, license and stream-limit services; CDN first segments |
| During, live | Joins and leaves at big moments; everyone requests the newest segment | CDN edges and origin shield, live packagers |
| Throughout | Heartbeats, progress, telemetry — in proportion to concurrency | Write paths: lease store, bookmarks, Kafka |
| After any blip | Every client retries at once | Everything — this turns a blip into an outage |

**Numbers:** 20M viewers arriving over ~5 minutes ≈ 67K starts/s on average, with the peak minute maybe 3× that (~200K/s). Each start is one API call + one manifest + one license + ~5 internal calls → roughly **1M internal requests/s**. Heartbeats: 20M ÷ 30 s ≈ 670K/s. Egress: 20M × 5 Mbps = **100 Tbps**, which needs capacity reserved across several CDNs.

## Architecture for the event

```text
 Clients: jittered retries, cached remote config, prefetched content
   │
   ↓
 CDN / edge: static home page, cached catalog and manifests, WAF,
             rate limits, pre-positioned encrypted segments
   │
   ↓
 API gateway: tag each request P0–P3, shed the lowest first,
              adaptive concurrency limits, cheap 503 + Retry-After
   │
   ├──→ P0  playback, license, stream limits   pre-scaled, own bulkheads
   ├──→ P1  browse, catalog                    cache first
   ├──→ P2  personalization, search            first to go
   └──→ writes: heartbeats, progress, QoE ──→ Kafka, drained later

 Panic-mode flags ←── war room: SLO dashboards, runbooks, owners
```

## The playbook, by phase

### Weeks before — capacity

- **Forecast** peak concurrency from comparable launches, pre-registrations, trailer and marketing numbers, and time zones. Turn it into a capacity model per service: calls per start × starts per second → instances, connections, database throughput.
- **Load-test above the forecast** with the real traffic mix (not one endpoint in isolation) and real client behaviour (retries, jitter, cold caches). Inject failures: lose a zone, degrade a CDN. Game days are rehearsals, not demos.
- **Raise limits:** cloud quotas, database throughput, load-balancer pre-warming (ask the provider), CDN capacity commitments across several CDNs.
- **Prune the P0 path:** playback must not synchronously call recommendations, search or anything else optional. Give P0 services their own bulkheads — thread pools, connection pools, even clusters.

### Hours before — pre-scale and pre-warm

- **Scale out ahead of demand, in steps.** Hotstar described pre-scaling in "ladders" keyed to expected concurrency, holding a buffer, and treating reactive autoscaling only as a top-up.
- **Warm caches:** the title's metadata, artwork and manifests; pre-position the first minutes of every rendition on CDN edges.
- **Pre-authenticate:** long-lived sessions and early token refreshes, so kickoff doesn't hit identity.
- **Freeze deploys**, arm the kill switches, staff the war room.

### T0 — the ramp

- The content is already on the edges, **encrypted**; the license service starts handing out keys at T0 ([note 17](17_Disney_Plus_Video_Streaming.md)), so there's no origin rush at release.
- Clients stagger background work with jitter; the server can push `Retry-After` values and heartbeat intervals through remote config.
- If starts truly exceed capacity, an admission queue ("you're in line — starting in 20 s") beats a crash. It's the last resort, but have it ready.

### During — control loops

- Watch the P0 SLOs; let load shedding and adaptive limits do their job; step through panic levels deliberately.
- Reactive autoscaling helps only at the margins (deep dive 1).

### After

- Drain the buffered writes; scale down slowly (people pause and come back); write the post-mortem while it's fresh.

## Deep dives

### 1. Why "just autoscale" fails

- Metrics are aggregated per minute, the alarm needs 1–3 periods, new instances take 1–3 minutes to boot, and the app needs another minute or two to warm up (JIT, connection pools, caches). That's **5–10 minutes**; a premiere ramps in 1–2.
- CPU lags, too: queues build before CPU saturates, latency climbs, clients time out and retry, and the load multiplies.
- So: scale ahead of time on a schedule; scale on **leading signals** (request rate, concurrency, active sessions) rather than CPU; keep warm pools and headroom.

### 2. Load shedding and priorities

- Tag requests at the edge: **P0** start playback, license, heartbeats; **P1** browse and catalog; **P2** personalization and search; **P3** background work (detailed telemetry, social, prefetching).
- Under pressure, reject the lowest priority first with a **cheap** `503` + `Retry-After`. A request rejected at the gateway costs microseconds; one that times out after touching five services wasted all five.
- **Adaptive concurrency limits:** cap in-flight requests per service and shrink the cap when latency rises — AIMD, like TCP congestion control (Netflix's concurrency-limits library does this). Short queues keep latency low for the requests you do admit.
- **Deadline propagation:** pass the caller's remaining time budget downstream, and drop work whose caller has already given up.

### 3. Panic mode — degradation decided in advance

| Level | What turns off | What viewers notice |
|---|---|---|
| L1 | P3: telemetry sampled to ~1%, social features, prefetching | Nothing |
| L2 | P2: personalized rows and search served from cache; heartbeats every 120 s instead of 30 s | A slightly staler home page |
| L3 | P1 through the API: a static, per-region home page from the CDN; stream-limit checks fail open; bookmarks paused | A generic home page — but Play works |

- Flipped by remote config in seconds, rehearsed on game days, with owners and triggers written down beforehand.
- Careful with heartbeats: if stream leases and license renewals depend on them ([note 16](16_Concurrent_Stream_Limits.md)), **shedding heartbeats stops playback**. Lengthen the interval, and the lease with it, instead of dropping them.

### 4. Retry storms and thundering herds

- A 30-second blip makes millions of clients retry in lock-step and pushes load to 2–10× normal, so the system can't recover even after the original cause is gone. That's a **metastable failure**.
- Clients: exponential backoff with **full jitter**; respect `429`/`503` + `Retry-After`; cap the attempts.
- Servers: **retry budgets** (retries capped at ~10–20% of requests) and retries at **one layer only** — 3 retries at each of 4 layers is up to 4⁴ = 256 calls for one click.
- **Synchronized timers:** tokens issued in the same minute expire in the same minute. Jitter every TTL and refresh interval.
- To recover: cut admission (let a fraction in), warm the caches, then open up gradually.

### 5. Cache stampedes

- A hot key — the premiere's title record — expires, and thousands of concurrent misses hit the database at once.
- Fixes: **request coalescing** (one loader per key while everyone else waits on it — see the LLD), **stale-while-revalidate** (serve the old value while one request refreshes), **probabilistic early refresh**, and pre-warming with long TTLs plus explicit invalidation for event keys.

### 6. What's different about live sports

- Every viewer wants the **newest segment** every 2–6 s, so the live edge is one very hot object. Requests for a segment that isn't written yet get a 404 — make sure the CDN doesn't cache those (negative caching), and that the shield collapses concurrent requests.
- Joins and leaves come in waves (kickoff, halftime, a wicket); viewers who rewind spread load back along the timeline.
- Run redundant encoders and packagers (two independent pipelines into two origins): a live stream can't be re-encoded later.
- Low-latency modes (sub-segment "parts") multiply request rates — use them only where latency really matters.

### 7. The dependencies that break first

- **Identity:** sessions must already be valid, and services validate tokens locally, so signing in is the only thing that fails if identity struggles.
- **Sign-up and payment:** people subscribe minutes before a premiere. Give checkout its own capacity, queue payment confirmations, and grant access provisionally while the charge settles ([note 30](30_Subscriptions_Billing.md)).
- **Databases:** reads go to caches and replicas; bursty writes go through a queue and drain at a sustainable rate (queue-based load levelling). Lag is fine; falling over isn't.
- **Third parties** — payment providers, ad servers, app-store APIs — have their own limits. Know them before the day.

### 8. Watching it live

- Real time: concurrency, starts/s, start success, video start p95, rebuffer ratio, errors by CDN × ISP × region, SLO burn rate, sign-ups/s.
- Synthetic probes on the P0 path from several regions; automated triggers for panic levels, with a human confirming.

## LLD

A priority load shedder at the gateway or in each service:

```java
public enum Priority { P0_PLAYBACK, P1_BROWSE, P2_PERSONALIZATION, P3_BACKGROUND }

@Component
public class PriorityLoadShedder extends OncePerRequestFilter {
    private final AdaptiveLimiter limiter;
    private final PanicMode panic;                        // current level, pushed by remote config

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        Priority p = RouteTable.priorityOf(req);          // /playback/** → P0, /home/** → P1, …
        if (panic.level().sheds(p) || !limiter.tryAcquire(p)) {
            res.setStatus(503);
            res.setHeader("Retry-After", String.valueOf(5 + ThreadLocalRandom.current().nextInt(20)));
            return;                                       // cheap: no downstream work was done
        }
        try {
            chain.doFilter(req, res);
        } finally {
            limiter.release();
        }
    }
}
```

Panic levels as data, so a config push changes behaviour without a deploy:

```java
public enum PanicLevel {
    NORMAL, L1, L2, L3;

    public boolean sheds(Priority p) {
        return switch (this) {
            case NORMAL -> false;
            case L1 -> p == Priority.P3_BACKGROUND;
            case L2 -> p.compareTo(Priority.P2_PERSONALIZATION) >= 0;
            case L3 -> p.compareTo(Priority.P1_BROWSE) >= 0;   // the static home page comes from the CDN
        };
    }
}
```

An adaptive limiter that keeps headroom for P0:

```java
public class AdaptiveLimiter {
    private static final double[] SHARE = {1.0, 0.8, 0.6, 0.4};   // P3 may use only 40% of the limit
    private static final Duration TARGET_P99 = Duration.ofMillis(250);
    private static final int MIN_LIMIT = 20, MAX_LIMIT = 2_000;
    private final AtomicInteger inFlight = new AtomicInteger();
    private final AtomicInteger limit = new AtomicInteger(200);

    public boolean tryAcquire(Priority p) {
        int allowed = (int) (limit.get() * SHARE[p.ordinal()]);
        while (true) {
            int cur = inFlight.get();
            if (cur >= allowed) return false;
            if (inFlight.compareAndSet(cur, cur + 1)) return true;
        }
    }

    public void release() { inFlight.decrementAndGet(); }

    /** Called every 100 ms with the window's p99 latency: AIMD, like TCP congestion control. */
    void adjust(Duration windowP99) {
        limit.updateAndGet(l -> windowP99.compareTo(TARGET_P99) > 0
                ? Math.max(MIN_LIMIT, (int) (l * 0.9))       // slow: cut multiplicatively
                : Math.min(MAX_LIMIT, l + 5));                // healthy: grow additively
    }
}
```

Request coalescing for hot keys ("single flight"):

```java
public class SingleFlight<K, V> {
    private final ConcurrentHashMap<K, CompletableFuture<V>> inFlight = new ConcurrentHashMap<>();

    public CompletableFuture<V> load(K key, Function<K, CompletableFuture<V>> loader) {
        CompletableFuture<V> mine = new CompletableFuture<>();
        CompletableFuture<V> running = inFlight.putIfAbsent(key, mine);
        if (running != null) return running;              // already loading: wait on that one
        try {
            loader.apply(key).whenComplete((value, error) -> {
                inFlight.remove(key, mine);
                if (error != null) mine.completeExceptionally(error); else mine.complete(value);
            });
        } catch (RuntimeException e) {
            inFlight.remove(key, mine);
            mine.completeExceptionally(e);
        }
        return mine;
    }
}
```

Client retries — exponential backoff with full jitter, honouring `Retry-After`:

```java
Duration nextDelay(int attempt, Optional<Duration> retryAfter) {
    ThreadLocalRandom rnd = ThreadLocalRandom.current();
    if (retryAfter.isPresent()) return retryAfter.get().plusMillis(rnd.nextLong(2_000));  // spread the herd
    long capMs = 30_000, baseMs = 500;
    long ceiling = Math.min(capMs, baseMs << Math.min(attempt, 10));
    return Duration.ofMillis(rnd.nextLong(ceiling + 1));                                   // "full jitter"
}
```

## Trick questions

- **"Why not just autoscale?"** — It reacts in 5–10 minutes and the ramp takes 1–2. Pre-scale, and scale on request rate rather than CPU.
- **"Why not run 10× capacity all year?"** — Cost — and it wouldn't even work: bottlenecks aren't linear (database connections, hot partitions, locks, third-party limits). Only a realistic load test finds them.
- **"The cause is fixed, but the system is still down. Why?"** — A metastable failure: retries and cold caches keep load above capacity. Shed load, cut admission, warm the caches, then reopen gradually.
- **"What do you shed first?"** — By priority: background, then personalization, then browse. Never heartbeats if leases and licenses depend on them — lengthen the interval instead.
- **"Should clients use longer timeouts during the event?"** — No: long timeouts pile up in-flight requests on the servers. Keep server timeouts short, propagate deadlines, and show progress in the app.
- **"Everyone's session token expires at 6:05 PM."** — Because they were all issued at 5:05. Jitter TTLs, and refresh early, before the event.
- **"The CDNs are the bottleneck."** — Reserve capacity across several CDNs, and temporarily cap the default bitrate (Netflix and YouTube lowered streaming quality across Europe for a month in March 2020 to ease network load).
- **"Would you put a waiting room in front of a streaming premiere?"** — For playback, only as a last resort; pre-scaling is the answer. For sign-up and payment spikes, a queue is perfectly reasonable.
- **"How do you know your load test was realistic?"** — It replayed the production mix, included client retry behaviour and cold caches, ran above the forecast, and injected failures. Anything less tests the happy path.
- **"Isn't the content exposed if it's on the CDN before T0?"** — It's encrypted, and the keys arrive at T0 ([note 17](17_Disney_Plus_Video_Streaming.md)).

## Similar systems

| System | What's the same, what changes |
|---|---|
| Ticketmaster on-sales (the 2022 Eras Tour presale meltdown) | A synchronized rush plus bots; inventory contention makes it harder (tracker #6) |
| E-commerce sales (Flipkart's Big Billion Days, Amazon Prime Day) | Pre-scaling, queues, degraded features, checkout protected as P0 |
| IRCTC Tatkal booking at 10 AM | A daily synchronized stampede where admission control and fairness matter most |
| Election-night results sites | Almost all reads: static pages at the CDN and short TTLs do most of the work |
| Game launches and MMO login queues | The waiting room built into the product as its safety valve |
| Disney+ Hotstar cricket | The streaming reference: ladder pre-scaling and panic mode, from 25M (2019) to 59M (2023) concurrent viewers |

## Commonly asked

- A premiere is expected to bring 10× normal traffic. Walk me through your plan: weeks before, hours before, T0, during.
- Why does reactive autoscaling fail for spikes, and what do you do instead?
- What is load shedding? How do you decide what to shed, and how do you make shedding cheap?
- What's a retry storm, and how do backoff, jitter and retry budgets prevent one?
- What's a cache stampede? Give three fixes.
- How is a live sports spike different from a VOD premiere?
- How do you know you're ready before the day?

## Sources

- [Scaling hotstar.com for 25 million concurrent viewers (AWS re:Invent 2019)](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf) · [How Hotstar scaled: ladders and panic mode (System Design newsletter)](https://newsletter.systemdesign.one/p/hotstar-scaling)
- [59M peak concurrent viewers for the 2023 World Cup final (Business Standard)](https://www.business-standard.com/cricket/world-cup/world-cup-ind-aus-match-records-peak-viewership-of-59-mn-on-disney-hotstar-123111900827_1.html)
