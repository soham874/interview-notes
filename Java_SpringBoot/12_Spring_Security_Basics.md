# Spring Security Basics

## Authentication vs authorization — say this distinction explicitly if asked

- **Authentication** — who are you? (verifying identity — credentials, token validity).
- **Authorization** — what are you allowed to do? (permissions/roles, evaluated after identity is established).
- Interview-relevant because it maps directly to HTTP status codes: failed authentication → 401, failed authorization → 403 (see REST notes).

## The filter chain

- Spring Security is implemented as a chain of Servlet `Filter`s sitting in front of `DispatcherServlet` — each filter handles one concern (CSRF check, authentication, exception translation, authorization decision) and passes the request along.
- Key filters to be able to name: `UsernamePasswordAuthenticationFilter` (form login), `BasicAuthenticationFilter`, a JWT filter (custom, for token-based auth — not built in by default, teams typically write their own `OncePerRequestFilter`), `ExceptionTranslationFilter` (converts security exceptions into 401/403 responses), `FilterSecurityInterceptor`/`AuthorizationFilter` (the actual access-decision point, usually last).
- `SecurityContextHolder` holds the current `Authentication` object (principal, credentials, granted authorities) for the duration of the request — typically backed by a `ThreadLocal`, which is why authentication info doesn't automatically propagate to a new thread you spawn (async work, `@Async` methods) without explicitly passing it along.

## Modern configuration style (Spring Security 5.7+/6.x)

- `WebSecurityConfigurerAdapter` is deprecated/removed — configuration is now a `SecurityFilterChain` `@Bean` using the lambda DSL:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable()) // typical for stateless token-based APIs
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
    return http.build();
}
```

## Password storage

- Never store plaintext or use fast general-purpose hashes (MD5/SHA-1/SHA-256 alone) for passwords — use a slow, salted algorithm designed for this: `BCryptPasswordEncoder` is the Spring Security default and standard answer; Argon2/PBKDF2 also acceptable to mention.
- `PasswordEncoder.matches(raw, encoded)` for verification — never decode-and-compare (one-way hash, by design).

## JWT (JSON Web Tokens) — very commonly asked for stateless APIs

- Structure: header.payload.signature (base64url-encoded, **not encrypted** — anyone can decode and read the payload, so never put secrets in it; the signature only proves integrity/authenticity, not confidentiality).
- Stateless auth flow: client authenticates once (credentials → server issues a signed JWT), client sends the JWT on subsequent requests (typically `Authorization: Bearer <token>`), server verifies the signature and reads claims — no server-side session lookup needed, which is what makes it scale horizontally without sticky sessions or a shared session store.
- Expiry (`exp` claim) is essential since JWTs can't be "un-issued" the way a server-side session can be instantly invalidated. Revocation before expiry needs an extra mechanism (short expiry + refresh tokens, or a server-side denylist — which reintroduces some statefulness).
- Access token (short-lived) + refresh token (longer-lived, used to obtain new access tokens) is the standard pattern — good to mention if asked how you'd handle token expiry gracefully.

## OAuth2 basics (enough to hold a conversation, not implement a provider)

- OAuth2 is about **delegated authorization** ("let app X access my data on service Y without giving X my password"), not authentication per se — **OpenID Connect (OIDC)** is the identity layer built on top of OAuth2 that actually standardizes authentication/login.
- Roles: Resource Owner (the user), Client (the app requesting access), Authorization Server (issues tokens, e.g. Okta/Auth0/Keycloak/Cognito), Resource Server (the API being protected, validates the token).
- Authorization Code flow (with PKCE) is the standard for user-facing apps today; Client Credentials flow is for service-to-service (no user involved).
- In Spring: `spring-boot-starter-oauth2-resource-server` for a service that *validates* incoming tokens (most common role for a backend API); `spring-boot-starter-oauth2-client` for a service acting as the OAuth2 *client* itself.

## CORS vs CSRF — another pair that's easy to mix up

- **CORS** (Cross-Origin Resource Sharing) — a *browser* mechanism controlling which origins are allowed to call your API from client-side JavaScript; it's relaxing a browser restriction, not a security feature of your server per se. Configured via `CorsConfigurationSource`/`@CrossOrigin`.
- **CSRF** (Cross-Site Request Forgery) — an attack where a malicious site tricks a logged-in user's browser into making a state-changing request to your app, riding on the user's existing cookies/session. Relevant mainly for **cookie/session-based** auth; typically disabled for stateless token-based APIs (no ambient session cookie to hijack) — but if you do use cookies for auth in an API, CSRF protection still matters.

## Commonly asked

- Difference between authentication and authorization, and how each maps to an HTTP status code.
- Walk through what Spring Security's filter chain does, at a high level, for an incoming authenticated request.
- Why is a JWT not encrypted, and what does the signature actually guarantee?
- How would you handle token revocation given that JWTs are stateless by design?
- What's the actual difference between CORS and CSRF? Why might you disable CSRF for a REST API but keep CORS configured?
- OAuth2 vs OIDC — what problem does OIDC solve that plain OAuth2 doesn't?
