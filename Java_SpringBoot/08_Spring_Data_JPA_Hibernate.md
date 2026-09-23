# Spring Data JPA & Hibernate

## JPA vs Hibernate — get this distinction right

- **JPA** (Jakarta Persistence API) is a *specification* — interfaces and annotations (`@Entity`, `EntityManager`, JPQL). **Hibernate** is the most common *implementation* of that spec (EclipseLink is another). Spring Data JPA sits on top of both — it's a Spring abstraction that generates repository implementations backed by whichever JPA provider you configure (almost always Hibernate).
- So: your code depends on JPA interfaces (portable across providers, in theory); Hibernate does the actual SQL generation, caching, dirty checking underneath.

## Entity lifecycle (states)

- **Transient** — plain `new` object, not associated with a persistence context, no DB row.
- **Managed/Persistent** — attached to a persistence context (`EntityManager`), changes are tracked (dirty checking) and flushed to DB automatically at transaction commit/flush.
- **Detached** — was managed, but the persistence context closed (e.g., after the transaction/request ended) — object still has data but changes to it are no longer tracked.
- **Removed** — marked for deletion, removed from DB on flush.
- `save()`/`persist()` moves transient → managed. `merge()` reattaches a detached entity's state onto a managed one (returns a *new* managed instance — a very common gotcha: `merge()` does not modify the entity you passed in).

## Lazy vs eager loading

- `@ManyToOne`/`@OneToOne` default to **EAGER**; `@OneToMany`/`@ManyToMany` default to **LAZY**. (Worth memorizing — it's asked constantly and the defaults are easy to get backwards.)
- Lazy association access outside an open persistence context throws `LazyInitializationException` — the classic "works in the service layer, breaks when the controller/serializer touches it after the transaction closed" bug. Fixes: fetch what you need within the transactional boundary (e.g. `JOIN FETCH` in a query, or an entity graph), map to a DTO before the transaction ends, or (last resort, has its own tradeoffs) `OpenSessionInView`.
- Prefer explicit fetch joins/DTO projections over blanket EAGER — EAGER everywhere silently pulls in huge object graphs and is a common performance foot-gun.

## The N+1 problem (very commonly asked — be ready to explain *and* fix)

- Symptom: you fetch a list of N parent entities, then access a lazy child collection/association on each — Hibernate issues 1 query for the parents + N additional queries (one per parent) for the children, instead of one combined query.
- Fixes: `JOIN FETCH` in JPQL (`SELECT o FROM Order o JOIN FETCH o.items WHERE ...`), `@EntityGraph` on the repository method, or batch fetching (`@BatchSize` / `hibernate.default_batch_fetch_size`, which turns N queries into `N/batch_size` `IN (...)` queries — a middle ground when a full join would multiply rows too much).
- Being able to say "I'd check the query log / enable `spring.jpa.show-sql` and `logging.level.org.hibernate.SQL=DEBUG`, spot the repeated pattern, then add a fetch join or entity graph" is the kind of concrete answer interviewers want here.

## @Transactional

- Applied at the service layer (not the repository layer — repository methods are already transactional per-call by default via Spring Data). Backed by AOP proxying (see Spring Core notes) — same self-invocation caveat applies.
- `readOnly = true` — hint to the provider to skip dirty checking / optimize for read-only queries; doesn't enforce immutability itself.
- Rollback rules: by default, Spring rolls back on unchecked exceptions (`RuntimeException`, `Error`) and **not** on checked exceptions unless you specify `rollbackFor = SomeCheckedException.class`. This trips people up constantly.
- **Propagation** (also covered in Database notes): `REQUIRED` (default — join existing transaction or create one), `REQUIRES_NEW` (suspend current, start a fresh independent transaction), `NESTED` (savepoint within the current transaction), `SUPPORTS`, `MANDATORY`, `NEVER`.

## Caching

- **First-level cache** — the persistence context/session itself; automatic, scoped to a single `EntityManager`/session, not shared across transactions.
- **Second-level cache** — optional, shared across sessions (e.g., Ehcache, Caffeine as the provider), needs explicit configuration and cache regions per entity; useful for read-heavy, rarely-changing reference data, dangerous for frequently-updated data (staleness/consistency risk).
- Different from Spring's general `@Cacheable` (application-level caching abstraction, unrelated to Hibernate's internal caches, often used for the same purpose at the service layer instead).

## Query methods

- Derived query methods from method name: `findByEmailAndStatus(String email, Status status)` — Spring Data parses the method name into a query.
- `@Query` for JPQL or native SQL when derivation gets unwieldy or you need something the naming convention can't express.
- Projections — return a DTO/interface instead of the full entity when you only need a few fields, avoiding overfetching.
- Pagination/sorting via `Pageable`/`Sort` parameters — `Page<T> findByStatus(Status status, Pageable pageable)`.

## Optimistic vs pessimistic locking (also see Database notes)

- Optimistic: `@Version` field on the entity — Hibernate checks the version on update, throws `OptimisticLockException` if it changed since read (someone else updated it first). Good default for low-contention scenarios; maps naturally to a 409 Conflict at the API layer.
- Pessimistic: `@Lock(LockModeType.PESSIMISTIC_WRITE)` — takes a DB-level row lock (`SELECT ... FOR UPDATE`) for the duration of the transaction. Use for high-contention critical sections where retry-on-conflict isn't acceptable.

## Commonly asked

- What's the relationship between JPA, Hibernate, and Spring Data JPA?
- Explain the N+1 problem with a concrete example, and name two ways to fix it.
- Why does accessing a lazy-loaded collection outside a transaction throw `LazyInitializationException`, and how would you avoid it?
- What's the default rollback behavior of `@Transactional`, and why doesn't it roll back on checked exceptions by default?
- Difference between `persist()`, `merge()`, and `save()` (Spring Data's).
- Optimistic vs pessimistic locking — when would you pick each?
