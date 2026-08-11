# HiveMind API Gateway

> Central entry point for all HiveMind platform traffic — handles routing, JWT validation, rate limiting, and security headers.

## Overview

The API Gateway is built on Spring Cloud Gateway and serves as the single ingress for all client requests. It validates JWT tokens, enforces rate limits on authentication endpoints via Redis, injects user identity headers (`X-User-Id`, `X-User-Role`, `X-User-Name`) into downstream requests, and applies CORS policies. WebSocket routes are supported for real-time features.

## Features

- JWT token validation on protected routes
- Rate limiting on auth endpoints (Redis-backed)
- Forwards user identity headers to downstream services
- CORS configuration for frontend origins
- WebSocket route support (STOMP upgrade)
- Security headers injection (XSS, Content-Type sniffing, etc.)

## API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| * | `/api/v1/auth/**` | Public | Authentication routes (login, register, OTP) |
| * | `/api/v1/users/**` | JWT | User service routes |
| * | `/api/v1/groups/**` | JWT | Group service routes |
| * | `/api/v1/posts/**` | JWT | Post service routes |
| * | `/api/v1/meetings/**` | JWT | Meeting service routes |
| * | `/api/v1/chat/**` | JWT | Chat service routes |
| * | `/api/v1/notifications/**` | JWT | Notification service routes |
| * | `/api/v1/media/upload` | JWT | Media upload |
| GET | `/api/v1/media/{id}/download` | Public | Media download (images served without auth) |

## Configuration

| Property | Description | Default |
|----------|-------------|---------|
| `server.port` | Gateway listen port | `8080` |
| `jwt.secret` | JWT signing secret | — |
| `spring.data.redis.host` | Redis host for rate limiting | `localhost` |
| `spring.cloud.gateway.globalcors.corsConfigurations` | CORS allowed origins | — |
| `eureka.client.serviceUrl.defaultZone` | Eureka registry URL | `http://localhost:8761/eureka` |

## Tech Stack

- Java 17
- Spring Boot 3.x
- Spring Cloud Gateway
- Spring Security (JWT filter)
- Redis (rate limiting)
- Eureka Client (service discovery)
- Maven

## Docker

```
Port: 8080
Base image: eclipse-temurin:17-jre-alpine
JVM flags: -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC
User: non-root (spring)
```

## CI/CD

- **Build**: Maven `clean package` with JDK 17 (Temurin)
- **Test**: Unit tests run during build phase
- **Docker**: Build and push to Docker Hub on `main` branch merge
- **Security**: Trivy vulnerability scan (CRITICAL, HIGH) on built image
