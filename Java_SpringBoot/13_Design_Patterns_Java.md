# Design Patterns (with Spring examples)

Interviewers usually don't want a textbook GoF recitation — they want you to recognize a pattern in real code and explain *why* it's the right shape. Spring itself is a great source of concrete examples, since you'll have used most of these without necessarily naming them.

## Singleton

- Ensure exactly one instance of a class exists. Classic Java implementation: private constructor + static instance, lazily initialized via the initialization-on-demand holder idiom (thread-safe without synchronization overhead) or `enum` singleton (also serialization-safe for free).
- **Spring example**: every `singleton`-scoped bean (the default scope) *is* this pattern — but the container manages it for you, so you rarely hand-roll a classic Singleton in Spring code. Good interview line: "In a Spring app I basically never write a manual singleton — I let the container manage scope."

## Factory Method / Abstract Factory

- Factory Method: defer object creation to a subclass/method rather than calling `new` directly, so the creation logic can vary.
- **Spring example**: `BeanFactory`/`ApplicationContext` itself, and `@Bean` factory methods in `@Configuration` classes — you're not calling `new SomeService()` yourself, you're declaring how to produce it and letting the container call the factory method.

## Builder

- Construct a complex object step by step, useful when a class has many optional parameters (avoids "telescoping constructors" with 6 overloaded signatures).
- **Java example**: `StringBuilder`, `Stream.builder()`. Lombok's `@Builder` generates this boilerplate automatically — worth mentioning since it's ubiquitous in real Spring codebases.
- Immutable objects (DTOs, value objects) are a natural fit — build up state via chained `.with...()` calls, then produce a final immutable instance.

## Strategy

- Define a family of interchangeable algorithms behind a common interface, select the implementation at runtime.
- **Spring example**: `PasswordEncoder` — `BCryptPasswordEncoder` vs others, injected wherever the interface is needed, swappable without touching calling code. Also: Spring's `Comparator` injection patterns, or multiple `@Service` implementations of the same interface selected via `@Qualifier`/`@Primary`.

## Observer

- Objects (observers) subscribe to state changes/events on a subject, get notified when something happens — decouples the event source from the reactors.
- **Spring example**: `ApplicationEventPublisher` / `@EventListener` — publish an `ApplicationEvent`, any number of `@EventListener`-annotated methods react without the publisher knowing who's listening. Common real use: publish an event after an order is placed, and have separate listeners send a confirmation email and update analytics, without coupling the order service to either.

## Proxy

- Provide a stand-in for another object that controls access to it (adds behavior before/after delegating to the real object).
- **Spring example**: this is *how AOP works* — `@Transactional`, `@Cacheable`, `@Async`, and Spring AOP aspects are all implemented via JDK dynamic proxies or CGLIB proxies wrapping the real bean (see Spring Core notes for the self-invocation gotcha this causes).

## Template Method

- Define the skeleton of an algorithm in a base class, let subclasses override specific steps without changing the overall structure.
- **Spring example**: `JdbcTemplate`, `RestTemplate`, `TransactionTemplate` — the "Template" naming in Spring is a direct nod to this pattern: Spring handles the boilerplate (connection acquisition, exception translation, resource cleanup), you supply the varying part (a callback/lambda).

## Decorator

- Attach additional responsibilities to an object dynamically, without modifying its class, by wrapping it in another object implementing the same interface.
- **Java example**: `java.io` streams — `new BufferedReader(new InputStreamReader(...))`, each layer adds behavior while implementing the same `Reader`/`InputStream` contract.

## Dependency Injection (as a pattern in its own right)

- Worth naming explicitly even though it's covered in depth in the Spring Core notes: DI is itself a design pattern (a specific form of Inversion of Control) — the object doesn't look up or construct its dependencies, they're supplied from outside. Good to connect this back to *why* it aids testability (swap real dependencies for mocks without changing the class).

## Commonly asked

- Name three design patterns you can point to directly in the Spring framework's own implementation.
- Why does Spring's `@Transactional`/AOP mechanism rely on the Proxy pattern, and what limitation does that introduce?
- When would you reach for Strategy vs just an `if/else` chain? What's the actual maintainability benefit?
- What problem does the Builder pattern solve that a large constructor doesn't?
- How is `JdbcTemplate` an example of the Template Method pattern?
- Where have you actually used the Observer pattern (Spring events or otherwise) in a project?
