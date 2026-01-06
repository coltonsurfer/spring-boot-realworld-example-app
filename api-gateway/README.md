# API Gateway

This is the API Gateway for the RealWorld application, implemented using Spring Cloud Gateway. It serves as the single entry point for all client requests and routes traffic to the backend services.

## Overview

The API Gateway is part of the microservices migration strategy (Phase 0). Currently, it acts as a pass-through proxy, forwarding all requests to the monolith application. As services are extracted in subsequent phases, the gateway will route requests to the appropriate microservices.

## Features

- Routes all REST API endpoints to the monolith
- Routes GraphQL endpoints (`/graphql`, `/graphiql`) to the monolith
- Preserves all headers including `Authorization` for JWT authentication
- Health check endpoints via Spring Actuator
- Debug logging for gateway routing decisions

## Prerequisites

- Java 11 or higher
- Gradle 7.x
- The monolith application running on port 8080

## Running the Gateway

### Option 1: Using Gradle Wrapper (from api-gateway directory)

```bash
cd api-gateway
../gradlew bootRun
```

### Option 2: Using Gradle directly

```bash
cd api-gateway
gradle bootRun
```

The gateway will start on port **8081**.

## Configuration

The gateway is configured in `src/main/resources/application.yml`. Key settings:

| Property | Default | Description |
|----------|---------|-------------|
| `server.port` | 8081 | Gateway listening port |
| `spring.cloud.gateway.routes[*].uri` | http://localhost:8080 | Backend service URL |

### Changing the Monolith URL

To point the gateway to a different monolith instance, update the `uri` in all routes in `application.yml`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: users-public
          uri: http://your-monolith-host:port
          # ...
```

## API Routes

The gateway defines explicit routes for all API endpoints:

| Route ID | Path Pattern | Description |
|----------|--------------|-------------|
| users-public | `/users`, `/users/login` | User registration and login |
| current-user | `/user` | Current user operations |
| articles-feed | `/articles/feed` | User's article feed |
| article-favorites | `/articles/{slug}/favorite` | Favorite/unfavorite articles |
| article-comments | `/articles/{slug}/comments` | Article comments |
| article-single | `/articles/{slug}` | Single article CRUD |
| articles | `/articles` | Article list and create |
| profiles | `/profiles/{username}` | User profiles and follow |
| tags | `/tags` | Tag list |
| graphql | `/graphql` | GraphQL API |
| graphiql | `/graphiql` | GraphQL IDE |
| fallback | `/**` | Catch-all for any other paths |

## Health Checks

The gateway exposes actuator endpoints:

- `GET /actuator/health` - Health status
- `GET /actuator/info` - Application info
- `GET /actuator/gateway/routes` - List all configured routes

## Testing

To verify the gateway is working:

1. Start the monolith on port 8080:
   ```bash
   ./gradlew bootRun
   ```

2. Start the gateway on port 8081:
   ```bash
   cd api-gateway && ../gradlew bootRun
   ```

3. Test through the gateway:
   ```bash
   # Get tags through gateway
   curl http://localhost:8081/tags
   
   # Health check
   curl http://localhost:8081/actuator/health
   
   # List routes
   curl http://localhost:8081/actuator/gateway/routes
   ```

## Architecture

```
                    ┌─────────────────┐
                    │     Clients     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   API Gateway   │
                    │   (port 8081)   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Monolith     │
                    │   (port 8080)   │
                    └─────────────────┘
```

## Next Steps (Future Phases)

As microservices are extracted, the gateway configuration will be updated to route specific endpoints to their respective services:

- Phase 1: Route `/users`, `/user` to User Service
- Phase 2: Route `/articles` (write operations) to Article Service
- Phase 3: Route `/profiles/{username}/follow` to Social Service
- Phase 4: Route `/articles/{slug}/comments`, `/articles/{slug}/favorite` to Engagement Service
- Phase 5: Route read operations to Query Service

## Troubleshooting

### Gateway returns 503 Service Unavailable

The monolith is not running or not accessible. Ensure the monolith is running on port 8080.

### Routes not matching

Enable debug logging to see routing decisions:

```yaml
logging:
  level:
    org.springframework.cloud.gateway: TRACE
```

### CORS issues

The gateway preserves the `Host` header. If you encounter CORS issues, ensure the monolith's CORS configuration allows requests from the gateway's origin.
