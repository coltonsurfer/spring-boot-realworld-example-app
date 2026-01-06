# API Gateway Migration Plan for Spring Boot Microservices

This document outlines a comprehensive migration plan for implementing an API Gateway to route traffic from the existing Spring Boot 2.6.3 monolith to a microservices architecture using the Strangler Fig Pattern.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Target Microservices Architecture](#target-microservices-architecture)
4. [API Gateway Responsibilities](#api-gateway-responsibilities)
5. [Routing Strategy](#routing-strategy)
6. [Authentication Flow Design](#authentication-flow-design)
7. [Migration Phases](#migration-phases)
8. [Cross-Service Data Aggregation](#cross-service-data-aggregation)
9. [Technology Recommendations](#technology-recommendations)
10. [Monitoring and Observability](#monitoring-and-observability)
11. [Risk Mitigation](#risk-mitigation)

## Executive Summary

The RealWorld application is a Spring Boot 2.6.3 monolith that implements a Medium-like blogging platform with both REST and GraphQL APIs. This migration plan describes how to introduce an API Gateway that will serve as the single entry point for all client requests, enabling gradual extraction of microservices while maintaining backward compatibility and zero downtime.

The migration follows the Strangler Fig Pattern, where the API Gateway initially routes all traffic to the monolith, then progressively redirects traffic to new microservices as they are extracted and deployed.

## Current Architecture Analysis

### Application Entry Point

The monolith entry point is `io.spring.RealWorldApplication`, a standard Spring Boot application that serves both REST and GraphQL endpoints on a single port (default: 8080).

### REST API Endpoints

The current REST API is organized into the following controllers:

**UsersApi** (`/users`)
- `POST /users` - User registration (public)
- `POST /users/login` - User authentication (public)

**CurrentUserApi** (`/user`)
- `GET /user` - Get current user (authenticated)
- `PUT /user` - Update current user (authenticated)

**ArticlesApi** (`/articles`)
- `POST /articles` - Create article (authenticated)
- `GET /articles` - List articles with filters (public)
- `GET /articles/feed` - Get user feed (authenticated)

**ArticleApi** (`/articles/{slug}`)
- `GET /articles/{slug}` - Get article by slug (public)
- `PUT /articles/{slug}` - Update article (authenticated, owner only)
- `DELETE /articles/{slug}` - Delete article (authenticated, owner only)

**ArticleFavoriteApi** (`/articles/{slug}/favorite`)
- `POST /articles/{slug}/favorite` - Favorite article (authenticated)
- `DELETE /articles/{slug}/favorite` - Unfavorite article (authenticated)

**CommentsApi** (`/articles/{slug}/comments`)
- `POST /articles/{slug}/comments` - Create comment (authenticated)
- `GET /articles/{slug}/comments` - List comments (public)
- `DELETE /articles/{slug}/comments/{id}` - Delete comment (authenticated, owner only)

**ProfileApi** (`/profiles/{username}`)
- `GET /profiles/{username}` - Get user profile (public)
- `POST /profiles/{username}/follow` - Follow user (authenticated)
- `DELETE /profiles/{username}/follow` - Unfollow user (authenticated)

**TagsApi** (`/tags`)
- `GET /tags` - List all tags (public)

### GraphQL API

The GraphQL schema (`src/main/resources/schema/schema.graphqls`) exposes:

**Queries:**
- `article(slug: String!)` - Get single article
- `articles(...)` - List articles with pagination and filters
- `me` - Get current user
- `feed(...)` - Get user feed
- `profile(username: String!)` - Get user profile
- `tags` - List all tags

**Mutations:**
- User: `createUser`, `login`, `updateUser`, `followUser`, `unfollowUser`
- Article: `createArticle`, `updateArticle`, `favoriteArticle`, `unfavoriteArticle`, `deleteArticle`
- Comment: `addComment`, `deleteComment`

### Authentication Mechanism

The application uses JWT token-based authentication implemented in:
- `JwtService` interface with `DefaultJwtService` implementation using JJWT library (HS512 algorithm)
- `JwtTokenFilter` extracts tokens from `Authorization: Token <jwt>` header
- `WebSecurityConfig` defines public vs authenticated endpoints
- Tokens contain user ID as subject with configurable expiration

### Data Aggregation Patterns

The `ArticleQueryService.fillExtraInfo()` method demonstrates the cross-cutting data aggregation challenge:
- Fetches article data from `ArticleReadService`
- Enriches with favorite count from `ArticleFavoritesReadService`
- Enriches with user's favorite status from `ArticleFavoritesReadService`
- Enriches with author following status from `UserRelationshipQueryService`

Similar patterns exist in `CommentQueryService` and `ProfileQueryService`.

## Target Microservices Architecture

### Proposed Services

**1. User Service**
- Owns: `users` table
- Responsibilities: User registration, authentication, JWT token generation, user profile updates
- Endpoints: `/users`, `/users/login`, `/user`

**2. Article Service**
- Owns: `articles`, `tags`, `article_tags` tables
- Responsibilities: Article CRUD, tag management
- Endpoints: `/articles`, `/articles/{slug}`, `/tags`

**3. Social Service**
- Owns: `follows` table
- Responsibilities: Follow/unfollow relationships, profile data
- Endpoints: `/profiles/{username}`, `/profiles/{username}/follow`

**4. Engagement Service**
- Owns: `comments`, `article_favorites` tables
- Responsibilities: Comments, favorites, engagement metrics
- Endpoints: `/articles/{slug}/comments`, `/articles/{slug}/favorite`

**5. Query Service (CQRS Read Model)**
- Owns: Read-optimized views/caches
- Responsibilities: Complex queries, feed generation, data aggregation
- Consumes events from other services to maintain materialized views

## API Gateway Responsibilities

### Primary Functions

**Request Routing**
The gateway routes incoming requests to the appropriate backend service based on URL path patterns and HTTP methods. During migration, it supports routing to both the monolith and new microservices simultaneously.

**Authentication and Authorization**
The gateway validates JWT tokens before forwarding requests to backend services. For authenticated endpoints, it extracts user information from the token and passes it to downstream services via headers.

**Protocol Translation**
The gateway handles both REST and GraphQL requests, routing them appropriately. For GraphQL, it can either proxy to a federated GraphQL gateway or route individual queries/mutations to specific services.

**Rate Limiting and Throttling**
The gateway implements rate limiting to protect backend services from abuse and ensure fair resource allocation.

**Request/Response Transformation**
The gateway can transform requests and responses to maintain backward compatibility during migration, allowing internal API changes without breaking client contracts.

**Load Balancing**
The gateway distributes traffic across multiple instances of backend services for high availability and scalability.

**Circuit Breaking**
The gateway implements circuit breaker patterns to prevent cascade failures when backend services are unhealthy.

### Secondary Functions

**Request Logging and Tracing**
The gateway generates correlation IDs and propagates distributed tracing headers (e.g., W3C Trace Context) for observability.

**CORS Handling**
The gateway centralizes CORS configuration, removing the need for individual services to handle cross-origin requests.

**SSL Termination**
The gateway handles TLS termination, allowing backend services to communicate over plain HTTP internally.

## Routing Strategy

### REST API Routing Table

The following table defines how REST endpoints map to target services:

| Path Pattern | Method | Target Service | Auth Required |
|-------------|--------|----------------|---------------|
| `/users` | POST | User Service | No |
| `/users/login` | POST | User Service | No |
| `/user` | GET, PUT | User Service | Yes |
| `/articles` | POST | Article Service | Yes |
| `/articles` | GET | Query Service | No |
| `/articles/feed` | GET | Query Service | Yes |
| `/articles/{slug}` | GET | Query Service | No |
| `/articles/{slug}` | PUT, DELETE | Article Service | Yes |
| `/articles/{slug}/favorite` | POST, DELETE | Engagement Service | Yes |
| `/articles/{slug}/comments` | GET | Query Service | No |
| `/articles/{slug}/comments` | POST | Engagement Service | Yes |
| `/articles/{slug}/comments/{id}` | DELETE | Engagement Service | Yes |
| `/profiles/{username}` | GET | Query Service | No |
| `/profiles/{username}/follow` | POST, DELETE | Social Service | Yes |
| `/tags` | GET | Article Service | No |

### GraphQL Routing Strategy

For GraphQL, two approaches are recommended depending on complexity requirements:

**Option A: GraphQL Federation (Recommended)**

Deploy Apollo Federation or a similar solution where each microservice exposes its own GraphQL subgraph. The gateway composes these into a unified supergraph.

Subgraph ownership:
- User Service: `User`, `createUser`, `login`, `updateUser`, `me`
- Article Service: `Article` (base type), `createArticle`, `updateArticle`, `deleteArticle`, `tags`
- Social Service: `Profile`, `followUser`, `unfollowUser`
- Engagement Service: `Comment`, `favoriteArticle`, `unfavoriteArticle`, `addComment`, `deleteComment`
- Query Service: `articles`, `feed`, `Article.comments` (extended)

**Option B: Gateway-Level GraphQL Proxy**

For simpler deployments, the gateway can proxy all GraphQL requests to a dedicated GraphQL service that orchestrates calls to backend REST services. This approach is easier to implement initially but may become a bottleneck.

### Backward Compatibility

During migration, the gateway maintains backward compatibility by:

1. Preserving all existing URL patterns and response formats
2. Supporting both `Authorization: Token <jwt>` and `Authorization: Bearer <jwt>` header formats
3. Returning consistent error response structures
4. Maintaining pagination parameters (`offset`, `limit`) alongside cursor-based pagination

## Authentication Flow Design

### Token Generation (User Service)

The User Service is the sole authority for JWT token generation. When a user registers or logs in:

1. User Service validates credentials against its database
2. User Service generates a JWT token containing:
   - `sub`: User ID (UUID)
   - `exp`: Expiration timestamp
   - `iat`: Issued at timestamp
3. Token is signed using HS512 algorithm with a shared secret
4. Token is returned to the client in the response body

### Token Validation (API Gateway)

For authenticated requests, the gateway performs token validation:

1. Extract token from `Authorization` header (supports both `Token` and `Bearer` prefixes)
2. Validate token signature using the shared JWT secret
3. Check token expiration
4. Extract user ID from token subject
5. For valid tokens, add `X-User-Id` header to downstream request
6. For invalid/expired tokens, return 401 Unauthorized

### Downstream Service Authentication

Backend services trust the gateway's authentication:

1. Services check for `X-User-Id` header from gateway
2. If present, the request is considered authenticated
3. Services can fetch additional user details from User Service if needed
4. Internal service-to-service calls use separate authentication (e.g., mTLS or service tokens)

### JWT Secret Management

The JWT signing secret must be shared between:
- User Service (for token generation)
- API Gateway (for token validation)

Recommendations:
- Store secret in a secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager)
- Rotate secrets periodically with overlap period for token validity
- Consider asymmetric keys (RS256) for larger deployments where only User Service needs the private key

## Migration Phases

### Phase 0: Gateway Introduction (Week 1-2)

**Objective:** Deploy API Gateway as a pass-through proxy to the monolith.

**Steps:**
1. Deploy Spring Cloud Gateway or Kong alongside the monolith
2. Configure gateway to route all traffic to monolith
3. Update DNS/load balancer to point to gateway
4. Verify all existing functionality works through gateway
5. Implement request logging and correlation IDs

**Validation:**
- All existing tests pass when run against gateway
- Response times within acceptable threshold (< 50ms added latency)
- No errors in gateway logs

**Rollback:** Point DNS back to monolith directly.

### Phase 1: User Service Extraction (Week 3-5)

**Objective:** Extract user registration, authentication, and profile management.

**Steps:**
1. Create User Service with its own database (copy `users` table)
2. Implement endpoints: `/users`, `/users/login`, `/user`
3. Move JWT generation logic to User Service
4. Configure gateway to route user endpoints to new service
5. Keep monolith's user data in sync via database replication or events
6. Gradually shift traffic using canary deployment (10% -> 50% -> 100%)

**Data Migration:**
- Set up CDC (Change Data Capture) from monolith to User Service
- User Service becomes source of truth for user data
- Monolith reads user data from User Service or shared cache

**Validation:**
- User registration creates users in new service
- Login works and returns valid JWT
- Existing sessions continue to work

### Phase 2: Article Service Extraction (Week 6-8)

**Objective:** Extract article and tag management.

**Steps:**
1. Create Article Service with `articles`, `tags`, `article_tags` tables
2. Implement endpoints: `/articles` (POST), `/articles/{slug}` (PUT, DELETE), `/tags`
3. Configure gateway routing for article write operations
4. Keep read operations on monolith initially (Query Service pattern)
5. Implement event publishing for article changes

**Data Migration:**
- Replicate article data to new service
- Article Service becomes source of truth for writes
- Monolith/Query Service handles reads during transition

**Validation:**
- Article creation works through new service
- Article updates and deletes work
- Tags are properly managed

### Phase 3: Social Service Extraction (Week 9-10)

**Objective:** Extract follow/unfollow functionality.

**Steps:**
1. Create Social Service with `follows` table
2. Implement endpoints: `/profiles/{username}/follow` (POST, DELETE)
3. Implement profile data aggregation (combines user data + follow status)
4. Configure gateway routing

**Data Migration:**
- Replicate follows data to new service
- Social Service becomes source of truth for relationships

**Validation:**
- Follow/unfollow operations work
- Profile endpoints return correct following status

### Phase 4: Engagement Service Extraction (Week 11-13)

**Objective:** Extract comments and favorites functionality.

**Steps:**
1. Create Engagement Service with `comments`, `article_favorites` tables
2. Implement endpoints for comments and favorites
3. Implement event publishing for engagement metrics
4. Configure gateway routing

**Data Migration:**
- Replicate engagement data to new service
- Engagement Service becomes source of truth

**Validation:**
- Comment CRUD operations work
- Favorite/unfavorite operations work
- Engagement counts are accurate

### Phase 5: Query Service Implementation (Week 14-16)

**Objective:** Implement CQRS read model for complex queries.

**Steps:**
1. Create Query Service with read-optimized data store
2. Consume events from all services to build materialized views
3. Implement complex queries: article lists, feeds, search
4. Migrate read endpoints from monolith to Query Service
5. Implement caching layer (Redis)

**Data Model:**
- Denormalized article views with embedded author, tags, engagement counts
- Pre-computed feed data per user
- Full-text search indexes

**Validation:**
- Article list queries return correct data
- Feed queries are performant
- Data consistency with write services

### Phase 6: Monolith Decommissioning (Week 17-18)

**Objective:** Remove monolith from production.

**Steps:**
1. Verify all traffic is routed to microservices
2. Remove monolith routes from gateway
3. Archive monolith codebase
4. Clean up shared resources

**Validation:**
- All functionality works without monolith
- No traffic reaching monolith
- Performance meets SLAs

## Cross-Service Data Aggregation

### The Challenge

The current `ArticleQueryService.fillExtraInfo()` method aggregates data from multiple sources:
- Article data (Article Service)
- Favorite count (Engagement Service)
- User's favorite status (Engagement Service)
- Author following status (Social Service)

In a microservices architecture, this requires cross-service communication.

### Solution: Query Service with Event Sourcing

**Event-Driven Materialized Views**

Each service publishes domain events:
- Article Service: `ArticleCreated`, `ArticleUpdated`, `ArticleDeleted`
- Engagement Service: `ArticleFavorited`, `ArticleUnfavorited`, `CommentAdded`, `CommentDeleted`
- Social Service: `UserFollowed`, `UserUnfollowed`
- User Service: `UserCreated`, `UserUpdated`

Query Service consumes these events and maintains denormalized views:

```
ArticleView {
  id, slug, title, description, body, createdAt, updatedAt,
  author: { id, username, bio, image },
  tagList: [...],
  favoritesCount: int,
  // User-specific fields computed at query time
}
```

**Request-Time Enrichment**

For user-specific data (favorited, following), Query Service:
1. Fetches base article data from materialized view
2. Calls Engagement Service to check if current user favorited
3. Calls Social Service to check if current user follows author
4. Combines results and returns

**Caching Strategy**

- Cache article views in Redis with TTL
- Invalidate cache on relevant events
- Use read-through caching for user-specific data

### Alternative: API Composition at Gateway

For simpler cases, the gateway can compose responses:

1. Gateway receives request for `/articles/{slug}`
2. Gateway calls Article Service for base data
3. Gateway calls Engagement Service for favorite data
4. Gateway calls Social Service for follow data
5. Gateway merges responses and returns to client

This approach is simpler but adds latency and complexity to the gateway.

## Technology Recommendations

### API Gateway Options

**Spring Cloud Gateway (Recommended)**

Pros:
- Native Spring Boot integration
- Reactive, non-blocking architecture
- Built-in support for circuit breakers, rate limiting
- Easy integration with Spring Security for JWT validation
- Compatible with Spring Boot 2.6.3

Cons:
- Requires Java expertise
- Less feature-rich than dedicated API gateways

**Kong Gateway**

Pros:
- Language-agnostic
- Rich plugin ecosystem
- Excellent performance
- Built-in admin API and dashboard

Cons:
- Additional infrastructure component
- Learning curve for Lua plugins

**Recommendation:** Start with Spring Cloud Gateway for consistency with existing stack. Consider Kong for larger scale deployments.

### GraphQL Federation

**Apollo Federation**

Pros:
- Industry standard for GraphQL federation
- Excellent tooling and documentation
- Supports gradual migration

Cons:
- Additional infrastructure (Apollo Router)
- Requires schema design changes

**Netflix DGS Federation**

Pros:
- Already using DGS in monolith
- Native Spring Boot integration
- Simpler setup for Spring ecosystem

Cons:
- Less mature than Apollo
- Smaller community

**Recommendation:** Use Netflix DGS Federation for consistency with existing GraphQL implementation.

### Service Communication

**Synchronous (REST/gRPC)**
- Use for real-time queries where latency matters
- gRPC recommended for internal service-to-service calls

**Asynchronous (Events)**
- Use Apache Kafka or RabbitMQ for event streaming
- Essential for CQRS and eventual consistency

**Recommendation:** 
- REST for external APIs (backward compatibility)
- gRPC for internal synchronous calls
- Kafka for event streaming

### Service Discovery

**Spring Cloud Netflix Eureka**
- Native Spring integration
- Simple setup

**Kubernetes Service Discovery**
- If deploying to Kubernetes, use native service discovery
- Simpler operational model

**Recommendation:** Use Kubernetes service discovery if deploying to K8s, otherwise Eureka.

### Configuration Management

**Spring Cloud Config**
- Centralized configuration
- Git-backed configuration
- Encryption support

**Recommendation:** Use Spring Cloud Config with Git backend.

## Monitoring and Observability

### Distributed Tracing

**Implementation:**
- Add Spring Cloud Sleuth for trace propagation
- Export traces to Jaeger or Zipkin
- Gateway generates trace IDs for all requests

**Key Traces:**
- Request latency breakdown by service
- Error rates by service and endpoint
- Dependency mapping

### Metrics

**Implementation:**
- Use Micrometer with Prometheus backend
- Export metrics from gateway and all services

**Key Metrics:**
- Request rate, latency (p50, p95, p99), error rate per endpoint
- Circuit breaker state
- Connection pool utilization
- JVM metrics (heap, GC, threads)

### Logging

**Implementation:**
- Structured JSON logging
- Correlation ID propagation
- Centralized log aggregation (ELK or Loki)

**Log Format:**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "service": "article-service",
  "traceId": "abc123",
  "spanId": "def456",
  "message": "Article created",
  "articleId": "xyz789"
}
```

### Alerting

**Critical Alerts:**
- Error rate > 1% for 5 minutes
- P99 latency > 2 seconds
- Circuit breaker open
- Service health check failures

**Warning Alerts:**
- Error rate > 0.1% for 15 minutes
- P95 latency > 1 second
- High CPU/memory utilization

### Health Checks

Each service exposes:
- `/actuator/health` - Basic health status
- `/actuator/health/liveness` - Kubernetes liveness probe
- `/actuator/health/readiness` - Kubernetes readiness probe

Gateway aggregates health from all services.

## Risk Mitigation

### Data Consistency

**Risk:** Data inconsistency between services during migration.

**Mitigation:**
- Use CDC (Change Data Capture) for data synchronization
- Implement idempotent event handlers
- Add data reconciliation jobs
- Monitor for consistency drift

### Performance Degradation

**Risk:** Added latency from gateway and service-to-service calls.

**Mitigation:**
- Implement caching at gateway and service levels
- Use connection pooling
- Consider gRPC for internal calls
- Set performance budgets and monitor

### Partial Failures

**Risk:** One service failure affects entire system.

**Mitigation:**
- Implement circuit breakers (Resilience4j)
- Define fallback behaviors
- Use bulkheads to isolate failures
- Implement retry with exponential backoff

### Security Vulnerabilities

**Risk:** New attack surface with multiple services.

**Mitigation:**
- Implement mTLS for service-to-service communication
- Use network policies to restrict traffic
- Regular security audits
- Centralize authentication at gateway

### Rollback Capability

**Risk:** Unable to rollback if migration fails.

**Mitigation:**
- Maintain monolith in runnable state during migration
- Use feature flags for traffic routing
- Keep database schemas backward compatible
- Document rollback procedures for each phase

## Appendix A: Gateway Configuration Example

```yaml
# Spring Cloud Gateway configuration
spring:
  cloud:
    gateway:
      routes:
        # User Service routes
        - id: user-registration
          uri: lb://user-service
          predicates:
            - Path=/users
            - Method=POST
        
        - id: user-login
          uri: lb://user-service
          predicates:
            - Path=/users/login
            - Method=POST
        
        - id: current-user
          uri: lb://user-service
          predicates:
            - Path=/user
          filters:
            - JwtAuthenticationFilter
        
        # Article Service routes
        - id: create-article
          uri: lb://article-service
          predicates:
            - Path=/articles
            - Method=POST
          filters:
            - JwtAuthenticationFilter
        
        # Query Service routes
        - id: list-articles
          uri: lb://query-service
          predicates:
            - Path=/articles
            - Method=GET
        
        # Fallback to monolith (during migration)
        - id: monolith-fallback
          uri: lb://monolith
          predicates:
            - Path=/**
```

## Appendix B: Event Schema Examples

```json
// ArticleCreated event
{
  "eventType": "ArticleCreated",
  "eventId": "uuid",
  "timestamp": "2024-01-15T10:30:00Z",
  "payload": {
    "articleId": "uuid",
    "slug": "how-to-train-your-dragon",
    "title": "How to train your dragon",
    "description": "Ever wonder how?",
    "body": "You have to believe",
    "authorId": "uuid",
    "tagList": ["dragons", "training"],
    "createdAt": "2024-01-15T10:30:00Z"
  }
}

// ArticleFavorited event
{
  "eventType": "ArticleFavorited",
  "eventId": "uuid",
  "timestamp": "2024-01-15T10:35:00Z",
  "payload": {
    "articleId": "uuid",
    "userId": "uuid"
  }
}

// UserFollowed event
{
  "eventType": "UserFollowed",
  "eventId": "uuid",
  "timestamp": "2024-01-15T10:40:00Z",
  "payload": {
    "followerId": "uuid",
    "followeeId": "uuid"
  }
}
```

## Appendix C: Migration Checklist

### Pre-Migration
- [ ] Document all existing API contracts
- [ ] Set up monitoring and alerting
- [ ] Create rollback procedures
- [ ] Set up CI/CD pipelines for new services
- [ ] Provision infrastructure (databases, message queues)

### Per-Service Migration
- [ ] Create service repository and project structure
- [ ] Implement domain logic and API endpoints
- [ ] Set up database and migrations
- [ ] Implement event publishing
- [ ] Write integration tests
- [ ] Deploy to staging environment
- [ ] Configure gateway routing
- [ ] Canary deployment (10% traffic)
- [ ] Monitor for errors and performance
- [ ] Gradual traffic increase (50%, 100%)
- [ ] Verify data consistency
- [ ] Update documentation

### Post-Migration
- [ ] Remove monolith routes from gateway
- [ ] Archive monolith codebase
- [ ] Clean up shared resources
- [ ] Update architecture documentation
- [ ] Conduct post-mortem review
