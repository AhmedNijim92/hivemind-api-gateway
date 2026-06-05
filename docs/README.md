# API Gateway

> HiveMind Central API Gateway & Security Layer

## Overview

The api-gateway is the single entry point for all client requests. It routes traffic to backend microservices via service discovery (Eureka), validates JWT tokens, applies rate limiting, adds security headers, injects request tracing IDs, and manages CORS.

## Service Info

| Property | Value |
|----------|-------|
| Port | 8080 |
| Service Name | `api-gateway` |
| Framework | Spring Cloud Gateway (WebFlux) |
| Spring Boot | 3.3.5 |
| Spring Cloud | 2023.0.3 |
| Java | 17 |

## Architecture

```
Client Request
  │
  ▼
┌─────────────────────────────────────────┐
│            API Gateway (:8080)           │
│                                         │
│  1. RequestIdFilter (X-Request-Id)      │
│  2. CORS validation                     │
│  3. Rate limiting (Redis)               │
│  4. JwtAuthenticationFilter             │
│     → validates token                   │
│     → injects X-User-Id, X-User-Role   │
│  5. Security headers (response)         │
│  6. Route to service (Eureka lb://)     │
└─────────────────────────────────────────┘
  │
  ▼
Backend Microservices (via Eureka service discovery)
```

## Route Configuration

### Public Routes (no JWT required)

| Route ID | Path | Target | Rate Limit |
|----------|------|--------|------------|
| auth-service-public | `/api/v1/auth/sendOtp`, `/signin`, `/createUser` | lb://auth-service | 5/s burst 10 |
| media-download | `/api/v1/media/*/download` | lb://media-service | — |

### Protected Routes (JWT required)

| Route ID | Path | Target |
|----------|------|--------|
| auth-service-admin | `/api/v1/auth/createAdmin` | lb://auth-service |
| user-service | `/api/v1/users/**` | lb://user-service |
| group-service | `/api/v1/groups/**` | lb://group-service |
| post-service | `/api/v1/posts/**` | lb://post-service |
| meeting-service | `/api/v1/meetings/**` | lb://meeting-service |
| notification-service | `/api/v1/notifications/**` | lb://notification-service |
| media-service | `/api/v1/media/**` | lb://media-service |

## Filters

### JwtAuthenticationFilter

Custom `AbstractGatewayFilterFactory` that:
1. Checks for `Authorization: Bearer <token>` header
2. Validates the JWT signature using the shared secret
3. Extracts `sub` (userId) and `role` claims
4. Injects `X-User-Id` and `X-User-Role` headers into the downstream request
5. Returns 401 if token is missing or invalid

### RequestIdFilter

Global filter that:
1. Checks for existing `X-Request-Id` header
2. Generates a UUID if not present
3. Adds to both request (downstream) and response (client debugging)

### RateLimiterConfig

- Strategy: IP-based (uses `X-Forwarded-For` or remote address)
- Backend: Redis reactive
- Applied to: auth public routes only
- Settings: 5 requests/second replenish, 10 burst capacity

### Default Security Headers (all responses)

| Header | Value |
|--------|-------|
| X-Content-Type-Options | nosniff |
| X-Frame-Options | DENY |
| X-XSS-Protection | 1; mode=block |
| Referrer-Policy | strict-origin-when-cross-origin |
| Permissions-Policy | camera=(), microphone=(), geolocation=() |
| Cache-Control | no-store |
| Server | (removed) |

## CORS Configuration

| Setting | Value |
|---------|-------|
| Allowed Origins | `${CORS_ALLOWED_ORIGINS:http://localhost:3000}` |
| Allowed Methods | GET, POST, PUT, DELETE, OPTIONS, PATCH |
| Allowed Headers | * |
| Exposed Headers | Authorization, X-Request-Id |
| Allow Credentials | true |
| Max Age | 3600s |

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| JWT_SECRET | (base64 key) | HMAC-SHA256 signing key (must match auth-service) |
| CORS_ALLOWED_ORIGINS | http://localhost:3000 | Allowed CORS origins |
| EUREKA_SERVER | http://localhost:8761/eureka | Eureka registry URL |

### Service Discovery

The gateway uses Eureka-based load balancing (`lb://service-name`) to route requests. It also enables discovery locator for automatic route generation from registered services (lowercase service IDs).

## Dependencies

- spring-cloud-starter-gateway
- spring-boot-starter-data-redis-reactive
- spring-boot-starter-actuator
- spring-cloud-starter-netflix-eureka-client
- jjwt-api / jjwt-impl / jjwt-jackson (0.12.3)
- lombok

## Running Locally

```bash
# Prerequisites: Redis running (for rate limiter), Eureka running
cd microservices/api-gateway
mvn spring-boot:run
```

The gateway should be started last — after all backend services have registered with Eureka.

## Request Flow Example

```
1. Client → POST /api/v1/groups (with Authorization: Bearer <jwt>)
2. RequestIdFilter → adds X-Request-Id: uuid
3. JwtAuthenticationFilter → validates JWT
   → extracts userId, role
   → adds X-User-Id: <userId>, X-User-Role: USER
4. Route matcher → /api/v1/groups/** → lb://group-service
5. Eureka lookup → group-service instance at 192.168.1.5:8083
6. Forward request to group-service with enriched headers
7. Response → default filters add security headers
8. Response → client
```

## Notes

- Max request header size: 16KB
- Max in-memory codec: 50MB (for media upload proxy)
- WebFlux (reactive) — does not use Servlet/Tomcat
