# Topic 7.3 — API Gateway: Spring Cloud Gateway

## Hedef

API Gateway pattern'ini Spring Cloud Gateway ile production-grade implement etmek. Routing, predicates, filters, JWT validation, Redis rate limiting, circuit breaker, retry, header propagation, observability. Banking için **tek entry point** + **cross-cutting concerns** soyutlama.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 7.1-7.2 bitti (DDD, decomposition)
- Phase 8'in JWT/OAuth2 conceptlerine genel aşinalık (detay Phase 8'de)
- Reactive programming basic (Mono, Flux)

---

## Kavramlar

### 1. API Gateway pattern — neden

```
Without Gateway:
Mobile App → account-service:8081
           → transfer-service:8082
           → fraud-service:8083
           → notification-service:8084

Sorunlar:
- Client her servisin URL'ini bilir (tight coupling)
- Auth her servis ayrı implement
- Rate limit her servis ayrı
- CORS her servis ayrı
- Versioning her servis ayrı
```

```
With Gateway:
Mobile App → api.banking.com
                ↓
            [API Gateway]
              ↓ ↓ ↓ ↓
       account transfer fraud notification

Faydalar:
- Single entry point
- Cross-cutting (auth, rate limit, logging, CORS) tek yerde
- Service hiding (internal endpoint'ler expose değil)
- Versioning + routing
- Backend evolution flexibility
```

**Banking için zorunlu.** TR bankalarındaki "Open Banking" API'leri gateway üzerinden.

### 2. Spring Cloud Gateway vs alternatives

| Gateway | Banking Adoption | Tech |
|---|---|---|
| **Spring Cloud Gateway** | ✓ Yüksek (Spring ekosistemde) | Java, Netty reactive |
| Kong | OK | Lua + Nginx |
| AWS API Gateway | Cloud-native | Managed |
| Apigee | Enterprise | Google managed |
| Zuul (Netflix) | Deprecated | Eski Spring projeleri |

Bu kursta **Spring Cloud Gateway** — Spring Boot + Java + reactive (Project Reactor).

### 3. Setup

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**Önemli:** Gateway **reactive** stack (Netty). Spring Web Servlet (Tomcat) ile **uyumsuz**. Standalone Spring Boot application.

```java
@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

### 4. Route definition — YAML

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: account-service
          uri: lb://account-service
          predicates:
            - Path=/v1/accounts/**
          filters:
            - name: StripPrefix
              args:
                parts: 0
        
        - id: transfer-service
          uri: lb://transfer-service
          predicates:
            - Path=/v1/transfers/**
            - Method=POST,GET
          filters:
            - name: RewritePath
              args:
                regexp: /v1/transfers/(?<segment>.*)
                replacement: /transfers/${segment}
        
        - id: fraud-service-internal
          uri: lb://fraud-service
          predicates:
            - Path=/internal/v1/fraud/**
            - Header=X-Internal-Token, .*
```

**Anahtar kavramlar:**

- **`id`:** Unique route identifier
- **`uri`:** Backend destination. `lb://` prefix → Spring Cloud LoadBalancer (service discovery)
- **`predicates`:** Match conditions (Path, Method, Header, Query, Cookie, Host)
- **`filters`:** Request/response transformation

### 5. Predicates — match conditions

```yaml
predicates:
  - Path=/v1/accounts/**
  - Method=GET,POST
  - Header=Authorization, Bearer.*
  - Query=tenant
  - Cookie=session, .*
  - Host=api.banking.com
  - After=2024-01-01T00:00:00+03:00[Europe/Istanbul]
  - Before=2025-12-31T23:59:59+03:00[Europe/Istanbul]
  - Between=2024-06-01T00:00:00+03:00[Europe/Istanbul],2024-09-01T00:00:00+03:00[Europe/Istanbul]
  - RemoteAddr=192.168.1.0/24
  - Weight=group-a, 80
```

Banking örnek:
- **Production traffic:** `Host=api.banking.com`
- **Internal traffic:** `Path=/internal/**` + `Header=X-Internal-Token`
- **Beta features:** `Header=X-Feature-Flag, beta` (canary)

### 6. Filters — request/response transformation

```yaml
filters:
  - AddRequestHeader=X-Gateway-Route, account
  - AddResponseHeader=X-Response-Time, "${nanosTime}"
  - StripPrefix=1
  - RewritePath=/v1/(?<segment>.*), /$\{segment}
  - SetRequestHeader=X-Tenant-Id, default
  - PreserveHostHeader
  - RemoveRequestHeader=X-Internal-Debug
  - RemoveResponseHeader=Server
```

Custom filters için Java DSL daha güçlü.

### 7. Java DSL alternative

```java
@Configuration
public class RoutesConfig {
    
    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("account-service", r -> r
                .path("/v1/accounts/**")
                .filters(f -> f
                    .filter(new JwtAuthenticationFilter())
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                    .circuitBreaker(c -> c
                        .setName("accountServiceCB")
                        .setFallbackUri("forward:/fallback/account")))
                .uri("lb://account-service"))
            
            .route("transfer-service", r -> r
                .path("/v1/transfers/**")
                .and().header("Authorization", "Bearer.*")
                .filters(f -> f
                    .filter(new IdempotencyKeyFilter())   // banking-specific
                    .filter(new JwtAuthenticationFilter())
                    .requestRateLimiter(c -> c
                        .setRateLimiter(strictRedisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://transfer-service"))
            
            .route("public-info", r -> r
                .path("/v1/public/**")
                .filters(f -> f.stripPrefix(1))
                .uri("lb://public-info-service"))
            
            .build();
    }
}
```

### 8. Load balancing

`uri: lb://account-service` → Spring Cloud LoadBalancer kullanır.

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 5s
      health-check:
        interval: 10s
```

Service discovery'den (Topic 7.4) instance listesi alır, round-robin (veya custom strategy) ile dağıtır.

### 9. JWT Authentication Filter

```java
@Component
public class JwtAuthenticationFilter extends AbstractGatewayFilterFactory<JwtAuthenticationFilter.Config> {
    
    private final JwtValidator jwtValidator;
    
    public JwtAuthenticationFilter(JwtValidator jwtValidator) {
        super(Config.class);
        this.jwtValidator = jwtValidator;
    }
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
            
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return unauthorized(exchange);
            }
            
            String token = authHeader.substring(7);
            
            try {
                Claims claims = jwtValidator.validate(token);
                
                String userId = claims.getSubject();
                String tenant = claims.get("tenant", String.class);
                List<String> roles = claims.get("roles", List.class);
                
                // Required role check (config'ten)
                if (config.getRequiredRole() != null && !roles.contains(config.getRequiredRole())) {
                    return forbidden(exchange);
                }
                
                // Propagate to downstream
                ServerHttpRequest mutated = exchange.getRequest().mutate()
                    .header("X-User-Id", userId)
                    .header("X-Tenant-Id", tenant != null ? tenant : "default")
                    .header("X-User-Roles", String.join(",", roles))
                    .build();
                
                return chain.filter(exchange.mutate().request(mutated).build());
            } catch (ExpiredJwtException e) {
                return unauthorized(exchange, "Token expired");
            } catch (JwtException e) {
                return unauthorized(exchange, "Invalid token");
            }
        };
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        return unauthorized(exchange, "Authentication required");
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange, String reason) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().add(HttpHeaders.WWW_AUTHENTICATE, "Bearer");
        return exchange.getResponse().setComplete();
    }
    
    private Mono<Void> forbidden(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        return exchange.getResponse().setComplete();
    }
    
    @Data
    public static class Config {
        private String requiredRole;
    }
}
```

Usage:

```yaml
filters:
  - name: JwtAuthentication
    args:
      requiredRole: customer
```

### 10. Rate limiting — Redis-backed

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10        # 10 token/saniye
      redis-rate-limiter.burstCapacity: 20         # max 20 token (burst)
      redis-rate-limiter.requestedTokens: 1        # her request 1 token
      key-resolver: "#{@userKeyResolver}"
```

Key resolver — kim'e göre rate limit:

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        if (userId == null) {
            // Anonymous user — IP-based
            String ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
            return Mono.just("ip:" + ip);
        }
        return Mono.just("user:" + userId);
    };
}

@Bean
public KeyResolver tenantKeyResolver() {
    return exchange -> {
        String tenant = exchange.getRequest().getHeaders().getFirst("X-Tenant-Id");
        return Mono.just("tenant:" + tenant);
    };
}
```

**Banking örnek:**
- **User-level:** 10 req/sec (normal)
- **VIP user:** 100 req/sec
- **Anonymous (login attempt):** 5 req/min (brute force protection)
- **Internal services:** 1000 req/sec

Rate limit aşıldı → HTTP 429.

### 11. Circuit Breaker filter (Resilience4j)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: accountServiceCB
      fallbackUri: forward:/fallback/account
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      accountServiceCB:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
```

Fallback handler:

```java
@RestController
@RequestMapping("/fallback")
public class FallbackController {
    
    @GetMapping("/account")
    public Mono<ResponseEntity<ProblemDetail>> accountFallback() {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.SERVICE_UNAVAILABLE,
            "Account service is temporarily unavailable. Please retry."
        );
        problem.setType(URI.create("https://api.mavibank.com/problems/service-unavailable"));
        problem.setTitle("Service Unavailable");
        problem.setProperty("code", "ACCOUNT_SERVICE_DOWN");
        
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem));
    }
}
```

Phase 7 Topic 5'te (Resilience4j) detay.

### 12. Retry filter

```yaml
filters:
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE, GATEWAY_TIMEOUT
      methods: GET, HEAD
      backoff:
        firstBackoff: 100ms
        maxBackoff: 1000ms
        factor: 2
        basedOnPreviousValue: false
```

**Önemli:** **GET, HEAD only** — POST/PUT retry idempotent değil (duplicate transfer riski).

Banking için POST retry: Sadece `Idempotency-Key` header varsa OK. Filter custom.

### 13. Custom IdempotencyKeyFilter — banking pattern

```java
@Component
public class IdempotencyKeyFilter extends AbstractGatewayFilterFactory<Object> {
    
    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> {
            HttpMethod method = exchange.getRequest().getMethod();
            
            if (HttpMethod.POST.equals(method) || HttpMethod.PUT.equals(method)) {
                String idempotencyKey = exchange.getRequest().getHeaders().getFirst("Idempotency-Key");
                
                if (idempotencyKey == null) {
                    return badRequest(exchange, "Idempotency-Key header required for POST/PUT");
                }
                
                try {
                    UUID.fromString(idempotencyKey);
                } catch (IllegalArgumentException e) {
                    return badRequest(exchange, "Invalid Idempotency-Key format (must be UUID)");
                }
            }
            
            return chain.filter(exchange);
        };
    }
    
    private Mono<Void> badRequest(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
        return exchange.getResponse().setComplete();
    }
}
```

Banking pattern: `/v1/transfers` POST'ta `Idempotency-Key` zorunlu (Phase 1 öğretisi).

### 14. Logging filter — request tracing

```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String requestId = exchange.getRequest().getHeaders().getFirst("X-Request-Id");
        if (requestId == null) {
            requestId = UUID.randomUUID().toString();
            ServerHttpRequest mutated = exchange.getRequest().mutate()
                .header("X-Request-Id", requestId)
                .header("X-Trace-Id", requestId)   // OpenTelemetry için de
                .build();
            exchange = exchange.mutate().request(mutated).build();
        }
        
        final String finalRequestId = requestId;
        long startTime = System.currentTimeMillis();
        
        log.info("[{}] {} {} ", 
            requestId, 
            exchange.getRequest().getMethod(), 
            exchange.getRequest().getURI());
        
        ServerHttpResponse response = exchange.getResponse();
        response.beforeCommit(() -> {
            long duration = System.currentTimeMillis() - startTime;
            log.info("[{}] {} {} - {} ({}ms)",
                finalRequestId,
                exchange.getRequest().getMethod(),
                exchange.getRequest().getURI(),
                response.getStatusCode(),
                duration);
            response.getHeaders().add("X-Request-Id", finalRequestId);
            response.getHeaders().add("X-Response-Time", duration + "ms");
            return Mono.empty();
        });
        
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() {
        return -1;   // ilk çalış (logging her route için)
    }
}
```

`GlobalFilter` — tüm route'larda.
`Ordered` — filter chain'de sırası.

### 15. CORS configuration

Banking — mobile app + web app farklı origin'lerden çağrı:

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins:
              - https://www.banking.com
              - https://mobile.banking.com
            allowed-methods: [GET, POST, PUT, PATCH, DELETE, OPTIONS]
            allowed-headers: [Authorization, Content-Type, X-Trace-Id, Idempotency-Key]
            exposed-headers: [X-Request-Id, X-Response-Time]
            allow-credentials: true
            max-age: 3600
```

**Banking pratiği:** `allowed-origins: ["*"]` **YASAK** — sadece bilinen domain'ler.

### 16. Header propagation downstream

Banking — backend service'lere header'lar:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - AddRequestHeader=X-Source, api-gateway
        - PreserveHostHeader
```

Backend service `X-User-Id`, `X-Tenant-Id` JWT'den extract'lenmiş, downstream'e propagate.

### 17. Banking örnek — full gateway

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - AddRequestHeader=X-Source, api-gateway
        - DedupeResponseHeader=Access-Control-Allow-Origin
      
      routes:
        # Account API — customer endpoint
        - id: account-api
          uri: lb://account-service
          predicates:
            - Path=/v1/accounts/**
          filters:
            - name: JwtAuthentication
              args:
                requiredRole: customer
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: accountServiceCB
                fallbackUri: forward:/fallback/account
        
        # Transfer API — sensitive, sıkı rate limit
        - id: transfer-api
          uri: lb://transfer-service
          predicates:
            - Path=/v1/transfers/**
          filters:
            - name: IdempotencyKey
            - name: JwtAuthentication
              args:
                requiredRole: customer
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: transferServiceCB
                fallbackUri: forward:/fallback/transfer
        
        # Auth (Keycloak) — public path
        - id: auth
          uri: http://keycloak:8080
          predicates:
            - Path=/auth/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 5
                redis-rate-limiter.burstCapacity: 10
                key-resolver: "#{@ipKeyResolver}"   # brute force protection
        
        # Internal API — JWT yok, mTLS şart
        - id: internal-api
          uri: lb://account-service
          predicates:
            - Path=/internal/v1/accounts/**
            - Header=X-Internal-Token, .*
          filters:
            - name: InternalTokenAuthentication
        
        # Admin API — high privilege
        - id: admin-api
          uri: lb://admin-service
          predicates:
            - Path=/admin/v1/**
          filters:
            - name: JwtAuthentication
              args:
                requiredRole: admin
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 50
                redis-rate-limiter.burstCapacity: 100
                key-resolver: "#{@userKeyResolver}"
      
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins:
              - https://www.banking.com
              - https://mobile.banking.com
            allowed-methods: [GET, POST, PUT, PATCH, DELETE]
            allowed-headers: "*"
            allow-credentials: true

resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
    instances:
      accountServiceCB:
        baseConfig: default
      transferServiceCB:
        baseConfig: default
        failureRateThreshold: 30   # daha sıkı (critical service)

management:
  endpoints:
    web:
      exposure:
        include: gateway, health, metrics, prometheus
```

### 18. Observability — metrics + tracing

```yaml
spring:
  cloud:
    gateway:
      metrics:
        enabled: true
  application:
    name: api-gateway

management:
  metrics:
    tags:
      application: api-gateway
```

Otomatik metric'ler:
- `spring.cloud.gateway.requests` (counter)
- `gateway.routes.count` (gauge)
- `gateway.requests.seconds` (timer with percentiles)

OpenTelemetry tracing — Phase 9 Topic 3.

### 19. Banking anti-pattern'leri

**Anti-pattern 1: Gateway'i business logic için kullanmak**

```java
.filter((exchange, chain) -> {
    // ❌ Business decision in gateway
    if (isHighRiskCountry(getCountry(exchange))) {
        return chain.filter(exchange.mutate().header("X-Fraud-Flag", "true").build());
    }
})
```

Gateway sadece cross-cutting concerns. Business logic backend'de.

**Anti-pattern 2: Tüm cross-cutting'i gateway'e koymak**

Gateway SPOF. Some cross-cutting backend'in kendi sorumluluğu (domain validation, business rules).

**Anti-pattern 3: Gateway sync chain (4+ hop)**

```
Gateway → A → B → C → D
```

5 hop sync. Latency × 5. Network failure × 5. Async event'e taşı veya direct call (service mesh).

**Anti-pattern 4: Hard-coded route URIs**

```yaml
uri: http://account-service:8081
```

Environment-specific config patlamaları. Service discovery (`lb://`).

**Anti-pattern 5: `allowed-origins: ["*"]`**

CORS wide-open → CSRF riski.

**Anti-pattern 6: Authentication backend service'lerde tekrarlanıyor**

Gateway JWT validate ettikten sonra backend de yapıyor. **Duplicate work + inconsistency**.

**Doğrusu:** Backend `X-User-Id` header'a güvenir (gateway tarafından validated). Internal endpoint'ler mTLS ile korunur — JWT bypass için.

**Anti-pattern 7: Public ve internal endpoint aynı port'ta**

Gateway external traffic için. Internal service-to-service direkt çağrı (gateway bypass) veya internal gateway ayrı.

---

## Önemli olabilecek araştırma kaynakları

- Spring Cloud Gateway reference (current 4.x)
- Project Reactor documentation (Mono/Flux)
- "Cloud Native Spring in Action" — gateway chapter
- Resilience4j + Spring Cloud Gateway integration docs
- Redis rate limiter algorithm (token bucket)
- "Building Microservices" (Sam Newman) — gateway chapter

---

## Mini task'ler

### Task 7.3.1 — Gateway Spring Boot project setup (30 dk)

```xml
<dependencies>
    <dependency>spring-cloud-starter-gateway</dependency>
    <dependency>spring-cloud-starter-loadbalancer</dependency>
    <dependency>spring-boot-starter-data-redis-reactive</dependency>
    <dependency>spring-cloud-starter-circuitbreaker-reactor-resilience4j</dependency>
</dependencies>
```

`api-gateway` Maven module. `ApiGatewayApplication` main class. Port 8080.

### Task 7.3.2 — 4 route YAML (30 dk)

`account-api`, `transfer-api`, `auth`, `admin-api` route'ları YAML'da. Test: curl ile her endpoint'e istek atıp doğru servise gittiğini logla.

### Task 7.3.3 — JwtAuthenticationFilter (60 dk)

Yukarıdaki implementation. Test:
- Authorization header yok → 401
- Invalid token → 401
- Expired token → 401  
- Valid token + insufficient role → 403
- Valid + correct role → backend'e geçer, X-User-Id propagate

### Task 7.3.4 — Redis rate limiter (45 dk)

Redis docker container. `userKeyResolver` + `ipKeyResolver` bean'leri. 
- Per-user: 10 req/sec
- Per-IP (anonymous): 5 req/min
- Per-tenant: 100 req/sec

Test: Aynı kullanıcıdan 1 saniyede 20 request → 10 başarılı, 10 → 429.

### Task 7.3.5 — Circuit breaker + fallback (45 dk)

`accountServiceCB` circuit breaker. Backend down → fallback URI `/fallback/account` → ProblemDetail 503.

Test:
- Account service up → normal response
- Account service down → 100 request'ten 50+ fail sonra CB OPEN → fallback response anında

### Task 7.3.6 — IdempotencyKeyFilter (30 dk)

POST/PUT için Idempotency-Key zorunlu. Test:
- POST /v1/transfers without header → 400
- POST with invalid UUID → 400
- POST with valid UUID → backend'e geçer

### Task 7.3.7 — LoggingFilter (30 dk)

GlobalFilter. X-Request-Id otomatik üret veya pass-through. Request ve response log. X-Response-Time header.

Test: Her response'ta `X-Request-Id` ve `X-Response-Time` görünür.

### Task 7.3.8 — End-to-end test (45 dk)

`@SpringBootTest` + WebTestClient. Backend service mock'lar. Tüm filter chain test:
- JWT validation
- Rate limiting
- Circuit breaker
- IdempotencyKey
- Logging

---

## Test yazma rehberi

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
@Testcontainers
class ApiGatewayIT {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void redisProps(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
    
    @Autowired WebTestClient webClient;
    
    @MockBean AccountServiceClient accountServiceClient;
    
    @Test
    void shouldReturn401WithoutAuthHeader() {
        webClient.get().uri("/v1/accounts/123")
            .exchange()
            .expectStatus().isUnauthorized();
    }
    
    @Test
    void shouldReturn401WithInvalidToken() {
        webClient.get().uri("/v1/accounts/123")
            .header("Authorization", "Bearer invalid.token.here")
            .exchange()
            .expectStatus().isUnauthorized();
    }
    
    @Test
    void shouldRouteValidRequestToAccountService() {
        String validJwt = generateValidJwt("user-123", "customer");
        
        webClient.get().uri("/v1/accounts/123")
            .header("Authorization", "Bearer " + validJwt)
            .exchange()
            .expectStatus().isOk();
        
        // Verify backend received correct headers
        verify(accountServiceClient).getAccount(eq("123"), 
            argThat(headers -> 
                headers.getFirst("X-User-Id").equals("user-123") &&
                headers.getFirst("X-Tenant-Id") != null
            ));
    }
    
    @Test
    void shouldRateLimitAfterThreshold() throws InterruptedException {
        String validJwt = generateValidJwt("user-spam", "customer");
        
        // 20 request hızlı gönder
        int successCount = 0;
        int rateLimitedCount = 0;
        for (int i = 0; i < 20; i++) {
            EntityExchangeResult<byte[]> result = webClient.get().uri("/v1/accounts/123")
                .header("Authorization", "Bearer " + validJwt)
                .exchange()
                .returnResult(byte[].class);
            
            if (result.getStatus().value() == 200) successCount++;
            else if (result.getStatus().value() == 429) rateLimitedCount++;
        }
        
        // Banking: 10 token + 10 burst = 20 success max, sonra 429
        assertThat(successCount).isLessThanOrEqualTo(20);
        assertThat(rateLimitedCount).isGreaterThan(0);
    }
    
    @Test
    void shouldReject403WhenRoleMismatch() {
        String validJwt = generateValidJwt("user-123", "customer");   // no "admin" role
        
        webClient.get().uri("/admin/v1/users")
            .header("Authorization", "Bearer " + validJwt)
            .exchange()
            .expectStatus().isForbidden();
    }
    
    @Test
    void shouldRequireIdempotencyKeyOnTransferPost() {
        String validJwt = generateValidJwt("user-123", "customer");
        
        webClient.post().uri("/v1/transfers")
            .header("Authorization", "Bearer " + validJwt)
            .bodyValue(Map.of("amount", "100", "currency", "TRY"))
            .exchange()
            .expectStatus().isBadRequest();
    }
    
    @Test
    void shouldFallbackWhenBackendDown() {
        when(accountServiceClient.getAccount(any(), any()))
            .thenReturn(Mono.error(new ConnectException("Connection refused")));
        
        String validJwt = generateValidJwt("user-123", "customer");
        
        // 100 fail request → circuit breaker opens
        for (int i = 0; i < 100; i++) {
            webClient.get().uri("/v1/accounts/123")
                .header("Authorization", "Bearer " + validJwt)
                .exchange();
        }
        
        // Circuit OPEN → instant fallback
        webClient.get().uri("/v1/accounts/123")
            .header("Authorization", "Bearer " + validJwt)
            .exchange()
            .expectStatus().isEqualTo(HttpStatus.SERVICE_UNAVAILABLE)
            .expectHeader().contentType(MediaType.APPLICATION_PROBLEM_JSON);
    }
}
```

---

## Claude-verify prompt

```
API Gateway implementation'ımı banking-grade kriterlere göre değerlendir:

1. Routing:
   - Path predicate doğru mu?
   - Method, Header predicate'leri uygun mu?
   - lb:// ile service discovery integration?

2. Filters:
   - JwtAuthentication tüm protected route'larda?
   - IdempotencyKeyFilter POST/PUT için?
   - Header propagation (X-User-Id, X-Tenant-Id, X-Trace-Id)?

3. Rate limiting:
   - Redis-backed (in-memory DEĞİL)?
   - Per-user + per-IP key resolver?
   - Endpoint-specific limits (login sıkı, normal API gevşek)?

4. Circuit breaker:
   - Her backend service için tanımlı?
   - Fallback URI ile ProblemDetail response?
   - Sliding window + failure threshold makul?

5. CORS:
   - allowed-origins explicit (wildcard YOK)?
   - Banking domain'leri whitelisted?

6. Observability:
   - GlobalFilter ile request logging?
   - X-Request-Id otomatik üretim/propagate?
   - Micrometer metrics?
   - OpenTelemetry tracing entegre?

7. Banking-specific:
   - IdempotencyKeyFilter banking pattern?
   - Internal endpoint'ler ayrı route + mTLS?
   - Admin endpoint role-based access?
   - Public endpoint (login) brute force protection?

8. Anti-pattern:
   - Business logic gateway'de?
   - Hard-coded URIs?
   - Backend service'lerde duplicate auth?
   - Wildcard CORS?
   - Sync HTTP chain 4+ hop?

9. Performance:
   - Reactive (Netty) — servlet (Tomcat) mix yok mu?
   - Load balancer cache aktif?

10. Test:
    - WebTestClient ile filter chain test?
    - JWT validation senaryoları?
    - Rate limit threshold test?
    - Circuit breaker open behavior?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Gateway Spring Boot project setup (Netty reactive)
- [ ] 5+ route YAML (account, transfer, auth, internal, admin)
- [ ] JwtAuthenticationFilter implement + header propagation
- [ ] Redis-backed rate limiter (per-user, per-IP)
- [ ] Circuit breaker + fallback (her backend için)
- [ ] IdempotencyKeyFilter (banking pattern)
- [ ] LoggingFilter + X-Request-Id
- [ ] CORS configuration banking domains
- [ ] Observability (metrics + tracing)
- [ ] 8+ integration test (WebTestClient)

---

## Defter notları (10 madde)

1. "API Gateway 4 ana sorumluluğu (entry point, cross-cutting, service hiding, evolution): ____."
2. "Spring Cloud Gateway reactive (Netty) — servlet ile uyumsuzluk: ____."
3. "Predicate vs Filter farkı: ____."
4. "JWT validation gateway'de + backend'de duplicate olmama prensibi: ____."
5. "Redis rate limiter token bucket algorithm (replenishRate + burstCapacity): ____."
6. "Circuit breaker fallback URI banking ProblemDetail: ____."
7. "IdempotencyKeyFilter banking POST/PUT zorunluluk: ____."
8. "Public (login) vs internal (mTLS) vs admin (role) endpoint ayrımı: ____."
9. "Gateway business logic anti-pattern + neden: ____."
10. "Gateway SPOF mitigation (3+ replica, K8s HPA): ____."
