# Spring Core — IoC, Dependency Injection, AOP

## Inversion of Control & the container

- IoC = the framework controls object creation/wiring instead of your code doing `new` everywhere. Spring's `ApplicationContext` (built on `BeanFactory`) is the container: it reads configuration (annotations, Java config, or XML historically), instantiates beans, wires their dependencies, manages their lifecycle.
- `BeanFactory` vs `ApplicationContext`: `ApplicationContext` is the superset (adds event publishing, internationalization, AOP integration, easier annotation config) — always what you use in practice; `BeanFactory` is the more primitive interface underneath.

## Dependency injection types

- **Constructor injection** (preferred) — dependencies passed via constructor, enables `final` fields, makes the class impossible to construct in an invalid state, plays well with immutability and testing (no reflection needed to inject mocks).
- **Setter injection** — for optional dependencies, allows reconfiguration after construction.
- **Field injection** (`@Autowired` directly on a field) — most concise but discouraged: hides dependencies from the constructor signature, makes the class harder to unit test without Spring, can't have `final` fields, and — the most cited reason in interviews — makes circular dependencies possible to compile when they should really be a design smell caught early.
- If asked "why constructor injection specifically": explicit dependencies, immutability, fails fast at construction time if a bean can't be found, and it's the only style that works cleanly if you ever construct the object outside Spring (e.g., in a plain unit test with `new`).

## Bean lifecycle (be able to sequence this)

1. Instantiate (constructor called).
2. Populate properties (dependency injection happens).
3. `Aware` interfaces invoked if implemented (`BeanNameAware`, `ApplicationContextAware`, etc.).
4. `BeanPostProcessor.postProcessBeforeInitialization()`.
5. `@PostConstruct` / `InitializingBean.afterPropertiesSet()` / custom `init-method`.
6. `BeanPostProcessor.postProcessAfterInitialization()` — this is where proxies (AOP, `@Transactional`, etc.) actually get wrapped around the bean.
7. Bean is ready for use.
8. On shutdown: `@PreDestroy` / `DisposableBean.destroy()` / custom `destroy-method`.

## Bean scopes

| Scope | Meaning |
|---|---|
| `singleton` (default) | One instance per Spring container |
| `prototype` | New instance every time it's requested |
| `request` | One instance per HTTP request (web-aware contexts) |
| `session` | One instance per HTTP session |
| `application` | One instance per `ServletContext` |

Gotcha to know: injecting a `prototype`-scoped bean into a `singleton`-scoped bean via normal field injection only resolves it *once*, at singleton creation — you'll keep getting the same prototype instance forever unless you use `ObjectProvider<T>`/`Provider<T>` or a scoped proxy to get a fresh one on each access.

## Circular dependencies

- Spring can resolve constructor-based circular dependencies... actually it **cannot** — constructor injection circular dependency fails at startup with `BeanCurrentlyInCreationException`. Setter/field injection circular dependencies *can* be resolved because Spring can instantiate the bean first (via the early bean reference cache/"third-level cache" mechanism) and inject dependencies afterward.
- The right fix is almost always to break the cycle via redesign (extract a shared dependency into a third bean, use `@Lazy`, or introduce an interface) rather than relying on setter injection to paper over it.

## AOP (Aspect-Oriented Programming)

- Purpose: cross-cutting concerns (logging, transactions, security, caching) that would otherwise be duplicated across many classes get factored into an "aspect" applied declaratively.
- Key vocabulary: **Aspect** (the module of cross-cutting logic), **Advice** (the action taken — `@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`), **Pointcut** (expression selecting which join points to apply advice to), **Join point** (a point during execution, e.g. a method call).
- `@Transactional`, `@Cacheable`, `@Async`, `@Retryable` are all implemented as Spring AOP under the hood.

## Proxies — how Spring AOP is actually implemented

- **JDK dynamic proxies** — used when the target class implements at least one interface; the proxy implements the same interface(s) and delegates through `InvocationHandler`.
- **CGLIB proxies** — used when the target class has no interface (or `proxyTargetClass=true` is forced); generates a runtime subclass that overrides methods to add advice.
- **This is why self-invocation breaks `@Transactional`/AOP**: if method A calls method B on `this` within the same class, that call bypasses the proxy entirely (it's a plain Java call, not going through the container's proxy), so B's `@Transactional`/`@Async`/etc. annotation is silently ignored. Fix: move B to another bean and inject it, or use `AopContext.currentProxy()` (needs `exposeProxy=true`), or restructure.
- Because of proxying, methods need to be `public` (JDK proxies can only proxy interface methods, which are implicitly public) and non-`final` classes/methods for CGLIB to subclass them.

## Commonly asked

- Why is constructor injection generally preferred over field injection?
- Walk through the Spring bean lifecycle from instantiation to destruction.
- Why does calling an `@Transactional` method from another method in the *same class* not trigger the transaction?
- Can Spring resolve a circular dependency between two beans using constructor injection? What about setter injection?
- What's the difference between a JDK dynamic proxy and a CGLIB proxy, and when does Spring choose each?
- What problem does injecting a prototype bean into a singleton bean create, and how do you fix it?
