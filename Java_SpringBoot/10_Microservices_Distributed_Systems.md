# Microservices & Distributed Systems

## Why microservices (be ready to argue both sides)

- Pros: independent deployability, independent scaling, technology flexibility per service, fault isolation (in theory), smaller codebases per team.
- Cons: distributed system complexity (network calls replace function calls — latency, partial failure), data consistency across service boundaries, operational overhead (more services to monitor/deploy), harder local development/testing, distributed tracing needed to debug anything.
- A mature answer acknowledges microservices solve *organizational* scaling problems (team autonomy) as much as technical ones — a well-built monolith can outperform a poorly-decomposed set of microservices. Don't just recite "scalability."

## Service discovery

- Problem: in a dynamic environment (auto-scaling, containers), instances' IPs change — hardcoding endpoints doesn't work.
- Client-side discovery (e.g., Netflix Eureka + Ribbon historically) — client queries a registry, picks an instance itself (often with client-side load balancing).
- Server-side discovery (e.g., a load balancer/service mesh in front) — client calls a stable endpoint, the infra routes to a live instance. Kubernetes' built-in Service abstraction is the most common version of this today, often making a separate discovery library unnecessary.

## Externalized configuration

- Spring Cloud Config Server (or equivalent — Consul, Kubernetes ConfigMaps/Secrets) — centralizes config across services/environments instead of baking it into each jar, supports refresh without redeploy (`@RefreshScope` + `/actuator/refresh` in the Spring Cloud world).

## API Gateway

- Single entry point for external clients, fronting many internal services — handles cross-cutting concerns centrally: routing, authentication, rate limiting, request/response transformation, aggregation of multiple backend calls into one client-facing response.
- Examples: Spring Cloud Gateway, Netflix Zuul (older), Kong, NGINX, cloud-native equivalents (API Gateway services).
- Tradeoff to mention: it's a potential single point of failure and can become a bottleneck/God-object if too much logic accumulates there — needs its own scaling and resilience.

## Resilience patterns

- **Circuit breaker** (Resilience4j is the modern standard; Hystrix is deprecated/legacy) — after a failure threshold, stop calling a failing downstream service for a cooldown period (open state), fail fast instead of piling up latency/threads waiting on a dead dependency, then probe (half-open) before fully closing again.
- **Retry** — with backoff (ideally exponential + jitter) for transient failures; must be paired with idempotency on the called operation, or retries can cause duplicate side effects.
- **Timeout** — every network call needs one; without it, a slow downstream can exhaust caller thread pools/connections (cascading failure).
- **Bulkhead** — isolate resource pools (thread pools, connection pools) per dependency so one slow/failing dependency can't starve resources needed by calls to other, healthy dependencies.
- **Rate limiting** — protect a service from being overwhelmed by its callers (token bucket / leaky bucket are the common algorithms to name).

## Messaging & async communication

- Sync (REST/gRPC request-response) vs async (message queue/event stream) — async decouples services in time (producer doesn't need consumer to be up) and can smooth load spikes, at the cost of eventual consistency and more complex failure/ordering reasoning.
- Message queue (RabbitMQ, SQS) — typically point-to-point or pub/sub with competing consumers, good for task distribution/work queues.
- Event stream (Kafka) — durable, ordered (per-partition) log, supports multiple independent consumers replaying/re-reading, better fit for event-sourcing/streaming analytics patterns and high-throughput pipelines.
- At-least-once vs exactly-once vs at-most-once delivery — exactly-once is very hard/expensive in practice; most systems design for **at-least-once + idempotent consumers** instead of chasing true exactly-once.

## Distributed transactions & the Saga pattern

- Two-phase commit (2PC) exists but is rarely used across microservices in practice — it's blocking, doesn't scale well, and ties services tightly together operationally.
- **Saga pattern** — a sequence of local transactions, each publishing an event/triggering the next step; if a step fails, compensating transactions undo the prior steps' effects (since you can't roll back a distributed transaction atomically). Two flavors: **choreography** (services react to each other's events, no central coordinator — simpler but harder to see the overall flow) and **orchestration** (a central saga orchestrator drives the sequence explicitly — easier to reason about, adds a coordinating component).
- Idempotency keys are the practical mechanism that makes retries and at-least-once delivery safe in this world — the consumer stores/checks a request ID to avoid double-processing.

## CAP theorem

- In the presence of a network **P**artition, you must choose between **C**onsistency (every read gets the latest write, or an error) and **A**vailability (every request gets a response, possibly stale). You cannot have both during a partition — partitions are a fact of distributed systems, not optional, so this is really "C vs A when P happens," not a three-way free choice.
- Most real systems aren't purely CP or AP everywhere — different operations within the same system make different tradeoffs (e.g., inventory count might favor consistency, product catalog display might favor availability).

## Commonly asked

- What problems do microservices solve, and what do they make harder? Give a concrete example of each.
- Explain the circuit breaker pattern and its three states.
- Why must retries be paired with idempotency, and how would you implement an idempotent POST endpoint?
- Explain the Saga pattern and the difference between choreography and orchestration.
- Kafka vs a traditional message queue (e.g., RabbitMQ/SQS) — when would you pick each?
- Explain CAP theorem in your own words with a concrete partition scenario.
