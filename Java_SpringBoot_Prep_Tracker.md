# Java & Spring Boot Interview Prep Tracker

Companion to `DSA_Restart_Plan.md` — same idea, different track: no fixed interview date, so the goal is building real depth and recognition on core Java/Spring topics rather than cramming a list. Notes live in the `Java_SpringBoot/` folder, one file per topic area. This file is just the map + progress log.

**You also have `Technical.pdf`** (42 pages, Q&A format) covering core Java fundamentals. The two sets are complementary — the PDF is strong on OOP/threads/collections/memory, and near-empty on Spring. The "PDF?" column below says where you already have coverage, so you can spend effort on the gaps rather than re-reading what you know. **Read `00_PDF_Corrections.md` first** — six entries in that PDF are wrong or outdated, and two of them (wait/sleep, volatile) are classic interview questions where the wrong answer is costly.

## Notes files (in `Java_SpringBoot/`)

| # | File | Covers | PDF? |
|---|---|---|---|
| 00 | PDF_Corrections.md | Errors in `Technical.pdf` + what it gets right | — |
| 01 | Core_Java_OOP.md | OOP pillars, association/aggregation/composition, interfaces vs abstract classes, static/final/transient, binding, wrappers, pass-by-value, equals/hashCode, immutability, exceptions, generics, lambdas, Streams | Strong |
| 02 | Collections_Framework.md | Hierarchy, all the pairwise comparisons, HashMap internals, ConcurrentHashMap, Comparable vs Comparator, iterator behavior | Strong |
| 03 | Concurrency_Multithreading.md | Thread vs process, states, synchronized/volatile, wait vs sleep, locks, ExecutorService, CompletableFuture, deadlocks, coding exercises | Strong |
| 04 | JVM_Memory_GC.md | Memory areas, heap vs stack, mark-sweep-compact, GC algorithms, class loading, JIT, leak debugging | Strong |
| 05 | Spring_Core_DI_AOP.md | IoC container, bean lifecycle/scopes, injection types, circular dependency, AOP, proxies | **None** |
| 06 | Spring_Boot_Fundamentals.md | Auto-configuration, starters, `@SpringBootApplication`, profiles, Actuator, Tomcat, 2.x→3.x→4.x, AOT | Partial |
| 07 | Spring_MVC_REST_API.md | REST design, controller annotations, exception handling, validation, status codes | **None** |
| 08 | Spring_Data_JPA_Hibernate.md | JPA vs Hibernate, entity lifecycle, lazy/eager, N+1, `@Transactional`, caching | **None** |
| 09 | Database_Transactions.md | ACID, isolation levels, propagation, locking, indexing basics | **None** |
| 10 | Microservices_Distributed_Systems.md | Service discovery, gateway, circuit breakers, messaging, saga, CAP theorem | **None** |
| 11 | Testing_JUnit_Mockito.md | Unit vs integration tests, Mockito, Spring test slices, Testcontainers | **None** |
| 12 | Spring_Security_Basics.md | Authn vs authz, filter chain, JWT, OAuth2, CORS/CSRF | **None** |
| 13 | Design_Patterns_Java.md | Singleton, Factory, Builder, Strategy, Observer, Proxy — with Spring examples | Singleton only |
| 14 | JDBC.md | API components, `PreparedStatement`, connection pooling, batching, `JdbcTemplate` | Strong |
| 15 | SQL_Query_Practice.md | Joins, GROUP BY/HAVING, window functions, the standard query questions, index gotchas | **Empty section** |

Each file ends with a short "commonly asked" list — use those as your self-check before marking confidence below.

**Where to start:** everything marked "None" above is both untested and interview-critical for a Spring Boot role — that's roughly files 05, 07–12 plus SQL. The PDF-covered topics need review, not learning.

## Phase 0 — Diagnostic (do once, untimed)

For each topic, try to answer the "commonly asked" questions at the bottom of that file from memory, no notes. Note what came back on its own vs. what felt foreign — this tells you where Phase 1 should actually start, not necessarily file order.

| Topic | Gut reaction (solid / shaky / blank) | Notes |
|---|---|---|
| Core Java & OOP | | |
| Collections | | |
| Concurrency | | |
| JVM / Memory / GC | | |
| Spring Core (DI/AOP) | | |
| Spring Boot fundamentals | | |
| Spring MVC / REST | | |
| Spring Data JPA / Hibernate | | |
| Database & Transactions | | |
| SQL query writing | | |
| JDBC | | |
| Microservices / Distributed | | |
| Testing | | |
| Spring Security | | |
| Design Patterns | | |

## Phase 1 — Topic-by-topic build

Work shaky/blank topics first. For each: read the notes file, close it, explain the topic out loud (or in writing) as if to an interviewer, then check yourself against the file again. Update confidence weekly, not daily — this isn't a race.

| Topic | Confidence (1–5) | Last reviewed | What's still shaky |
|---|---|---|---|
| Core Java & OOP | | | |
| Collections | | | |
| Concurrency | | | |
| JVM / Memory / GC | | | |
| Spring Core (DI/AOP) | | | |
| Spring Boot fundamentals | | | |
| Spring MVC / REST | | | |
| Spring Data JPA / Hibernate | | | |
| Database & Transactions | | | |
| SQL query writing | | | |
| JDBC | | | |
| Microservices / Distributed | | | |
| Testing | | | |
| Spring Security | | | |
| Design Patterns | | | |

When most rows sit at 4+, move to Phase 2.

## Phase 2 — Applied practice (not yet)

Once the topics feel solid individually, the gap that's left is usually *wiring things together under pressure*. This phase is about that, not new material:

- Build a small end-to-end feature from scratch (a REST CRUD service backed by Spring Data JPA, with validation, exception handling, and a couple of unit + integration tests) without copy-pasting boilerplate — time how long it takes and where you hesitate.
- Do a few "design a service that..." questions out loud (e.g., a rate limiter, a URL shortener, an order service with idempotent retries) — practice narrating tradeoffs, not just naming buzzwords.
- Mix in "why" follow-ups on your own past projects: why constructor injection over field injection, why you chose a given isolation level, why a queue over a direct call, etc. Interviewers probe here more than on textbook definitions.
- Do 2–3 mock interviews (or record yourself) explaining a design decision end to end.

SQL is the exception to "phase 2 only" — it's a hands-on skill that decays without typing, so start working actual query problems (LeetCode's database section, StrataScratch, or DataLemur) during phase 1, alongside the reading. `15_SQL_Query_Practice.md` lists the patterns that keep recurring; aim to recognize which pattern a question is before writing anything.

Same goes for the two threading exercises in `03_Concurrency_Multithreading.md` — write them from a blank file a few times rather than reading them.

## Running log

Freeform — dated entries on what clicked, what's still fuzzy, what came up in a real interview that isn't covered yet.

-
