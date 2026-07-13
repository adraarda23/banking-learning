# Topic 7.5 — Resilience4j: Circuit Breaker, Retry, Bulkhead, RateLimiter, TimeLimiter, Fallback

## Hedef

Microservice'ler arası çağrılarda **resilience pattern**'leri uygulamak. Cascading failure, timeout cascade, retry storm gibi distributed sistem patolojilerini Resilience4j ile önlemek. Banking için **transfer-service → account-service** gibi critical path'lerde fail-safe pattern.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 7.1-7.4 bitti (microservice architecture)
- Phase 3 (concurrency) — bulkhead concept
- Phase 6 (Kafka) — retry, DLT patterns

---

## Kavramlar

### 1. Neden resilience patterns gerekli

**Distributed sistem patolojileri:**

#### Cascading failure

```
account-service yavaşladı (500ms → 5s)
   ↓
transfer-service'in thread pool dolu (50 thread × 5s = 250s wait)
   ↓
gateway timeout (30s timeout)
   ↓
external client retry
   ↓
yeni request'ler thread pool'u daha da doldurur
   ↓
TÜM SİSTEM ÇÖKÜYOR
```

Bir downstream service yavaşladı → tüm upstream'ler patlar.

#### Retry storm

```
account-service 5xx döndü
transfer-service: "retry 3 kez"
3x request load → account-service daha yavaş
transfer-service: "retry 3 kez" (yeni request'ler)
9x load → death spiral
```

Naive retry hatayı katlar.

#### Timeout cascade

```
gateway timeout: 30s
transfer-service timeout: 30s
account-service timeout: 30s
DB timeout: 30s

Tam 30s'te DB timeout veriyor, ama:
- account-service hala bekliyor (timeout: 30s, başlangıç önce)
- transfer-service hala bekliyor
- gateway hala bekliyor
- User 30s+ bekliyor, sonra timeout
```

Timeout'lar aşağıdan yukarı **azalan** sırada olmalı (bulkhead pattern).

### 2. Resilience4j — 6 ana pattern

| Pattern | Amaç |
|---|---|
| **Circuit Breaker** | Backend down → fail-fast, recovery period |
| **Retry** | Transient hata → otomatik yeniden dene |
| **Bulkhead** | Resource isolation (thread/semaphore limit) |
| **TimeLimiter** | Max execution time, cancel |
| **RateLimiter** | Outbound request rate limit |
| **Fallback** | Failure'da graceful degradation |

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 3. Circuit Breaker — state machine

```
       failure rate < threshold
CLOSED ────────────────────────→ CLOSED
  │
  │ failure rate >= threshold
  ↓
OPEN (waitDuration)
  │
  │ wait duration sona erdi
  ↓
HALF_OPEN
  │
  ├─ trial request success → CLOSED
  └─ trial request fail → OPEN
```

**State davranışı:**

- **CLOSED:** Normal. Request'ler geçer. Failure rate izlenir.
- **OPEN:** Failure threshold aşıldı. Tüm request'ler **anında fail** (`CallNotPermittedException`). Backend'i koru.
- **HALF_OPEN:** Wait duration sonrası, **sınırlı trial request**. Success → CLOSED. Fail → OPEN.

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowType: COUNT_BASED   # veya TIME_BASED
        slidingWindowSize: 100            # son 100 call'a bak
        minimumNumberOfCalls: 10          # CB değerlendirmesi için min 10 call
        failureRateThreshold: 50          # %50 fail → OPEN
        slowCallRateThreshold: 100        # disabled
        slowCallDurationThreshold: 2s
        waitDurationInOpenState: 30s      # 30s OPEN, sonra HALF_OPEN
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - org.springframework.web.reactive.function.client.WebClientResponseException
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.mavibank.banking.account.exception.AccountNotFoundException
          - com.mavibank.banking.account.exception.InsufficientFundsException
    instances:
      accountService:
        baseConfig: default
      fraudService:
        baseConfig: default
        failureRateThreshold: 70   # daha gevşek
```

**Önemli:** `ignoreExceptions` — **business exception'lar** circuit breaker'ı triggerlamamalı.

`InsufficientFundsException` business hata değil teknik hata. Backend down değil, kullanıcının bakiyesi yetersiz. CB tetiklenirse her insufficient funds → backend down sayılır → sistem CB-open.

### 4. Banking örnek — TransferService → AccountService

```java
@Service
public class AccountServiceClient {
    
    private final WebClient accountServiceClient;
    
    @CircuitBreaker(name = "accountService", fallbackMethod = "fallbackGetBalance")
    public Mono<Money> getBalance(UUID accountId) {
        return accountServiceClient.get()
            .uri("/accounts/{id}/balance", accountId)
            .retrieve()
            .bodyToMono(Money.class);
    }
    
    public Mono<Money> fallbackGetBalance(UUID accountId, Throwable t) {
        log.warn("Account service unavailable for {}: {}", accountId, t.getMessage());
        
        // Banking: critical data — fail-fast, mock data DON'T
        return Mono.error(new ServiceUnavailableException(
            "Cannot retrieve balance — account service unavailable. Please retry."));
    }
    
    @CircuitBreaker(name = "accountService", fallbackMethod = "fallbackGetAccountInfo")
    public Mono<AccountInfo> getAccountInfo(UUID accountId) {
        return accountServiceClient.get()
            .uri("/accounts/{id}/info", accountId)
            .retrieve()
            .bodyToMono(AccountInfo.class);
    }
    
    // Non-critical: cached value OK
    public Mono<AccountInfo> fallbackGetAccountInfo(UUID accountId, Throwable t) {
        log.warn("Falling back to cached account info: {}", accountId);
        return Mono.justOrEmpty(accountInfoCache.get(accountId))
            .switchIfEmpty(Mono.error(new ServiceUnavailableException("No cached data")));
    }
}
```

**Banking pratiği:**
- **Critical path** (balance, debit) → fallback = fail-fast (mock data tehlikeli)
- **Non-critical** (display name, history) → cached value OK

### 5. Retry — transient hatalar için

```yaml
resilience4j:
  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.springframework.web.reactive.function.client.WebClientResponseException$ServiceUnavailable
          - org.springframework.web.reactive.function.client.WebClientResponseException$GatewayTimeout
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.mavibank.banking.account.exception.AccountNotFoundException
          - com.mavibank.banking.account.exception.InsufficientFundsException
          - com.mavibank.banking.common.exception.ValidationException
    instances:
      accountService:
        baseConfig: default
      fraudService:
        baseConfig: default
        maxAttempts: 2   # daha az retry (fraud non-blocking)
```

```java
@Service
public class AccountServiceClient {
    
    @CircuitBreaker(name = "accountService", fallbackMethod = "fallbackGetBalance")
    @Retry(name = "accountService")
    public Mono<Money> getBalance(UUID accountId) { ... }
}
```

**Retry kuralları:**
- `retryExceptions`: Sadece transient hatalar (5xx, network)
- `ignoreExceptions`: Business hatalar (4xx — bug fix değil retry)
- `enableExponentialBackoff`: 500ms, 1s, 2s — backoff strategy
- `maxAttempts`: 3-5 typical, banking için. Aşırı retry storm.

#### Retry only idempotent operations

**GET, HEAD, PUT, DELETE:** Idempotent, retry safe.

**POST:** Idempotent **DEĞİL** — retry duplicate yaratabilir.

Banking pattern: POST retry **sadece** `Idempotency-Key` ile (backend dedup).

```java
@Retry(name = "transferService")
public Mono<Transfer> postTransfer(TransferRequest req) {
    return transferServiceClient.post()
        .uri("/transfers")
        .header("Idempotency-Key", req.getIdempotencyKey().toString())   // dedup
        .bodyValue(req)
        .retrieve()
        .bodyToMono(Transfer.class);
}
```

### 6. Bulkhead — resource isolation

Pattern adı: gemi su geçirmez bölmelerden geliyor. Bir bölme delinse diğerleri ayakta kalır.

**Sorun:** Tek thread pool — bir slow service tüm thread'leri tüketir.

**Çözüm:** Service başına ayrı thread pool.

#### Semaphore Bulkhead — async/reactive için

```yaml
resilience4j:
  bulkhead:
    instances:
      accountService:
        maxConcurrentCalls: 50      # max 50 concurrent
        maxWaitDuration: 100ms       # 100ms wait, sonra fail
      fraudService:
        maxConcurrentCalls: 20
```

```java
@Bulkhead(name = "accountService", type = Bulkhead.Type.SEMAPHORE)
@CircuitBreaker(name = "accountService", fallbackMethod = "fallback")
public Mono<Money> getBalance(UUID id) { ... }
```

**Banking örnek:**
- account-service slow → max 50 concurrent → 51. request hemen fail (`BulkheadFullException`)
- fraud-service slow → ayrı bulkhead, account-service etkilenmedi

#### ThreadPool Bulkhead — blocking call için

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      legacyService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
        keepAliveDuration: 20ms
```

```java
@Bulkhead(name = "legacyService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<Response> callLegacy(Request req) { ... }
```

Banking için: Legacy CBS (blocking JDBC vs.) için ThreadPool bulkhead.

### 7. TimeLimiter — max execution time

```yaml
resilience4j:
  timelimiter:
    instances:
      accountService:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

```java
@TimeLimiter(name = "accountService")
@CircuitBreaker(name = "accountService", fallbackMethod = "fallback")
public CompletableFuture<Money> getBalance(UUID id) {
    return accountServiceClient.get()...toFuture();
}
```

Reactive version (Mono.timeout):

```java
public Mono<Money> getBalance(UUID id) {
    return accountServiceClient.get()
        .uri("/accounts/{id}/balance", id)
        .retrieve()
        .bodyToMono(Money.class)
        .timeout(Duration.ofSeconds(3));
}
```

**Banking timeout hierarchy** (önemli):

```
Client → Gateway:           60s (user max wait)
Gateway → transfer-service: 30s
transfer-service → account-service: 5s
account-service → DB:        3s
```

Her seviye **aşağıdaki**ndan **uzun**. Aksi halde:
- DB 4s'te döner, account-service zaten 3s'te timeout vermiş → exception
- Sonra gateway 30s'te bekler, transfer-service 5s'te döndü ama gateway hala bekliyor

Banking SLO'larla **bottom-up timeout** tasarımı.

### 8. RateLimiter — outbound rate limit

Inbound (gateway) rate limit Topic 7.3'te. **Outbound** rate limit:

Banking örnek: TCMB FX API günde max 1000 request.

```yaml
resilience4j:
  ratelimiter:
    instances:
      tcmbFxApi:
        limitForPeriod: 100         # 100 call
        limitRefreshPeriod: 60s     # her 60 saniyede
        timeoutDuration: 500ms      # 500ms wait, sonra fail
```

```java
@RateLimiter(name = "tcmbFxApi", fallbackMethod = "cachedFxRate")
public Mono<FxRate> getFxRate(Currency from, Currency to) {
    return tcmbClient.get()...
}

public Mono<FxRate> cachedFxRate(Currency from, Currency to, Throwable t) {
    return Mono.justOrEmpty(fxCache.get(from, to))
        .switchIfEmpty(Mono.error(new ServiceUnavailableException("No cached FX rate")));
}
```

**Banking pratiği:** External API quota'sını koru. Aşma → API key suspend riski.

### 9. Composition order — kritik

Birden fazla annotation birleştirilince **sıra önemli**:

```
[Retry (en dışta)]
  → [CircuitBreaker]
    → [TimeLimiter]
      → [Bulkhead]
        → [actual call]
```

```java
@Retry(name = "x")
@CircuitBreaker(name = "x", fallbackMethod = "fallback")
@TimeLimiter(name = "x")
@Bulkhead(name = "x")
public CompletableFuture<Account> getAccount(UUID id) { ... }
```

**Neden bu sıra:**

- **Retry dışta:** CB OPEN → retry no-op. CB CLOSED → fail → retry.
- **CB içte Bulkhead'in:** Bulkhead reject → CB sayar mı? Spring resilience4j config'e göre.
- **TimeLimiter Bulkhead'in dışında:** Timeout sayılmalı CB için.

**Yanlış sıra örnekleri:**

```java
@CircuitBreaker
@Retry           // ❌ CB başlatmadan retry, CB hiç tetiklenmez
public ...
```

### 10. Fallback strategies — banking

**Strategy 1: Fail-fast (critical path)**

```java
public Mono<Money> fallbackGetBalance(UUID id, Throwable t) {
    return Mono.error(new ServiceUnavailableException("..."));
}
```

Banking transfer'da balance bilmek zorunda. Mock data = yanlış kararlar.

**Strategy 2: Cached value (non-critical)**

```java
public Mono<AccountInfo> fallbackGetAccountInfo(UUID id, Throwable t) {
    return Mono.justOrEmpty(cache.get(id))
        .switchIfEmpty(Mono.just(AccountInfo.unknown()));
}
```

Display name için cached value OK. Stale data tolere edilir.

**Strategy 3: Queue and retry later**

```java
public Mono<Void> fallbackSendNotification(NotificationRequest req, Throwable t) {
    pendingNotificationRepo.save(new PendingNotification(req));
    return Mono.empty();
}
```

Notification queue'ya at, scheduled job sonradan retry.

**Strategy 4: Degrade functionality**

```java
public Mono<FraudScore> fallbackFraudCheck(TransferRequest req, Throwable t) {
    log.warn("Fraud service down — applying conservative default");
    if (req.getAmount().compareTo(LARGE_AMOUNT_THRESHOLD) > 0) {
        return Mono.just(FraudScore.HIGH_RISK);   // safe default
    }
    return Mono.just(FraudScore.UNKNOWN_BUT_PROCEED);
}
```

Banking — fraud service down → büyük amount'ları HIGH_RISK varsay (conservative).

### 11. Bulk operation patterns

Cross-service bulk operations için:

```java
public Mono<List<AccountInfo>> getAccountsInfo(List<UUID> accountIds) {
    return Flux.fromIterable(accountIds)
        .flatMap(this::getAccountInfo, 10)   // max 10 concurrent
        .collectList()
        .timeout(Duration.ofSeconds(30));
}
```

`flatMap(_, concurrency)` — paralel ama bounded. Bulkhead pattern reactive.

### 12. Banking örnek — full TransferService

```java
@Service
public class TransferService {
    
    private final AccountServiceClient accountClient;
    private final FraudServiceClient fraudClient;
    private final NotificationServiceClient notificationClient;
    
    @Transactional
    public Mono<Transfer> execute(TransferRequest req, UUID userId) {
        return validateRequest(req)
            .flatMap(v -> fraudClient.scoreTransfer(req, userId))
            .filter(score -> !score.isHighRisk())
            .switchIfEmpty(Mono.error(new HighRiskTransferException()))
            .flatMap(score -> accountClient.debit(req.fromAccountId(), req.amount(), userId))
            .flatMap(debited -> accountClient.credit(req.toAccountId(), req.amount(), userId))
            .flatMap(credited -> saveTransfer(req, userId))
            .flatMap(transfer -> {
                // Notification — best effort
                return notificationClient.send(NotificationRequest.from(transfer))
                    .timeout(Duration.ofSeconds(2))
                    .onErrorResume(e -> {
                        log.warn("Notification failed, will retry async", e);
                        pendingNotificationRepo.save(new PendingNotification(transfer));
                        return Mono.empty();
                    })
                    .thenReturn(transfer);
            });
    }
}

@Service
public class AccountServiceClient {
    
    @CircuitBreaker(name = "accountService", fallbackMethod = "fallbackDebit")
    @Retry(name = "accountService")
    @Bulkhead(name = "accountService", type = Bulkhead.Type.SEMAPHORE)
    public Mono<Account> debit(UUID accountId, Money amount, String userToken) {
        return accountServiceClient.post()
            .uri("/accounts/{id}/debit", accountId)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + userToken)
            .header("Idempotency-Key", UUID.randomUUID().toString())
            .bodyValue(new DebitRequest(amount))
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, response -> 
                Mono.error(new ClientErrorException(response.statusCode().value())))
            .bodyToMono(Account.class)
            .timeout(Duration.ofSeconds(5));
    }
    
    public Mono<Account> fallbackDebit(UUID accountId, Money amount, String userToken, Throwable t) {
        if (t instanceof CallNotPermittedException) {
            log.error("Account service circuit OPEN, cannot debit: {}", accountId);
        }
        // Banking: critical path, no mock fallback
        return Mono.error(new ServiceUnavailableException("Account service unavailable"));
    }
}

@Service
public class FraudServiceClient {
    
    @CircuitBreaker(name = "fraudService", fallbackMethod = "fallbackScoreTransfer")
    @Retry(name = "fraudService")
    @TimeLimiter(name = "fraudService")
    public Mono<FraudScore> scoreTransfer(TransferRequest req, String userToken) {
        return fraudServiceClient.post()
            .uri("/score")
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + userToken)
            .bodyValue(req)
            .retrieve()
            .bodyToMono(FraudScore.class);
    }
    
    public Mono<FraudScore> fallbackScoreTransfer(TransferRequest req, String userToken, Throwable t) {
        log.warn("Fraud service unavailable — conservative default");
        
        if (req.getAmount().compareTo(new BigDecimal("10000")) > 0) {
            return Mono.just(FraudScore.HIGH_RISK);
        }
        return Mono.just(FraudScore.PROCEED_WITH_CAUTION);
    }
}
```

### 13. Metrics + monitoring

Resilience4j Micrometer ile entegre:

```yaml
management:
  metrics:
    tags:
      application: transfer-service
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus, circuitbreakers
```

Otomatik metric'ler:

- `resilience4j_circuitbreaker_state` (gauge)
- `resilience4j_circuitbreaker_calls` (counter, with tags: kind=successful/failed/ignored)
- `resilience4j_circuitbreaker_failure_rate` (gauge)
- `resilience4j_retry_calls` (counter, with: kind=successful_with_retry/successful_without_retry/failed_with_retry)
- `resilience4j_bulkhead_available_concurrent_calls` (gauge)
- `resilience4j_ratelimiter_available_permissions` (gauge)

Grafana dashboard:
- CB state (OPEN/CLOSED) over time
- Failure rate per service
- Retry count
- Bulkhead saturation

### 14. Event listeners

```java
@Configuration
public class CircuitBreakerEventConfig {
    
    @Autowired CircuitBreakerRegistry registry;
    @Autowired NotificationService notifier;
    
    @PostConstruct
    public void setup() {
        registry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher()
                .onStateTransition(event -> {
                    log.warn("Circuit breaker {} state changed: {} → {}",
                        cb.getName(), 
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState());
                    
                    if (event.getStateTransition().getToState() == State.OPEN) {
                        notifier.alertOps("Circuit breaker OPEN: " + cb.getName());
                    }
                });
        });
    }
}
```

Banking pratiği: CB state transition → ops alert. SRE pager.

### 15. Health indicator

Circuit breaker open → service "DOWN":

```yaml
management:
  health:
    circuitbreakers:
      enabled: true
```

`/actuator/health`:

```json
{
  "status": "UP",
  "components": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "accountService": { "state": "CLOSED", "failureRate": "5.0%" },
        "fraudService": { "state": "OPEN", "failureRate": "75.0%" }   // health DOWN olabilir
      }
    }
  }
}
```

K8s readiness probe etkilenir — instance traffic'ten çıkarılabilir.

### 16. Banking anti-pattern'leri

**Anti-pattern 1: Aggressive retry → DDoS kendi DB'ne**

```yaml
retry:
  maxAttempts: 10
  waitDuration: 0ms   # ❌
```

10x request rate, no backoff → backend ezildi. **3 attempt + exponential backoff** standart.

**Anti-pattern 2: Generic exception retry**

```yaml
retryExceptions:
  - Exception   # ❌ everything retried
```

Business exception (InsufficientFunds, AccountNotFound) retry **anlamsız** — sonuç değişmez.

**Anti-pattern 3: Fallback mock data critical path'te**

```java
public Mono<Money> fallbackGetBalance(UUID id, Throwable t) {
    return Mono.just(Money.of("0", "TRY"));   // ❌ yanlış balance → yanlış decisions
}
```

Banking transfer'da fallback mock = yanlış kararlar = customer kaybı.

**Anti-pattern 4: CB olmadan downstream call**

```java
@Service
public class AccountServiceClient {
    public Mono<Money> getBalance(UUID id) {   // ❌ no CB
        return webClient.get()...
    }
}
```

Backend slow → tüm thread'ler tükenir → cascading failure.

**Anti-pattern 5: Bulkhead yok, shared pool**

Tek slow service tüm app'ı yavaşlatır. Bulkhead per-service mandatory.

**Anti-pattern 6: TimeLimiter cascade yok**

DB → service → gateway timeout hep 30s. DB 35s'te döner → mantıksız layered timeout.

Doğrusu: DB 3s, service 5s, gateway 30s. Bottom-up.

**Anti-pattern 7: Business exception CB tetikler**

```yaml
recordExceptions:
  - InsufficientFundsException   # ❌
```

Insufficient funds backend down değil. CB OPEN olmamalı.

`ignoreExceptions` listesinde olmalı.

**Anti-pattern 8: Retry ile non-idempotent POST**

POST /transfers retry → duplicate transfer (Idempotency-Key olmadan).

**Çözüm:** POST retry **sadece** Idempotency-Key ile + backend dedup.

---

## Önemli olabilecek araştırma kaynakları

- Resilience4j documentation (current 2.x)
- "Release It!" (Michael Nygard) — stability patterns
- Spring Cloud Circuit Breaker
- Hystrix (legacy) vs Resilience4j karşılaştırması
- Netflix tech blog — circuit breaker original

---

## Mini task'ler

### Task 7.5.1 — Resilience4j setup (30 dk)

`pom.xml`, application.yml config. 4 instance: accountService, fraudService, notificationService, tcmbFxApi.

### Task 7.5.2 — Circuit Breaker + fallback (60 dk)

`AccountServiceClient.getBalance` ile CB + fallback. Test:
- Backend up → normal
- Backend down → 100 fail sonra CB OPEN
- CB OPEN → instant fallback (latency düşer)
- Wait duration sonrası → HALF_OPEN → trial request
- Trial success → CLOSED

### Task 7.5.3 — Retry exponential backoff (45 dk)

`@Retry` ile 3 attempt + 500ms, 1s, 2s backoff. Test:
- Transient error (5xx) → 3 retry → fail
- Business error (4xx) → 0 retry → fail immediately
- 1. retry success → 2. attempt'ta dönüyor

### Task 7.5.4 — Bulkhead semaphore (45 dk)

`maxConcurrentCalls: 50`. 100 concurrent request → 50 başarılı, 50 BulkheadFullException.

Test: 100 paralel request, count success/fail.

### Task 7.5.5 — TimeLimiter (30 dk)

`timeoutDuration: 3s`. Backend kasten 5s sleep → timeout → fallback.

### Task 7.5.6 — RateLimiter outbound (30 dk)

TCMB FX API mock — 100 req/min limit. Aşımda cached value fallback.

### Task 7.5.7 — Multi-annotation composition (60 dk)

```java
@Retry(name = "x")
@CircuitBreaker(name = "x", fallbackMethod = "fallback")
@TimeLimiter(name = "x")
@Bulkhead(name = "x")
public CompletableFuture<Account> getAccount(UUID id) { ... }
```

Tüm pattern'leri birleştir. Test edge case'ler.

### Task 7.5.8 — Event listener + alert (30 dk)

CB state transition listener. OPEN'a geçtiğinde Slack alert.

### Task 7.5.9 — Grafana dashboard (45 dk)

Prometheus + Grafana docker-compose. Resilience4j metrics scrape. Dashboard:
- CB state per service (gauge)
- Failure rate (gauge)
- Retry count (counter)
- Bulkhead saturation (gauge)

---

## Test yazma rehberi

```java
@SpringBootTest
class AccountServiceClientResilienceTest {
    
    @Autowired AccountServiceClient client;
    @Autowired CircuitBreakerRegistry cbRegistry;
    
    @MockBean WebClient.Builder webClientBuilder;
    
    @Test
    void shouldOpenCircuitBreakerAfterFailures() {
        when(webClient.get()).thenThrow(new RuntimeException("Service down"));
        
        for (int i = 0; i < 100; i++) {
            assertThatThrownBy(() -> client.getBalance(UUID.randomUUID()).block());
        }
        
        CircuitBreaker cb = cbRegistry.circuitBreaker("accountService");
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }
    
    @Test
    void shouldNotTriggerCBOnBusinessException() {
        when(webClient.get()).thenReturn(Mono.error(new InsufficientFundsException()));
        
        for (int i = 0; i < 100; i++) {
            assertThatThrownBy(() -> client.getBalance(UUID.randomUUID()).block());
        }
        
        CircuitBreaker cb = cbRegistry.circuitBreaker("accountService");
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.CLOSED);   // ignoreExceptions list'te
    }
    
    @Test
    void shouldRetryTransientErrorsThenFail() {
        AtomicInteger calls = new AtomicInteger();
        when(webClient.get()).thenAnswer(inv -> {
            calls.incrementAndGet();
            return Mono.error(new WebClientResponseException(503, "Service Unavailable", null, null, null));
        });
        
        assertThatThrownBy(() -> client.getBalance(UUID.randomUUID()).block());
        
        assertThat(calls.get()).isEqualTo(3);   // 3 retry attempts
    }
    
    @Test
    void shouldFallbackWhenCBOpens() {
        // Force CB to OPEN
        CircuitBreaker cb = cbRegistry.circuitBreaker("accountService");
        cb.transitionToOpenState();
        
        Mono<Money> result = client.getBalance(UUID.randomUUID());
        
        StepVerifier.create(result)
            .expectError(ServiceUnavailableException.class)
            .verify();
    }
}
```

---

## Claude-verify prompt

```
Resilience4j implementation'ımı banking-grade kriterlere göre değerlendir:

1. Circuit Breaker:
   - Her external call'da var mı?
   - failureRateThreshold makul (50-70%)?
   - waitDurationInOpenState (15-60s)?
   - ignoreExceptions banking business exception'ları (InsufficientFunds, vb.)?
   - Health indicator + state transition alert?

2. Retry:
   - 3-5 maxAttempts (10+ DEĞİL)?
   - Exponential backoff?
   - retryExceptions sadece transient (5xx, network)?
   - ignoreExceptions business hatalar?
   - POST retry sadece Idempotency-Key ile?

3. Bulkhead:
   - Her service için ayrı (shared pool DEĞİL)?
   - maxConcurrentCalls realistic?
   - Semaphore vs ThreadPool karar gerekçeli?

4. TimeLimiter:
   - Banking timeout hierarchy (DB < service < gateway)?
   - Reactive Mono.timeout veya @TimeLimiter?

5. RateLimiter:
   - External API quota koruma (TCMB, Visa)?
   - Cached fallback rate limit aşımında?

6. Composition order:
   - Retry → CB → TimeLimiter → Bulkhead → call (en dıştan içe)?
   - @Retry @CircuitBreaker @TimeLimiter @Bulkhead sıra doğru?

7. Fallback strategies:
   - Critical path (balance) → fail-fast (no mock)?
   - Non-critical (display name) → cached value?
   - Fraud service down → conservative default (HIGH_RISK)?
   - Notification down → queue + retry async?

8. Metrics + monitoring:
   - Micrometer + Prometheus integration?
   - CB state Grafana dashboard?
   - Event listener + Slack alert on OPEN?

9. Banking-specific:
   - Critical path identify edilmiş (transfer balance check)?
   - Non-critical degradation tolerable?
   - Conservative defaults (high amount → HIGH_RISK)?

10. Anti-pattern:
    - Aggressive retry (10+)?
    - Generic exception retry?
    - Mock data fallback critical path'te?
    - CB olmadan downstream call?
    - Business exception CB triggerlıyor?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] 4 instance config (accountService, fraudService, notificationService, tcmbFxApi)
- [ ] CircuitBreaker + fallback her external call'da
- [ ] Retry + exponential backoff + ignoreExceptions
- [ ] Bulkhead semaphore per service
- [ ] TimeLimiter banking hierarchy
- [ ] RateLimiter outbound (TCMB)
- [ ] Composition order doğru (Retry → CB → TL → Bulkhead)
- [ ] Fallback strategies 4'ü (fail-fast, cached, queue, degraded)
- [ ] Event listener + alert
- [ ] Grafana dashboard

---

## Defter notları (10 madde)

1. "Cascading failure ve retry storm distributed sistem patolojileri: ____."
2. "CircuitBreaker state machine (CLOSED → OPEN → HALF_OPEN): ____."
3. "recordExceptions vs ignoreExceptions banking için karar: ____."
4. "Retry exponential backoff banking için neden gerekli: ____."
5. "Bulkhead semaphore vs ThreadPool farkı: ____."
6. "Banking timeout hierarchy (DB → service → gateway): ____."
7. "RateLimiter external API quota koruma (TCMB örneği): ____."
8. "Composition order (Retry dışta) — neden: ____."
9. "Fallback strategy decision (critical vs non-critical): ____."
10. "Anti-pattern: business exception CB triggerlıyor — neden yanlış: ____."
