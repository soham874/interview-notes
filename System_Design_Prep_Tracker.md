# System Design Prep Tracker

Companion to `Java_SpringBoot_Prep_Tracker.md` — same idea, for the design round, aimed at Disney. `System_Design/00_Disney_System_Design_Questions.md` covers which Disney org asks what and what candidates have reported; this file is the map of every system worth practising, plus the progress log. Each system gets its own note in `System_Design/` as we go, numbered to match this list; a system's name links to its note once it exists.

**Priority:** ★ reported by a candidate in a Disney, Hulu or Hotstar interview · ◆ named by prep guides, or a direct model of a Disney product — likely · ○ classic or supporting — good to know.

**Asked by:** Streaming (Disney+, Hulu) · ESPN · Ads · Supply chain (the media supply chain) · Parks (Disney Experiences: parks, resorts, cruise, shopDisney) · Any.

Numbers are IDs, not positions: a system keeps its number, and a new one gets the next free number in whichever group it belongs to.

## Which order

- **No fixed date:** go in order. Group A sets up the answer template (requirements → estimates → API → data model → architecture → deep dive) on small problems, and each later group adds one new idea on top. Slot the LLD ones (group L) in whenever you want a shorter session.
- **Interview soon:** the ★ ones first — 14 → 26 → 25 → 6 → 20 → 24 → 1 — then your org's ◆ ones: Parks 7, 12, 39, 40 · Streaming 17, 18, 16, 30 · ESPN 18, 21 · Supply chain 33, 32 · Ads 37. If the loop has an LLD round, add 43, 44, 45.

## How each session runs

1. I pose the prompt the way an interviewer would. Take a first pass yourself, or say "walk me through it".
2. We cover it fully: clarifying questions, estimates, API and data model, architecture, the deep dive on the hard part, failure modes, trade-offs, and the follow-ups interviewers ask.
3. It's saved as `System_Design/NN_Name.md`, ending with a "commonly asked" list like the Java notes; the row below gets a confidence score and date.

When every row in a group sits at 4+, do one timed 45-minute mock on a system from that group, out loud — I can play the interviewer.

## A. Foundations — building blocks every later design reuses

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 1 | ★ URL shortener | ID generation (counter + base62 vs hashing), read-heavy caching, 301 vs 302 redirects, click analytics | Any (Hulu) | | |
| 2 | ○ Rate limiter | Token bucket vs sliding window, Redis counters with Lua, enforcing at the gateway, fail open vs closed | Any | | |
| 3 | ○ Distributed cache | Consistent hashing, eviction, cache-aside vs write-through, stampede and hot-key protection (the LRU class itself is #45) | Any | | |
| 4 | ◆ Notification service | Push/email/SMS channels, user preferences, dedupe and retries, provider rate limits, priorities | Any | | |
| 5 | ○ Distributed job scheduler | Time-bucketed scheduling, leases so a job runs once, retries with idempotency | Any | | |

## B. Booking & inventory under contention — the Ticketmaster family

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 6 | ★ Movie ticket booking (BookMyShow) — do the LLD and schema too | Seat holds with a TTL, pessimistic vs optimistic locking, keeping booking and payment consistent | Any (Hotstar) | | |
| 7 | ◆ Lightning Lane return-time booking | Capacity counters with guarded updates, per-guest limits, idempotency keys, the 7 AM surge, sharded counters | Parks | | |
| 8 | ◆ Dining reservations | Inventory per time slot × table size, booking windows that open at a fixed time, cancellations and waitlists | Parks | | |
| 9 | ◆ Hotel & cruise booking | Date-range inventory (all nights or none), lock ordering, TTL holds, a hold → pay → confirm saga, overbooking | Parks | | |
| 10 | ◆ Park tickets, pricing & capacity | Date-based pricing rules, per-day capacity, multi-day tickets and passes with blockout dates | Parks | | |
| 11 | ○ Mobile food ordering | Pickup windows capped by kitchen capacity, an order state machine, the "I'm here" flow | Parks | | |

## C. Flash traffic & fairness

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 12 | ◆ Virtual queue (boarding groups) | Exactly-once allocation in a one-second stampede, pre-issued eligibility tokens, first-come vs lottery, calling groups | Parks | | |
| 13 | ◆ Limited-edition merch drop | A virtual waiting room, TTL inventory holds, per-customer limits, bot defense, no overselling | Parks (shopDisney) | | |

## D. User state at huge write volume

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 14 | ★ Continue Watching (+ recommendations row) | Heartbeats through a stream, coalescing, conditional writes so the newest position wins, NoSQL modeling, multi-region | Streaming | | |
| 15 | ◆ Watchlist | Per-profile CRUD at huge scale, idempotent add/remove, read-your-writes across devices | Streaming | | |
| 16 | ◆ [Concurrent stream limits & household sharing](System_Design/16_Concurrent_Stream_Limits.md) | A distributed semaphore with leases, heartbeat renewal, DRM-backed enforcement, fail open vs closed | Streaming | | |

## E. Video streaming

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 17 | ◆ [Disney+ video streaming (on demand, incl. downloads)](System_Design/17_Disney_Plus_Video_Streaming.md) | Transcoding ladder, HLS/DASH + DRM, CDNs, entitlement at play time, time to first frame, offline licenses | Streaming | | |
| 18 | ◆ [Premiere / live-event traffic spike](System_Design/18_Premiere_Traffic_Spike.md) | Forecast-based pre-scaling, panic mode (load shedding), retry storms, protecting the playback path | Streaming, ESPN | | |
| 19 | ○ Live sports streaming | Live ingest and transcoding, low-latency HLS/DASH, the DVR window, ad-break markers, glass-to-glass latency | ESPN | | |

## F. Real-time fan-out

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 20 | ★ Live scores & alerts ("an ESPN-like system") | An ordered per-game stream, CDN polling vs sockets, push to tens of millions of phones, corrections | ESPN | | |
| 21 | ◆ Fantasy sports: draft room + live scoring | Stateful WebSocket rooms, a server-side pick clock, optimistic concurrency on picks, player → team fan-out | ESPN | | |
| 22 | ◆ Live chat | WebSocket gateways, presence, message ordering and storage, very large rooms | Any | | |
| 23 | ○ Watch party (synchronized playback) | Shared control messages, clock-offset estimation, drift correction | Streaming | | |
| 24 | ★ Photo-sharing feed (Instagram) | Upload pipeline, fan-out on write vs on read, the celebrity problem, feed ranking | Any (Hotstar) | | |

## G. Ranking, personalization & search

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 25 | ★ Global game leaderboard | Sorted sets, rank queries, ties, sharding to 100M players, season settlement | Any (games) | | |
| 26 | ★ Home page & recommendations | Offline candidates + online re-ranking, rows with per-row timeouts, cold start, fallbacks | Streaming | | |
| 27 | ◆ Search & autocomplete (titles, products) | Inverted index, typeahead, ranking, faceted browsing, filtering by country availability | Streaming, Parks (shopDisney) | | |
| 28 | ○ Trending / "Top 10 today" | Streaming top-K (count-min sketch + heap), time windows, per-region lists | Streaming | | |

## H. Identity, entitlements & money

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 29 | ◆ MyDisney single sign-on | OAuth2/OIDC, local JWT validation, TV device-code login, sign out everywhere, credential stuffing | Any | | |
| 30 | ◆ [Subscriptions, bundles & billing](System_Design/30_Subscriptions_Billing.md) | Billing vs entitlements, idempotent webhooks, a subscription state machine, renewals and retries | Streaming | | |
| 31 | ○ Payments & checkout | Idempotency keys, a payment state machine, provider webhooks, reconciliation, refunds | Parks, Any | | |
| 32 | ◆ Catalog, rights & availability | Rules by country/date/plan/device, snapshots precomputed before windows open, the final check at play time | Streaming, Supply chain | | |

## I. Content supply chain

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 33 | ◆ Media ingest & transcoding pipeline | Huge-file uploads, workflow orchestration, chunked parallel transcoding, resume instead of restart, priorities | Supply chain | | |
| 34 | ○ Media asset management & delivery workflow | Title/version/component model, publishing gated on dependencies, per-destination packages, securing unreleased content | Supply chain | | |

## J. Data, experiments & ads

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 35 | ○ Playback quality (QoE) telemetry | Event ingestion at volume, stream processing with watermarks, OLAP stores, sampling, cardinality | Streaming | | |
| 36 | ◆ A/B testing & feature flags | Deterministic bucketing, config propagation, kill switches, experiment metrics | Any (a Bengaluru posting mentions A/B/N testing) | | |
| 37 | ◆ Ad insertion & decisioning (SSAI) | Manifest stitching, deciding inside a latency budget, frequency caps, live ad-break stampedes | Ads | | |
| 38 | ○ Ad impression counting | Exactly-once-ish counting, dedupe, late events, reconciliation for billing | Ads | | |

## K. Parks — edge & IoT

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 39 | ◆ Park gate entry (works offline) | Signed entitlements at the edge, local decisions, sync and revocation, choosing what risk to accept offline | Parks | | |
| 40 | ◆ Ride status & wait times | Edge aggregation, surviving network partitions, one writer per ride, Little's law, smoothing | Parks | | |
| 41 | ○ Tap to do everything (band or phone as ticket, room key, payment) | Opaque device IDs, offline door credentials, online payment authorization, fast revocation | Parks | | |
| 42 | ○ Ride & character photo matching | Tagging at capture, time-window matching, an image pipeline, retention and purchase rights | Parks | | |

## L. Low-level (object-oriented) design

| # | System | What it teaches | Asked by | Conf. (1–5) | Last reviewed |
|---|---|---|---|---|---|
| 43 | ★ Calculator with nested operations | Composite / Interpreter, a recursive-descent parser | Any (Disney) | | |
| 44 | ★ Badge door-access system | Access policies by role and schedule, an audit log — Strategy / Chain of Responsibility | Any (Hulu) | | |
| 45 | ★ LRU cache | HashMap + doubly linked list for O(1) get/put, thread safety | Any (Hulu) | | |
| 46 | ◆ Video player | State pattern: Idle, Loading, Playing, Paused, Buffering, Ended, Error | Streaming | | |
| 47 | ◆ Ride queue with Lightning Lane merge | Strategy for the merge ratio, Observer to notify parties | Parks | | |
| 48 | ◆ Subscription plans & pricing | Strategy / Decorator for discounts and bundles, versioned plans | Streaming | | |
| 49 | ○ Parking lot | The standard warm-up: entities, a spot-allocation strategy, fees | Any | | |
| 50 | ○ Elevator system | Dispatch strategies, a per-car state machine, request queues | Any | | |

## Running log

Freeform — dated entries on what clicked, what's still fuzzy, what came up in a real interview that isn't covered yet.

-
