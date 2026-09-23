# Spring MVC & REST API Design

## Request flow (be able to sequence this)

`DispatcherServlet` (front controller) receives the request → consults `HandlerMapping` to find the right controller method → `HandlerAdapter` invokes it → controller returns a value (or throws) → for `@RestController`, `HttpMessageConverter` (Jackson by default) serializes the return value to JSON directly into the response body → for `@Controller` + view name, `ViewResolver` resolves a view to render instead.

- `@Controller` returns a view name (traditional MVC, server-rendered pages). `@RestController` = `@Controller` + `@ResponseBody` on every method — return value is written directly to the response body, which is what almost all modern API work uses.

## Key annotations

| Annotation | Purpose |
|---|---|
| `@RequestMapping` / `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` / `@PatchMapping` | Route methods to HTTP method + path |
| `@PathVariable` | Bind a URI template variable |
| `@RequestParam` | Bind a query parameter (supports `required`, `defaultValue`) |
| `@RequestBody` | Deserialize the request body into an object (via Jackson) |
| `@ResponseStatus` | Set the HTTP status returned by a handler or exception |
| `@Valid` / `@Validated` | Trigger bean validation on `@RequestBody`/`@ModelAttribute` |
| `@RestControllerAdvice` | Global exception handling + response body support, across controllers |

## REST design principles (be ready to discuss, not just define)

- Resources as nouns in URLs (`/orders/{id}`), HTTP methods carry the verb — `GET` (safe, idempotent), `POST` (not idempotent, creates), `PUT` (idempotent, full replace), `PATCH` (partial update, not necessarily idempotent), `DELETE` (idempotent).
- **Idempotent** means calling it once vs. N times leaves the system in the same state — not "returns the same response." Repeated `DELETE /orders/5` is idempotent even though the second call returns 404 instead of 204, because the *resulting state* is identical.
- Statelessness — no server-side session state between requests (auth via token, not server session) — this is what lets REST APIs scale horizontally without sticky sessions.
- HATEOAS is often mentioned as "the REST maturity model's top level" (Richardson Maturity Model) but rarely implemented in practice at most companies — good to know the term exists, not worth over-investing in.

## HTTP status codes — the ones you must know cold

| Code | Meaning | When |
|---|---|---|
| 200 OK | Success | GET, successful PUT/PATCH |
| 201 Created | Resource created | Successful POST, include `Location` header |
| 204 No Content | Success, no body | Successful DELETE, or PUT with nothing to return |
| 400 Bad Request | Client sent malformed/invalid data | Validation failure |
| 401 Unauthorized | Not authenticated | Missing/invalid credentials |
| 403 Forbidden | Authenticated but not allowed | Authz failure |
| 404 Not Found | Resource doesn't exist | |
| 409 Conflict | State conflict | Duplicate resource, optimistic locking version mismatch |
| 422 Unprocessable Entity | Semantically invalid | Sometimes used instead of 400 for validation — teams vary |
| 500 Internal Server Error | Unhandled server-side failure | |
| 503 Service Unavailable | Server temporarily can't handle request | Overload, downstream dependency down |

Interview trap: **401 vs 403** — 401 means "I don't know who you are" (or your credentials are invalid), 403 means "I know who you are, you're just not allowed." A lot of engineers get this backwards.

## Centralized exception handling

- `@RestControllerAdvice` + `@ExceptionHandler(SomeException.class)` — maps exceptions thrown from any controller to a consistent error response, instead of scattering try/catch across every handler method.
- Good practice: define a consistent error response shape (timestamp, status, error code/message, path, maybe a trace id) and return it from every handler, including a catch-all `@ExceptionHandler(Exception.class)` for unexpected failures — this also keeps you from leaking stack traces to clients.
- Since Spring 6/Boot 3: `ProblemDetail` (RFC 7807) is a built-in standard shape for this — worth mentioning if asked about "modern" error handling.

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(HttpStatus.NOT_FOUND.value(), ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return new ErrorResponse(HttpStatus.BAD_REQUEST.value(), message);
    }
}
```

## Validation

- Bean Validation (JSR 380 / Jakarta Bean Validation) annotations on DTO fields: `@NotNull`, `@NotBlank`, `@Size`, `@Min`/`@Max`, `@Email`, `@Pattern`. Trigger with `@Valid` on the `@RequestBody` param; failures throw `MethodArgumentNotValidException`, handled as above.
- Validate at the DTO/API boundary, not the entity — keeps persistence concerns separate from API contract concerns (also avoids exposing entity internals directly over the wire — map to/from DTOs).

## Versioning strategies

- URI versioning (`/v1/orders`) — most common, most visible, easiest to route/cache differently per version.
- Header versioning (`Accept: application/vnd.company.v1+json` or a custom header) — cleaner URLs, less discoverable, harder to test casually (e.g. in a browser).
- Query param versioning (`/orders?version=1`) — least common, easy to forget.
No universally "correct" answer — be ready to discuss the tradeoff (discoverability/simplicity vs. purity) rather than name one as objectively best.

## Commonly asked

- Walk through what happens from an HTTP request hitting `DispatcherServlet` to a JSON response going out.
- What makes an HTTP method idempotent, and is `POST` ever idempotent in practice (e.g., with an idempotency key)?
- 401 vs 403 — explain the difference with an example of each.
- How do you handle validation errors and unexpected exceptions consistently across a whole API?
- Why validate the DTO instead of the entity directly?
- What are the tradeoffs between URI-based and header-based API versioning?
