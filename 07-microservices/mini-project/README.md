# Phase 7 Mini-Project — Microservices Split & Resilience

## Hedef

`core-banking` monolitini **production-grade 4 microservice'e** böl. API Gateway + Service Discovery + Resilience4j + OpenTelemetry tracing + Saga + TCC pattern. Her bileşen banking-grade:
- mTLS service-to-service
- JWT propagation
- Circuit breaker + fallback per backend
- End-to-end trace Jaeger'da
- Cross-bank Saga + TCC

Sonunda elinde: TR bank tech ekibinin **referans implementation** olarak alabileceği microservice platform.

## Süre

10-15 gün (günde 2-3 saat).

## Önbilgi

Phase 7'nin 7 topic'i bitti. Phase 1-6'da `core-banking` monolit'i + outbox + Kafka hazır.

---

## Görev 1 — Maven multi-module split (1 gün)

### 1.1 Proje yapısı

```
core-banking-parent/
├── pom.xml (parent POM)
├── banking-commons/
│   └── src/main/java/.../{Money, Currency, AccountId, OwnerId, TransferId, ...}
├── banking-events/
│   └── src/main/java/.../{TransferCompletedEvent, AuditLogEvent, ...}
├── account-service/
│   ├── pom.xml (depends on banking-commons)
│   ├── src/
│   └── Dockerfile
├── transfer-service/
│   ├── pom.xml (depends on banking-commons, banking-events)
│   ├── src/
│   └── Dockerfile
├── fraud-service/
├── notification-service/
├── api-gateway/
├── eureka-server/ (optional)
├── docker-compose.yml
└── k8s/ (manifests)
```

### 1.2 Parent POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.mavibank</groupId>
    <artifactId>core-banking-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    
    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2024.0.0</spring-cloud.version>
        <resilience4j.version>2.2.0</resilience4j.version>
    </properties>
    
    <modules>
        <module>banking-commons</module>
        <module>banking-events</module>
        <module>account-service</module>
        <module>transfer-service</module>
        <module>fraud-service</module>
        <module>notification-service</module>
        <module>api-gateway</module>
    </modules>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### 1.3 banking-commons

`Money`, `Currency`, `AccountId`, `OwnerId`, `TransferId` (Phase 1'den).

```xml
<artifactId>banking-commons</artifactId>
<dependencies>
    <!-- Sadece çok minimal — Phase 1 value objects -->
</dependencies>
```

### 1.4 banking-events

Avro schema'lar veya POJO event class'ları.

```
banking-events/src/main/avro/
├── TransferCompletedEvent.avsc
├── AuditLogEvent.avsc
├── NotificationRequest.avsc
└── FraudAlert.avsc
```

`xjc` ile Java class generate.

### 1.5 Code migration

Mevcut `core-banking` monolit'inden 4 servise dağıt:

```
Monolith                          →  Target service
─────────────────────────────────────────────────────
AccountController                 →  account-service
AccountService (use case)         →  account-service
JpaAccountRepository              →  account-service
AccountJpaEntity                  →  account-service
JournalEntry/Line                 →  account-service
Phase 4 PL/SQL packages           →  account-service
                                     (interest_pkg, eod_reconciliation_pkg)

TransferController                →  transfer-service
TransferService                   →  transfer-service
Phase 6 Saga (CrossBankTransferSaga) → transfer-service
OutboxPublisher (Phase 6)         →  transfer-service
IdempotencyKey handling           →  transfer-service

FraudDetectionTopology            →  fraud-service
Phase 4 fraud_check_pkg           →  fraud-service
FraudAlert                        →  fraud-service

NotificationConsumer (Phase 6)    →  notification-service
SMS/Email gateway client          →  notification-service

ApiGatewayApplication             →  api-gateway (new)
```

### Deliverables

- [ ] Multi-module Maven structure
- [ ] 5 service module (account, transfer, fraud, notification, gateway)
- [ ] 2 library module (banking-commons, banking-events)
- [ ] Mevcut monolit code 4 service'e taşındı
- [ ] `mvn install` çalışıyor
- [ ] Her servis kendi `application.yml`'inde
- [ ] Defterimde: hangi sınıf hangi servise gitti, gerekçe

---

## Görev 2 — Database per service (1 gün)

### 2.1 PostgreSQL schema split

```sql
-- Master DB
CREATE DATABASE banking;

\c banking

CREATE SCHEMA account_db;
CREATE SCHEMA transfer_db;
CREATE SCHEMA fraud_db;
CREATE SCHEMA notification_db;

-- Yeni user (servis başına) — RLS/role isolation
CREATE USER account_service WITH PASSWORD '...';
GRANT USAGE ON SCHEMA account_db TO account_service;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA account_db TO account_service;
-- vs.
```

Production: tamamen ayrı DB cluster'lar (account-db, transfer-db, vb.). Lokal dev için schema isolation yeterli.

### 2.2 Migration split

Mevcut `db/migration/` klasörünü servis bazlı böl:

```
account-service/src/main/resources/db/migration/
├── V1__create_accounts_table.sql
├── V2__create_journal_tables.sql
└── R__create_balance_summary_view.sql

transfer-service/src/main/resources/db/migration/
├── V1__create_transfers_table.sql
├── V2__create_idempotency_keys.sql
├── V3__create_outbox_events.sql
└── V4__create_saga_states.sql

fraud-service/src/main/resources/db/migration/
├── V1__create_fraud_rules.sql
├── V2__create_fraud_alerts.sql

notification-service/src/main/resources/db/migration/
├── V1__create_notification_log.sql
├── V2__create_processed_events.sql
```

Her servis kendi Flyway'i ile kendi schema'sını migrate eder.

### 2.3 ArchUnit cross-service dependency rule

```java
@Test
void servicesShouldNotShareData() {
    // account-service entity'leri transfer-service'te kullanılmamalı
    noClasses().that().resideInAPackage("com.mavibank.transfer..")
        .should().dependOnClassesThat().resideInAPackage("com.mavibank.account.adapter.out.persistence..")
        .check(classes);
}
```

### Deliverables

- [ ] 4 schema (account_db, transfer_db, fraud_db, notification_db)
- [ ] Migration'lar her servis kendi içinde
- [ ] Cross-service entity import YOK (ArchUnit kanıtı)
- [ ] Her servis kendi DB user'ı (production-like isolation)

---

## Görev 3 — API Gateway (1.5 gün)

### 3.1 Spring Cloud Gateway setup

Topic 7.3'teki implementation. 5 route (account, transfer, auth, internal, admin).

### 3.2 JwtAuthenticationFilter

JWT validate + X-User-Id, X-Tenant-Id, X-User-Roles header propagate.

### 3.3 Redis rate limiter

Per-user (10 req/sec) + per-IP (5 req/min anonymous) + per-tenant.

### 3.4 Circuit breaker + fallback

Her backend için CB. Fallback URI ProblemDetail 503.

### 3.5 IdempotencyKeyFilter (banking)

POST/PUT için Idempotency-Key header zorunlu.

### 3.6 OpenTelemetry tracing

GlobalFilter + Micrometer Tracing.

### Deliverables

- [ ] Gateway Spring Boot project (Netty reactive)
- [ ] 5 route YAML
- [ ] JWT filter + header propagation
- [ ] Redis rate limiter + per-user + per-IP
- [ ] CB + fallback (her backend için)
- [ ] IdempotencyKeyFilter
- [ ] LoggingFilter + X-Request-Id
- [ ] CORS banking domains
- [ ] 8+ integration test (WebTestClient)

---

## Görev 4 — Service Discovery (1 gün)

### 4.1 Eureka veya K8s Service

Banking için K8s native tercih. minikube veya kind ile local cluster.

`account-service/k8s/deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: account-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: banking/account-service:1.0.0
          ports:
            - containerPort: 8081

---
apiVersion: v1
kind: Service
metadata:
  name: account-service
spec:
  selector:
    app: account-service
  ports:
    - port: 8081
      targetPort: 8081
```

### 4.2 Health endpoint + probes

Spring Boot Actuator. K8s liveness + readiness + startup probes.

### 4.3 Service-to-service WebClient

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

@Bean
public WebClient accountServiceClient(WebClient.Builder builder) {
    return builder.baseUrl("http://account-service:8081").build();
}
```

K8s DNS resolve: `account-service` → cluster IP → pod.

### Deliverables

- [ ] K8s manifests (Deployment + Service)
- [ ] Spring Boot Actuator health + probes
- [ ] Service-to-service WebClient
- [ ] minikube'da deploy + test
- [ ] Cross-service ping (transfer → account)

---

## Görev 5 — Resilience4j integration (1.5 gün)

### 5.1 Per-backend CB config

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
    instances:
      accountService:
        baseConfig: default
        ignoreExceptions:
          - com.mavibank.banking.account.exception.AccountNotFoundException
          - com.mavibank.banking.account.exception.InsufficientFundsException
      fraudService:
        baseConfig: default
        failureRateThreshold: 70   # daha gevşek
```

### 5.2 Retry per backend

Transient exception only. Business exception ignore. Idempotent operation retry.

### 5.3 Bulkhead semaphore

Her backend için maxConcurrentCalls.

### 5.4 TimeLimiter banking hierarchy

```
Client → Gateway: 60s
Gateway → service: 30s
service → service: 5s
service → DB: 3s
```

Bottom-up timeout.

### 5.5 RateLimiter outbound

TCMB FX API: 100 req/min (mock).

### 5.6 Fallback strategies

- Critical (balance): fail-fast
- Non-critical (display): cached
- Fraud: conservative default (HIGH_RISK)
- Notification: queue + async retry

### Deliverables

- [ ] 4 backend için CB + Retry + Bulkhead + TimeLimiter
- [ ] Composition order doğru (Retry → CB → TL → Bulkhead)
- [ ] Banking timeout hierarchy
- [ ] Fallback strategies 4'ü implement
- [ ] Event listener + Slack alert on CB OPEN
- [ ] Grafana dashboard Resilience4j metrics

---

## Görev 6 — OpenTelemetry tracing (1 gün)

### 6.1 Jaeger docker-compose

```yaml
jaeger:
  image: jaegertracing/all-in-one:1.55
  environment:
    COLLECTOR_OTLP_ENABLED: "true"
  ports:
    - "16686:16686"
    - "4318:4318"
```

### 6.2 Tüm servislerde Micrometer Tracing

```yaml
management:
  tracing:
    sampling.probability: 1.0   # dev
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
```

### 6.3 End-to-end transfer trace

User POST → Gateway → transfer-service → account-service (debit + credit) → outbox → notification-service (consume).

Jaeger UI'da **tek trace** olarak görünmeli (Kafka propagation dahil).

### 6.4 Custom span — business critical

```java
@Observed(name = "transfer.execute")
public Mono<Transfer> execute(TransferRequest req) { ... }

@Observed(name = "fraud.score")
public Mono<FraudScore> scoreTransfer(TransferRequest req) { ... }
```

### Deliverables

- [ ] Jaeger docker-compose
- [ ] 4 servis OTLP tracing enabled
- [ ] End-to-end transfer trace UI'da görünür
- [ ] Custom @Observed business operations
- [ ] Kafka trace propagation
- [ ] Log MDC traceId entegrasyonu

---

## Görev 7 — Saga + TCC cross-bank (2 gün)

### 7.1 Phase 6 Topic 6.7 + Phase 7 Topic 7.7 birleştir

Phase 6'da yazdığın `CrossBankTransferSaga` orchestrator'a TCC pattern uygula:

```java
public class CrossBankTransferSaga {
    
    public UUID initiate(CrossBankTransferRequest req) {
        UUID sagaId = UUID.randomUUID();
        SagaState saga = sagaRepo.save(SagaState.init(sagaId, req));
        
        // Phase 1: Try (TCC)
        // Bank A hold
        kafka.send("bank-a.try-hold", sagaId.toString(),
            new TryHoldRequest(sagaId, req.fromAccountId(), req.amount()));
        
        return sagaId;
    }
    
    @KafkaListener(topics = "bank-a.try-hold.success")
    public void onBankAHoldSuccess(BankAHoldSuccessEvent event) {
        SagaState saga = sagaRepo.findById(event.getSagaId()).orElseThrow();
        saga.recordTry("BANK_A_HOLD", event.getHoldId());
        
        // Phase 1 cont: FX rate + Bank B reserve
        kafka.send("fx.try-quote", saga.getSagaId().toString(),
            new TryFxQuoteRequest(saga.getSagaId(), ...));
    }
    
    // ... benzer event handler'lar
    
    @KafkaListener(topics = "bank-b.reserve.success")
    public void onAllTrySuccess(BankBReserveSuccessEvent event) {
        SagaState saga = sagaRepo.findById(event.getSagaId()).orElseThrow();
        saga.recordTry("BANK_B_RESERVE", event.getReserveId());
        
        // Phase 2: Confirm all
        kafka.send("bank-a.confirm-hold", saga.getSagaId().toString(),
            new ConfirmHoldRequest(saga.getResourceId("BANK_A_HOLD")));
        kafka.send("bank-b.confirm-reserve", saga.getSagaId().toString(),
            new ConfirmReserveRequest(saga.getResourceId("BANK_B_RESERVE")));
        
        saga.setCurrentState("CONFIRMING");
        sagaRepo.save(saga);
    }
    
    @KafkaListener(topics = "saga.any-try.failure")
    public void onAnyTryFailure(TryFailureEvent event) {
        SagaState saga = sagaRepo.findById(event.getSagaId()).orElseThrow();
        saga.setCurrentState("COMPENSATING");
        sagaRepo.save(saga);
        
        // Phase 3: Cancel all reservations
        saga.getTryRecords().forEach(record -> {
            kafka.send(record.getCancelTopic(), saga.getSagaId().toString(),
                new CancelRequest(record.getResourceId()));
        });
    }
}
```

### 7.2 Mock external bank services

`bank-a-service` ve `bank-b-service` mock Spring Boot apps:

```java
@RestController
@RequestMapping("/internal/bank-a")
public class BankAController {
    
    @PostMapping("/try-hold")
    public Mono<HoldResponse> tryHold(@RequestBody HoldRequest req) {
        // TCC try implementation
    }
    
    @PostMapping("/confirm-hold")
    public Mono<Void> confirmHold(@RequestBody ConfirmRequest req) { ... }
    
    @PostMapping("/cancel-hold")
    public Mono<Void> cancelHold(@RequestBody CancelRequest req) { ... }
}
```

### 7.3 Stuck saga recovery

Scheduled task — 5 dakika eski PENDING saga → compensation trigger.

### Deliverables

- [ ] CrossBankTransferSaga orchestrator + TCC pattern
- [ ] Mock bank-a-service + bank-b-service
- [ ] Happy path test geçiyor
- [ ] Compensation test geçiyor (Bank B reserve fail → Bank A cancel)
- [ ] Stuck saga recovery scheduler
- [ ] Async API + status polling

---

## Görev 8 — Kasten kırma reproduction (1 gün)

5 senaryo:

### 8.1 Network latency simulation (Toxiproxy)

```yaml
toxiproxy:
  image: ghcr.io/shopify/toxiproxy:2.7.0
  ports:
    - "8474:8474"

# Proxy account-service through Toxiproxy
account-service-proxy: 8881 (toxiproxy port)
account-service-real: 8081
```

Transfer-service → account-service-proxy (8881). Toxiproxy:
- latency 5000ms (timeout aşımı)
- bandwidth limit
- connection drop

Test: CB OPEN'a düşmeli, fallback tetiklenmeli.

### 8.2 Backend kasten patlat

account-service crash. CB OPEN sonra HALF_OPEN sonra CLOSED (recovery).

### 8.3 Saga compensation test

Bank B reserve fail → Bank A hold cancel olmalı. Idempotency test: aynı cancel 2 kez → tek effect.

### 8.4 Lock split-brain simulation

Redis cluster 2-node partition → her partition kendi lock veriyor mu? (Banking için bu yüzden critical op DB-level lock.)

### 8.5 Distributed trace gaps

Async chain'de bir hop trace context propagate etmiyorsa nasıl görünür? Manuel context propagation'ı KAPAT, gör.

### Deliverables

- [ ] 5 kasten kırma senaryosu reproduced + fixed
- [ ] Toxiproxy network simulation
- [ ] Defterimde her senaryo için before/after notu

---

## Görev 9 — Integration test suite (1 gün)

`@SpringBootTest + TestContainers + ContractTest`:

**Per-service unit tests:** Phase 1-6'dan korumalı.

**Integration tests:**
- API Gateway → backend routing test
- Service-to-service Resilience4j test
- End-to-end trace test (Jaeger query)
- Saga happy path
- Saga compensation
- TCC try-confirm-cancel
- Idempotency cross-service
- Multi-instance ShedLock

**Contract tests (Spring Cloud Contract — Phase 12'de detay):**
- transfer-service consumer ↔ account-service provider
- fraud-service contract

Toplam 25+ integration test.

---

## Görev 10 — Observability stack (yarım gün)

docker-compose ile:

- Prometheus
- Grafana
- Jaeger
- Loki (log aggregation)
- AlertManager

Dashboard:
- RED method per service (Rate, Errors, Duration)
- Circuit breaker state per backend
- Saga success rate + p99 duration
- Rate limit 429 count
- Kafka producer/consumer lag

Alert:
- CB OPEN → Slack
- p99 > 500ms → Slack
- Error rate > 1% → PagerDuty

---

## Acceptance criteria

- [ ] 4 microservice ayrı Maven module
- [ ] Database per service (4 schema)
- [ ] API Gateway + 5 route + JWT + rate limit + CB + IdempotencyKey
- [ ] K8s Service discovery (veya Eureka)
- [ ] Resilience4j: CB + Retry + Bulkhead + TimeLimiter + RateLimiter + Fallback per backend
- [ ] OpenTelemetry end-to-end trace Jaeger'da görünür
- [ ] Saga + TCC cross-bank transfer (Bank A hold → Bank B reserve → Confirm/Cancel)
- [ ] Outbox pattern (Phase 6) tüm servislerde
- [ ] Idempotency cross-service
- [ ] 5 kasten kırma senaryosu reproduced + fixed
- [ ] 25+ integration test passing
- [ ] Observability stack (Prometheus + Grafana + Jaeger + Loki)
- [ ] CB OPEN alert + p99 alert
- [ ] CI/CD pipeline 4 service için ayrı build

---

## Claude-verify prompt

```
Phase 7 microservices mini-project'imi banking-grade kriterlere göre değerlendir 
kapsamlı olarak. Eksiklerimi söyle, kanıt göster:

A. Architecture
   1. Bounded context'ler net (4 servis)?
   2. Database per service strict (shared DB anti-pattern yok)?
   3. ArchUnit cross-service dependency tests?

B. API Gateway
   1. Spring Cloud Gateway reactive (Netty)?
   2. JWT filter + header propagation?
   3. Redis rate limiter per-user + per-IP + per-tenant?
   4. CB + fallback per backend?
   5. IdempotencyKey filter banking?

C. Service Discovery
   1. K8s Service veya Eureka cluster?
   2. Spring Boot Actuator health endpoints?
   3. Service-to-service @LoadBalanced WebClient?

D. Resilience4j
   1. CB her external call'da + ignoreExceptions business?
   2. Retry transient only + Idempotency-Key for POST?
   3. Bulkhead per service?
   4. TimeLimiter banking hierarchy (DB → service → gateway)?
   5. RateLimiter outbound (TCMB)?
   6. Composition order (Retry → CB → TL → Bulkhead)?
   7. Fallback strategies banking-appropriate?

E. Distributed Tracing
   1. OpenTelemetry + OTLP exporter?
   2. Auto-instrumentation HTTP + DB + Kafka?
   3. Cross-service trace propagation (Kafka dahil)?
   4. Custom @Observed business operations?
   5. PII span attribute'larda YOK?
   6. Sampling production-appropriate?

F. Saga + TCC
   1. CrossBankTransferSaga orchestrator + TCC?
   2. Happy path + compensation test?
   3. Stuck saga recovery scheduler?
   4. Compensation idempotent?
   5. Async API (202 + status polling)?

G. Banking-specific
   1. PII cross-service propagation YOK (sadece internal ID)?
   2. mTLS service-to-service?
   3. Outbox pattern (Phase 6 entegre)?
   4. Idempotency cross-service?

H. Kasten kırma
   1. Network latency reproduction (Toxiproxy)?
   2. Backend crash → CB OPEN → fallback?
   3. Saga compensation reproduce + fix?
   4. 5 senaryo defterimde?

I. Testing
   1. 25+ integration test?
   2. TestContainers multi-container?
   3. Contract tests (Phase 12 hazırlık)?

J. Observability
   1. Prometheus + Grafana + Jaeger + Loki stack?
   2. RED method dashboard per service?
   3. CB OPEN alert + p99 alert?

K. Anti-pattern
   1. Distributed monolith risk değerlendirmesi?
   2. Sync HTTP chain 4+ hop var mı?
   3. Shared DB yok mu?
   4. Business exception CB trigger ediyor mu?
   5. POST retry Idempotency-Key olmadan?
   6. PII span attribute leak?

Her madde için PASS / FAIL / EKSIK işaretle, kanıt göster (file path + code reference).
```

---

## Defter notları (25 madde)

1. "Monolith → 4 microservice split'in stratejik DDD bounded context rasyoneli: ____."
2. "Database per service prensibi + cross-service data access stratejisi: ____."
3. "banking-commons shared library yönetimi (Money, Currency, AccountId): ____."
4. "API Gateway 4 cross-cutting concerns (auth, rate limit, CB, logging): ____."
5. "Spring Cloud Gateway reactive (Netty) — servlet mix uyumsuzluğu: ____."
6. "JWT filter header propagation downstream (X-User-Id, X-Tenant-Id): ____."
7. "K8s Service discovery DNS-based vs Eureka karşılaştırması: ____."
8. "Spring Boot Actuator probes (liveness, readiness, startup) banking için: ____."
9. "Resilience4j composition order (Retry → CB → TimeLimiter → Bulkhead): ____."
10. "Banking timeout hierarchy bottom-up (DB → service → gateway): ____."
11. "ignoreExceptions banking business exception (InsufficientFunds): ____."
12. "Fallback strategy karar (fail-fast vs cached vs queue vs degraded): ____."
13. "OpenTelemetry end-to-end trace Jaeger UI'da waterfall: ____."
14. "Kafka trace propagation header üzerinden async chain: ____."
15. "Saga + TCC kombine cross-bank transfer 3-phase: ____."
16. "TCC Cancel idempotent compensation banking için zorunlu: ____."
17. "Stuck saga recovery scheduler timeout SLA: ____."
18. "Idempotency cross-service Idempotency-Key middleware: ____."
19. "Toxiproxy ile network failure simulation banking için faydası: ____."
20. "Backend crash → CB OPEN → fallback recovery cycle: ____."
21. "Distributed monolith 7 belirtisi + assessment'tan çıkan sonuç: ____."
22. "Async HTTP 202 + status polling banking UX impact: ____."
23. "mTLS service-to-service banking için neden gerekli: ____."
24. "Outbox pattern (Phase 6) tüm servislerde + Saga kombinasyon: ____."
25. "TR bank tech ekibinin bu mimariyi kullanabilirliği — eksik gördüğüm 3 nokta: ____."
