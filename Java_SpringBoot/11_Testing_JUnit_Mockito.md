# Testing — JUnit, Mockito, Spring Test

## Test pyramid framing

Unit tests (many, fast, isolated) → integration tests (fewer, test real wiring between components) → end-to-end tests (fewest, slowest, full system). A good answer to "how do you test a Spring Boot service" walks through this shape rather than jumping straight to `@SpringBootTest` for everything — over-relying on full-context integration tests makes the suite slow and brittle.

## JUnit 5 basics

- `@Test`, `@BeforeEach`/`@AfterEach` (per test), `@BeforeAll`/`@AfterAll` (once per class, must be static unless `@TestInstance(PER_CLASS)`), `@DisplayName`, `@Disabled`.
- `@ParameterizedTest` + `@ValueSource`/`@CsvSource`/`@MethodSource` — run the same test logic across multiple inputs instead of copy-pasting test methods.
- Assertions: JUnit's built-in `assertEquals`/`assertThrows`/`assertAll`, or AssertJ (`assertThat(x).isEqualTo(y)`) for more fluent/readable chains — AssertJ is common in Spring shops and worth naming.

## Mockito basics

- `@Mock` — creates a mock with all methods stubbed to return defaults (null/0/false) unless configured. `@InjectMocks` — creates a real instance of the class under test and injects the `@Mock`s into it (via constructor if possible, else setter/field).
- `when(mock.method(args)).thenReturn(value)` to stub; `verify(mock).method(args)` to assert an interaction happened (and `verify(mock, times(n))`, `never()`, etc.).
- `@Spy` — wraps a *real* object, so unstubbed methods call through to real logic instead of returning defaults; use sparingly, usually a sign the design could be cleaner.
- `ArgumentCaptor` — capture the actual argument passed to a mocked call, to assert on its contents rather than just that *a* call happened.
- Common mistake to be able to name: mocking types you don't own everywhere ("mocking the world") makes tests brittle and coupled to implementation details rather than behavior — prefer testing through real collaborators where feasible and mocking only true external boundaries (DB, HTTP clients, clocks).

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private OrderRepository orderRepository;
    @InjectMocks private OrderService orderService;

    @Test
    void throwsWhenOrderNotFound() {
        when(orderRepository.findById(1L)).thenReturn(Optional.empty());

        assertThrows(OrderNotFoundException.class, () -> orderService.getOrder(1L));
    }
}
```

## Spring test slices — know the difference, this is asked a lot

| Annotation | Loads | Use for |
|---|---|---|
| `@SpringBootTest` | Full application context (optionally with a real/random web server) | True integration tests, end-to-end wiring checks — slowest |
| `@WebMvcTest(SomeController.class)` | Only the web layer (controller, `@ControllerAdvice`, Jackson config) — services/repos must be mocked (`@MockBean`) | Testing controller logic, request mapping, validation, serialization in isolation |
| `@DataJpaTest` | Only JPA-related components — by default backed by an in-memory/embedded DB and transactional (rolled back after each test) | Testing repository queries |
| `@JsonTest` | Just Jackson serialization config | Testing custom serializers/deserializers |
| Plain unit test (no Spring annotations at all) | Nothing — just your class + Mockito | Service/business logic — should be the majority of your tests |

- `@MockBean` (or `@MockitoBean` in newer Spring Boot versions) replaces a bean in the Spring context with a Mockito mock — used inside slice tests to isolate the layer under test from its collaborators.
- Loading `@SpringBootTest` for every test class is a common anti-pattern that makes suites slow (full context startup each time, or per unique context configuration) — reach for it deliberately, not by default.

## Testcontainers

- Spins up real dependencies (Postgres, Kafka, Redis, ...) in Docker containers for integration tests, instead of relying on in-memory fakes (like H2 standing in for Postgres) that can silently diverge in behavior/SQL dialect from production.
- Common pairing: `@SpringBootTest` + Testcontainers' `@Container` + `@DynamicPropertySource` to point the app's datasource at the containerized DB — gives you confidence closer to production without a shared/flaky external test environment.
- Worth mentioning as the modern answer to "how do you test against a real database without an in-memory substitute."

## What "good" unit tests look like (useful framing if asked more open-endedly)

- Test behavior/contracts, not implementation details — a refactor that doesn't change behavior shouldn't break the test.
- One logical assertion focus per test, clear naming (`shouldThrowWhenX`, `returnsYWhenZ`), arrange-act-assert structure.
- Fast and deterministic — no sleeps, no reliance on real wall-clock time (inject a `Clock` instead of calling `Instant.now()` directly, for testability) or real network calls in unit tests.

## Commonly asked

- Difference between `@Mock` and `@Spy`, and when would you actually want a spy?
- What's the difference between `@WebMvcTest`, `@DataJpaTest`, and `@SpringBootTest`, and when would you use each?
- Why is loading the full `@SpringBootTest` context for every test considered an anti-pattern?
- How would you test a repository method that relies on a custom JPQL query?
- What problem does Testcontainers solve that an in-memory H2 database doesn't?
- How do you make a test involving `Instant.now()` or randomness deterministic?
