# Spring Boot Fundamentals

## What Spring Boot actually adds over plain Spring

Plain Spring requires you to wire up configuration (data sources, view resolvers, servlet containers) by hand. Spring Boot's value proposition is three things: **auto-configuration** (sensible defaults inferred from what's on the classpath), **starters** (curated dependency bundles), and **an embedded server** (no external Tomcat/Jetty install needed — the app is a runnable jar). This framing is a good way to open an answer if asked "what is Spring Boot / why use it."

## @SpringBootApplication — break it down

`@SpringBootApplication` is a meta-annotation combining three:

- `@SpringBootConfiguration` — itself a specialization of `@Configuration`, marks the class as a source of bean definitions.
- `@EnableAutoConfiguration` — the real magic: triggers Spring Boot's auto-configuration mechanism.
- `@ComponentScan` — scans the package of the annotated class (and subpackages) for `@Component`/`@Service`/`@Repository`/`@Controller` etc. This is *why* your main application class's package placement matters — beans outside that package tree won't be found without an explicit `@ComponentScan(basePackages=...)`.

## Auto-configuration mechanism

- Auto-configuration classes are conditionally activated using `@Conditional`-family annotations: `@ConditionalOnClass` (a library is on the classpath), `@ConditionalOnMissingBean` (only apply if the user hasn't defined their own), `@ConditionalOnProperty`, `@ConditionalOnWebApplication`, etc.
- Concretely: adding `spring-boot-starter-data-jpa` to the classpath causes `DataSourceAutoConfiguration`/`HibernateJpaAutoConfiguration` to kick in *only because* the relevant classes are present — remove the dependency and that auto-configuration silently doesn't apply.
- `@ConditionalOnMissingBean` is why defining your own `@Bean` of a given type (e.g., your own `ObjectMapper`) transparently overrides Boot's default — Boot's auto-configured bean backs off.
- Since Spring Boot 2.7+/3.x, auto-configuration classes are registered via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (older versions used `spring.factories`) — worth knowing the mechanism exists even if the exact file name has changed across versions.
- Debugging tool: run with `--debug` (or `debug=true` in properties) to get an auto-configuration report showing what was applied and why, and what was excluded and why not.

## Starters

- A starter (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-security`, ...) is just a Maven/Gradle dependency descriptor with no code of its own — it pulls in a coherent, version-aligned set of libraries, avoiding manual version juggling ("dependency hell").
- `spring-boot-starter-parent` (or the BOM `spring-boot-dependencies` if not using the parent POM) manages compatible versions across the whole Spring ecosystem.

## Embedded server

- Default is embedded Tomcat (via `spring-boot-starter-web`); swappable for Jetty or Undertow by excluding Tomcat and adding the alternative starter.
- The app becomes a self-contained "fat jar" (`java -jar app.jar`) — no separate app server deployment step, which is a big part of why Spring Boot fits containerized/microservice deployment so well.

### What Tomcat actually does, and where Spring begins

Tomcat is a **servlet container / web server**: it implements the Jakarta Servlet spec (plus Jakarta EL and WebSocket), owns the network socket, and manages the servlet lifecycle. Spring is not a web server — it's what runs *inside* one.

The inversion worth understanding: **pre-Boot**, you built a `.war` and deployed it *into* a standalone Tomcat installation you'd installed and configured separately. **With Boot**, Tomcat is a library embedded *inside* your jar, and your `main()` starts it. Same component, opposite direction of containment — this is the single biggest reason Boot apps fit container/Kubernetes deployment cleanly.

End-to-end request path, worth being able to narrate:

1. Tomcat listens on a port and accepts raw HTTP traffic, parsing it into a `HttpServletRequest`.
2. It hands the request to the registered servlet — for a Spring MVC app, that's the **`DispatcherServlet`**, the single entry point into the framework.
3. `DispatcherServlet` consults `HandlerMapping` to find the controller method matching the path/method, and `HandlerAdapter` invokes it.
4. The method returns an object; an `HttpMessageConverter` (Jackson) serializes it to JSON into the response body.
5. Tomcat writes the `HttpServletResponse` back over the socket.

Tomcat also owns the **thread pool**: each request is handled on a worker thread (`server.tomcat.threads.max`, default 200). This matters for the classic scaling question — a blocking downstream call holds a Tomcat worker thread for its whole duration, so 200 slow calls saturate the server regardless of CPU. That's the motivation for timeouts, bulkheads, reactive stacks (WebFlux), or virtual threads (`spring.threads.virtual.enabled=true` on Java 21+).

## Version landscape — know roughly where the boundaries are

Being able to place yourself in the version timeline signals you've worked with the framework recently rather than learning it from an old course.

**Spring Boot 2.x → 3.0** (Nov 2022) — the disruptive one:

- **Java 17 minimum** (2.x allowed Java 8).
- **`javax.*` → `jakarta.*`** namespace change across every EE API (servlets, JPA, validation). This is what made the migration painful — it touches imports in nearly every file.
- Built on Spring Framework 6, Hibernate 6, Spring Security 6.
- Proper GraalVM **native image** support via the AOT engine.
- Observability rework: Micrometer metrics + Spring Cloud Sleuth tracing consolidated into the **Micrometer Observation API** (Sleuth retired).
- Trailing-slash matching removed — `/api/stuff/` no longer matches a `/api/stuff` mapping.
- Auto-configuration registration moved from `spring.factories` to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (introduced in 2.7, mandatory in 3.0).
- Tooling: `spring-boot-properties-migrator` reports renamed/deprecated properties during startup.

**Spring Boot 3.x → 4.0** (Nov 2025), with **4.1** following in June 2026 — a much gentler migration:

- **Java 17 remains the minimum**; Java 21/25 recommended (virtual threads).
- **No namespace rewrite** — the jakarta migration is done. Jakarta EE 11 is fully adopted.
- The main breaking change is **modularization**: auto-configuration was split into smaller modules, so some features now need an explicit dependency that used to come transitively.
- Jackson 3 replaces Jackson 2; JUnit 4 support removed entirely (JUnit Jupiter only).
- New capabilities worth naming: first-class **API versioning** on `@RequestMapping`, declarative **HTTP interface clients** (`@HttpExchange`), **JSpecify null-safety** annotations, built-in **resilience annotations** (`@Retryable`, `@ConcurrencyLimit` via `@EnableResilientMethods` — reducing the need for Resilience4j for basic cases), and official OpenTelemetry integration.
- Spring Data AOT moves query generation to build time, cutting startup substantially.

If asked "which version have you used," answer honestly about what you've shipped, then show awareness of where the ecosystem currently is. Interviewers care more that you know 3.x's jakarta break exists than that you've personally run 4.1.

### The AOT engine (worth understanding, since it's the 3.x+ headline)

Traditionally Spring does its work at **runtime startup**: scan the classpath, evaluate `@Conditional` logic, reflectively instantiate beans, generate proxies dynamically. Flexible, but it costs startup time and memory, and it's hostile to GraalVM native compilation (which needs to know the whole reachable world at build time).

**AOT (ahead-of-time) processing** shifts that work to **build time**: Spring inspects your code during the build, evaluates which conditional beans actually apply, and generates static configuration classes plus reflection/resource hints. The runtime then does far less discovery.

Tradeoffs to state: much faster startup and lower memory (dramatic in native images — milliseconds instead of seconds), but longer build times, and genuinely dynamic bean registration becomes difficult because the bean graph is largely fixed at build time.

## Configuration & profiles

- `application.properties`/`application.yml` — property resolution order (highest wins) roughly: command-line args → `SPRING_APPLICATION_JSON` env var → `-D` JVM system properties → OS environment variables → profile-specific config file (`application-{profile}.yml`) → `application.yml` → `@PropertySource` → defaults. (Exact precedence list is long — knowing there *is* a well-defined precedence, and that env vars/command-line typically beat the packaged file, is usually enough.)
- Profiles (`application-dev.yml`, `application-prod.yml`) activated via `spring.profiles.active` — lets the same jar run with different config per environment without code changes.
- `@ConfigurationProperties` — type-safe binding of a whole config prefix to a POJO, preferred over scattering `@Value("${...}")` everywhere for related settings; supports validation via `@Validated`.
- `@Value("${some.prop:default}")` — single property injection with an optional default.

## Actuator

- `spring-boot-starter-actuator` exposes production-readiness endpoints: `/actuator/health`, `/actuator/metrics`, `/actuator/info`, `/actuator/env`, `/actuator/loggers` (change log levels at runtime without redeploy), `/actuator/beans` (inspect the whole context).
- Health indicators are composable/custom — you can implement `HealthIndicator` to report your own dependency's health (e.g., a downstream service or queue) and it rolls up into the overall `/health` status.
- In production, most actuator endpoints beyond health/info are typically secured or disabled by default exposure — a real interview follow-up is "how would you secure Actuator endpoints" (separate management port, Spring Security rules, or restrict `management.endpoints.web.exposure.include`).

## Common annotations quick-reference

| Annotation | Purpose |
|---|---|
| `@Component` / `@Service` / `@Repository` / `@Controller` | Stereotypes — functionally similar, but `@Repository` adds exception translation (persistence exceptions → Spring's `DataAccessException` hierarchy), `@Controller`/`@RestController` are Spring MVC-aware |
| `@Configuration` + `@Bean` | Java-based bean definition, alternative/complement to component scanning |
| `@Value` | Inject a single property |
| `@ConfigurationProperties` | Bind a config block to a POJO |
| `@Profile("dev")` | Only register this bean for a given active profile |
| `@Conditional...` | Fine-grained conditional bean registration |

## Commonly asked

- What does `@SpringBootApplication` actually expand to, and why does component scan package placement matter?
- How does Spring Boot decide which auto-configurations to apply, and how would you find out why a particular bean wasn't auto-configured?
- How would you override a Boot-provided default bean (e.g., a custom `ObjectMapper`) without disabling auto-configuration entirely?
- What's the property source precedence order, at least the parts that matter day-to-day (env vars vs. profile file vs. default file)?
- Why prefer `@ConfigurationProperties` over a pile of `@Value` fields for related settings?
- What does the Actuator health endpoint show by default, and how would you add a custom health check for a downstream dependency?
- What is Tomcat's role in a Spring Boot app, and how did that relationship change from the old WAR deployment model?
- Why does a slow downstream HTTP call limit your throughput even when CPU is idle?
- What was the hardest part of migrating from Spring Boot 2.x to 3.x, and why?
- What does the AOT engine do, and what do you give up by using it?
