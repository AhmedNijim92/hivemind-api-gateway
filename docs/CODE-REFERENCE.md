# API Gateway — Code-Level Reference

## GatewayApplication

**Package:** `com.hivemind.gateway`

**Annotations:**
- `@SpringBootApplication` — Enables auto-configuration, component scanning, and configuration properties
- `@EnableDiscoveryClient` — Registers with Eureka service registry for service discovery and route resolution

**Design Pattern:** Application Entry Point (Spring Boot convention)

**Note:** This is a Spring Cloud Gateway (reactive/WebFlux-based) application — NOT a traditional Spring MVC application.

### Methods

#### `main(String[] args)`
- **Signature:** `public static void main(String[] args)`
- **Logic:** `SpringApplication.run(GatewayApplication.class, args)`
- **Returns:** void

---

## RateLimiterConfig

**Package:** `com.hivemind.gateway.config`

**Annotations:**
- `@Configuration`

**Design Pattern:** Strategy — defines the key resolution strategy for rate limiting

### Beans

#### `ipKeyResolver()`
- **Signature:** `@Bean public KeyResolver ipKeyResolver()`
- **Logic:**
  1. Returns a `KeyResolver` lambda that extracts the client IP for rate limiting
  2. First checks for `X-Forwarded-For` header (set by load balancers/proxies)
  3. If `X-Forwarded-For` exists and contains commas (multiple proxies): extracts the first IP (original client)
  4. If `X-Forwarded-For` exists without commas: uses the header value directly
  5. If no `X-Forwarded-For`: falls back to `request.getRemoteAddress()` (direct connection IP)
  6. If remote address is also null: returns `"anonymous"` as fallback key
- **Returns:** `Mono<String>` — the rate limit key (client IP address)

**Rate Limiting Integration:** This resolver is used by Spring Cloud Gateway's `RequestRateLimiter` filter (configured in `application.yml`) backed by Redis. Each IP gets its own rate limit bucket.

```
IP Resolution Priority:
1. X-Forwarded-For header (first IP if comma-separated)
2. RemoteAddress from ServerWebExchange
3. "anonymous" (fallback)
```

---

## JwtAuthenticationFilter

**Package:** `com.hivemind.gateway.filter`

**Annotations:**
- `@Component`

**Extends:** `AbstractGatewayFilterFactory<JwtAuthenticationFilter.Config>`

**Design Patterns:**
- Factory Method (AbstractGatewayFilterFactory pattern)
- Chain of Responsibility (gateway filter chain)
- Template Method (Config-based filter creation)

### Fields

| Field | Type | Source |
|-------|------|--------|
| jwtSecret | String | `@Value("${jwt.secret}")` |

### Constructor

```java
public JwtAuthenticationFilter() {
    super(Config.class);
}
```
Calls parent constructor with the Config class — required by Spring Cloud Gateway's filter factory pattern.

### Methods

#### `apply(Config config)`
- **Signature:** `@Override public GatewayFilter apply(Config config)`
- **Logic:** Returns a `GatewayFilter` lambda `(exchange, chain) -> { ... }`:
  1. **Header Check:** Gets `Authorization` header from the request
  2. **Bearer Prefix Check:** Validates header exists and starts with `"Bearer "`
     - If missing or invalid → calls `onError(exchange, HttpStatus.UNAUTHORIZED)` and short-circuits
  3. **Token Extraction:** Extracts the JWT string (after "Bearer ")
  4. **Validation:** Calls `validateToken(token)`
     - If invalid → calls `onError(exchange, HttpStatus.UNAUTHORIZED)` and short-circuits
  5. **Claims Extraction:** Parses the token to extract:
     - `subject` (userId) from JWT subject claim
     - `role` from JWT custom claim
  6. **Request Mutation:** Mutates the downstream request to add headers:
     - `X-User-Id` = extracted userId
     - `X-User-Role` = extracted role
  7. **Chain Continuation:** Calls `chain.filter(mutatedExchange)` to pass to next filter/route
- **Returns:** `GatewayFilter`

#### `validateToken(String token)` (Private)
- **Signature:** `private boolean validateToken(String token)`
- **Logic:**
  1. Creates signing key: `Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8))`
  2. Parses and validates: `Jwts.parser().verifyWith(key).build().parseSignedClaims(token)`
  3. Returns `true` if parsing succeeds
  4. Returns `false` if any exception (expired, malformed, invalid signature, etc.)
- **Returns:** `boolean`

#### `onError(ServerWebExchange exchange, HttpStatus httpStatus)` (Private)
- **Signature:** `private Mono<Void> onError(ServerWebExchange exchange, HttpStatus httpStatus)`
- **Logic:**
  1. Sets the response status code to the given `httpStatus` (typically 401)
  2. Calls `exchange.getResponse().setComplete()` to short-circuit the filter chain
- **Returns:** `Mono<Void>` — completes the response without forwarding to downstream service

### Inner Class: Config

```java
public static class Config {
    // Empty — required by AbstractGatewayFilterFactory contract
}
```

**Purpose:** Spring Cloud Gateway's filter factory pattern requires a Config class. Even when no configuration is needed, the class must exist. This allows the filter to be referenced in `application.yml` routes:

```yaml
filters:
  - JwtAuthenticationFilter
```

---

## RequestIdFilter

**Package:** `com.hivemind.gateway.filter`

**Annotations:**
- `@Component`

**Implements:** `GlobalFilter`, `Ordered`

**Design Patterns:**
- Chain of Responsibility (global filter chain)
- Decorator (enriches request/response with correlation ID)

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| REQUEST_ID_HEADER | `"X-Request-Id"` | Header name for request correlation |

### Methods

#### `filter(ServerWebExchange exchange, GatewayFilterChain chain)`
- **Signature:** `@Override public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)`
- **Logic:**
  1. Checks if `X-Request-Id` header already exists on the incoming request
  2. If not present: generates a new UUID as the request ID
  3. Mutates the request to include the `X-Request-Id` header (for downstream services)
  4. Also adds `X-Request-Id` to the response headers (for client-side correlation)
  5. Calls `chain.filter(mutatedExchange)` to continue the filter chain
- **Returns:** `Mono<Void>`
- **Purpose:** Enables distributed tracing and log correlation across all microservices

#### `getOrder()`
- **Signature:** `@Override public int getOrder()`
- **Logic:** Returns `-1`
- **Returns:** `int` — filter order priority
- **Purpose:** Order `-1` ensures this filter runs before most other filters (including JwtAuthenticationFilter). Every request gets a correlation ID before any authentication or routing logic.

---

## Gateway Route Configuration (application.yml)

While not a Java class, the gateway routing is configured declaratively and is critical to understanding the system:

### Route Pattern

```yaml
spring.cloud.gateway.routes:
  - id: auth-service
    uri: lb://AUTH-SERVICE        # Load-balanced via Eureka
    predicates:
      - Path=/api/v1/auth/**
    filters:
      - StripPrefix=0

  - id: user-service
    uri: lb://USER-SERVICE
    predicates:
      - Path=/api/v1/users/**
    filters:
      - JwtAuthenticationFilter   # Requires valid JWT
      - StripPrefix=0

  - id: group-service
    uri: lb://GROUP-SERVICE
    predicates:
      - Path=/api/v1/groups/**
    filters:
      - JwtAuthenticationFilter
      - StripPrefix=0

  - id: post-service
    uri: lb://POST-SERVICE
    predicates:
      - Path=/api/v1/posts/**
    filters:
      - JwtAuthenticationFilter
      - StripPrefix=0

  - id: meeting-service
    uri: lb://MEETING-SERVICE
    predicates:
      - Path=/api/v1/meetings/**
    filters:
      - JwtAuthenticationFilter
      - StripPrefix=0

  - id: notification-service
    uri: lb://NOTIFICATION-SERVICE
    predicates:
      - Path=/api/v1/notifications/**
    filters:
      - JwtAuthenticationFilter
      - StripPrefix=0

  - id: media-service
    uri: lb://MEDIA-SERVICE
    predicates:
      - Path=/api/v1/media/**
    filters:
      - JwtAuthenticationFilter
      - StripPrefix=0
```

### Key Points:
- **Auth service** routes do NOT have the JwtAuthenticationFilter (public endpoints)
- All other services require JWT authentication
- `lb://` prefix enables client-side load balancing via Eureka service discovery
- `StripPrefix=0` preserves the full path when forwarding to downstream services

---

## Architecture Overview

```
Client Request
    │
    ▼
┌─────────────────────────────┐
│    RequestIdFilter (-1)     │  ← Adds X-Request-Id
├─────────────────────────────┤
│  JwtAuthenticationFilter    │  ← Validates JWT, adds X-User-Id / X-User-Role
├─────────────────────────────┤
│  RequestRateLimiter         │  ← Rate limits by IP (Redis-backed)
├─────────────────────────────┤
│  Route Predicate Matching   │  ← Matches path to service
├─────────────────────────────┤
│  Load Balanced Forwarding   │  ← lb:// via Eureka discovery
└─────────────────────────────┘
    │
    ▼
Downstream Microservice
```

### Headers Injected by Gateway

| Header | Source | Description |
|--------|--------|-------------|
| `X-Request-Id` | RequestIdFilter | Correlation ID for distributed tracing |
| `X-User-Id` | JwtAuthenticationFilter | Authenticated user's UUID |
| `X-User-Role` | JwtAuthenticationFilter | User's role (USER/ADMIN/SUPER_ADMIN) |
