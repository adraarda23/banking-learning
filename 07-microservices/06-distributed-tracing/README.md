# Topic 7.6 — Distributed Tracing: OpenTelemetry + Jaeger

## Hedef

Microservice'ler arası request flow'unu **end-to-end trace** etmek. OpenTelemetry standard API/SDK, trace propagation (W3C traceparent), span context, baggage, sampling stratejileri. Spring Boot 3 Micrometer Tracing entegrasyonu. Banking: bir transfer request'inin 5 servisten geçişini Jaeger UI'da takip.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 7.1-7.5 bitti (microservice architecture)
- Phase 1'de basit `X-Trace-Id` filter (Topic 1.7)
- Phase 9 (Observability) detaylı tracing — burası temel

---

## Kavramlar

### 1. Distributed tracing problemi

Mikroservice'lerde:

```
User: POST /v1/transfers
   ↓
[Gateway]
   ↓
[transfer-service]
   ↓ ↓ ↓
[account-service] [fraud-service] [audit-service]
   ↓
[DB]
   ↓
[Kafka publish]
   ↓
[notification-service consume]
   ↓
[SMS gateway]
```

10+ hop. "Yavaş request" şikayetinde:
- Hangi hop yavaş?
- Network mi, DB mi, business logic mi?
- Hangi servisin hangi version'u?
- Cross-service correlation nasıl?

**Tracing** = request flow'un end-to-end recording'i.

### 2. Trace, Span, Context

**Trace:** Bir request'in **tam yaşam döngüsü**. Unique `traceId` ile tanımlanır.

**Span:** Trace içinde **bir iş birimi**. Method call, DB query, external request.
- `spanId` (unique)
- `parentSpanId` (hierarchical)
- `traceId` (root)
- `startTime`, `endTime`
- `attributes` (tags)
- `events` (log point'ler)
- `status` (OK / ERROR)

**Trace = span'ler ağacı:**

```
traceId: abc123
├── span 1: Gateway POST /v1/transfers (200ms)
│   ├── span 2: Auth filter (5ms)
│   └── span 3: transfer-service.execute (180ms)
│       ├── span 4: account-service.debit (50ms)
│       │   └── span 5: postgres UPDATE (10ms)
│       ├── span 6: account-service.credit (45ms)
│       ├── span 7: fraud-service.score (30ms)
│       │   └── span 8: redis.GET (2ms)
│       └── span 9: kafka.publish (15ms)
```

Visualize edildiğinde **timeline + waterfall**. Bottleneck görsel.

### 3. OpenTelemetry (OTel) — modern standard

**Eski:**
- OpenTracing (API spec, no implementation)
- OpenCensus (Google, metrics + traces)
- Vendor-specific (Datadog APM, New Relic)

**2021 merger:** OpenTelemetry = OpenTracing + OpenCensus.

**Stack:**
- **API:** Vendor-neutral interfaces (Tracer, Span)
- **SDK:** Implementation (sampling, processors, exporters)
- **OTLP** (OpenTelemetry Protocol): Binary, gRPC veya HTTP
- **Collector:** Aggregator, route to backend
- **Backend:** Jaeger, Zipkin, Tempo, Datadog, vb.

### 4. Spring Boot 3 + Micrometer Tracing

Spring Boot 3 default `Micrometer Tracing` (Sleuth replacement).

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # %100 trace (dev). Production: 0.1
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
      compression: gzip
      timeout: 30s

spring:
  application:
    name: transfer-service
```

Auto-instrumentation otomatik:
- HTTP server (incoming requests → root span)
- HTTP client (WebClient, RestTemplate → child span)
- JDBC (database calls)
- Kafka (producer/consumer)
- Reactive (Project Reactor)

**Zero-code tracing.** Trace + span otomatik üretilir.

### 5. W3C traceparent — trace propagation

**Standard:** W3C Trace Context.

HTTP header:

```
traceparent: 00-{trace-id-32hex}-{parent-span-id-16hex}-{trace-flags-2hex}

Example:
traceparent: 00-4a1b2c3d5e6f7a8b9c0d1e2f3a4b5c6d-00f067aa0ba902b7-01
            ╰─ version (00)
                ╰─ trace-id (32 hex chars = 16 bytes)
                                                ╰─ parent span id (16 hex = 8 bytes)
                                                                  ╰─ flags (sampled=01)
```

`tracestate` — vendor-specific data:

```
tracestate: dd=trace-segment,nr=newrelic-segment
```

#### Kafka trace propagation

Producer header'a koyar:

```
producer.send(record):
  record.headers = {
    "traceparent": "00-abc...-def...-01",
    "X-Trace-Id": "abc..."   (legacy compat)
  }
```

Consumer okuyor:

```
@KafkaListener
public void onMessage(@Payload Event e, @Header("traceparent") String traceparent) {
    // OTel context'i restore
    ...
}
```

Spring Kafka + Micrometer Tracing **otomatik** propagate eder.

### 6. Manual span — domain-specific instrumentation

Auto-instrumentation çoğu zaman yeterli. Bazen **business-critical** span eklemek gerekir:

```java
@Service
public class TransferService {
    
    private final Tracer tracer;
    
    public Mono<Transfer> execute(TransferRequest req) {
        return Mono.deferContextual(ctx -> {
            Span span = tracer.spanBuilder("transfer.execute")
                .setAttribute("transfer.amount", req.getAmount().toString())
                .setAttribute("transfer.currency", req.getCurrency())
                .setAttribute("transfer.from_account", req.getFromAccountId().toString())
                .startSpan();
            
            try (Scope scope = span.makeCurrent()) {
                return doExecute(req)
                    .doOnSuccess(transfer -> {
                        span.setAttribute("transfer.id", transfer.getId().toString());
                        span.setStatus(StatusCode.OK);
                    })
                    .doOnError(e -> {
                        span.recordException(e);
                        span.setStatus(StatusCode.ERROR, e.getMessage());
                    })
                    .doFinally(signalType -> span.end());
            }
        });
    }
}
```

**Banking custom span'ler:**
- `transfer.execute`
- `fraud.score`
- `account.balance_check`
- `kyc.verify`
- `tcmb.fx_rate_fetch`

### 7. @Observed annotation — Spring Boot 3.2+

```java
@Service
public class TransferService {
    
    @Observed(name = "transfer.execute", 
              contextualName = "execute-transfer",
              lowCardinalityKeyValues = {"operation", "transfer"})
    public Mono<Transfer> execute(TransferRequest req) {
        return doExecute(req);
    }
}
```

`@Observed` otomatik span yaratır, exception kaydeder, duration ölçer. Daha temiz.

### 8. Span attributes — semantic conventions

OpenTelemetry **semantic conventions** — standart attribute isimleri.

```java
span.setAttribute("http.method", "POST");
span.setAttribute("http.url", "/v1/transfers");
span.setAttribute("http.status_code", 201);
span.setAttribute("db.system", "postgresql");
span.setAttribute("db.statement", "SELECT * FROM accounts WHERE id = ?");
span.setAttribute("messaging.system", "kafka");
span.setAttribute("messaging.destination", "banking.transfers");
```

Custom (domain):

```java
span.setAttribute("banking.transfer.amount", amount.toString());
span.setAttribute("banking.account.id", accountId.toString());
span.setAttribute("banking.currency", "TRY");
```

### 9. PII uyarısı — banking için kritik

Span attribute'larında **PII YOK**:

```java
// ❌ PII LEAK
span.setAttribute("user.tc_kimlik", "12345678901");
span.setAttribute("user.full_name", "Ali Veli");
span.setAttribute("card.pan", "4111-1111-1111-1234");
span.setAttribute("account.balance", "150000.00");

// ✓ Internal ID OK
span.setAttribute("user.id", uuid);
span.setAttribute("account.id", uuid);
span.setAttribute("transfer.amount", "AMOUNT_REDACTED");   // veya bucket
```

Tracing UI'da herkes (devops, contractor, vb.) görür. KVKK + PCI-DSS violation.

### 10. Sampling — production cost

**Tüm request'leri trace etmek pahalı.** Volume × span × storage × query.

Banking 1000 req/sec × her trace 10 span × 100 byte/span = 1 MB/sec = 86 GB/gün.

**Sampling stratejileri:**

#### Head-based sampling (yaygın)

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # %10
```

Request başında karar — %10 sampled, %90 dropped. **Simple, cheap.**

**Tehlike:** Error veya slow request'ler de %10 yakalanır. Critical event'ler kaçabilir.

#### Tail-based sampling (advanced)

Tüm span'ler collector'a gönderilir. **Collector**: error veya slow keep, normal drop.

```yaml
# OTel Collector config
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-policy
        type: latency
        latency: {threshold_ms: 1000}
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: {sampling_percentage: 5}
```

Error + slow → %100. Normal → %5.

Banking için ideal — "ilginç" trace'leri yakalar.

**Maliyet:** Collector'ın tüm span'leri 10s buffer'ı tutması gerekir.

#### Head + tail combo

```
App:        head 100% (tüm span'leri OTel'e gönder)
Collector:  tail decide → keep / drop
```

Best of both. Banking modern stack'lerde yaygın.

### 11. Sampling per-tenant / per-endpoint

```yaml
management:
  tracing:
    sampling:
      probability: 0.1
```

Bazen **belli endpoint'leri** %100 sample etmek istersin.

```java
@Bean
public Sampler customSampler() {
    return Sampler.parentBased(
        new Sampler() {
            @Override
            public SamplingResult shouldSample(Context parentContext, String traceId, 
                                               String name, SpanKind spanKind,
                                               Attributes attributes, List<LinkData> parentLinks) {
                if (name.startsWith("transfer.")) {
                    return SamplingResult.recordAndSample();   // %100
                }
                if (name.startsWith("fraud.")) {
                    return SamplingResult.recordAndSample();
                }
                return new ProbabilitySampler(0.1).shouldSample(...);   // diğer %10
            }
        }
    );
}
```

Critical operation'lar (transfer, fraud) %100 sampled. Genel %10.

### 12. Jaeger UI workflow

```yaml
# docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one:1.55
  environment:
    COLLECTOR_OTLP_ENABLED: "true"
  ports:
    - "16686:16686"   # UI
    - "4318:4318"     # OTLP HTTP
    - "4317:4317"     # OTLP gRPC
```

`http://localhost:16686`:

1. Service dropdown — `transfer-service` seç
2. Operation dropdown — `POST /v1/transfers`
3. Time range, lookback
4. "Find Traces" → liste

Trace detail:
- Timeline (waterfall view)
- Span hierarchy
- Span attributes
- Errors highlighted

**Banking diagnostic workflow:**

```
1. Grafana alert: p99 latency > 500ms
2. Exemplar tıkla → traceId al
3. Jaeger'a git → trace'i bul
4. Waterfall'da en geniş span'i gör
5. Span attribute'larından kök neden bul
6. Log'a git (traceId ile filter) → detay
```

### 13. Trace + log + metric correlation

**Tracing + logging + metrics = observability üçlüsü.**

#### Log → trace correlation

Logback pattern'inde traceId:

```yaml
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

Log line:

```
2025-05-13T10:30:00.123Z INFO [transfer-service,abc123,def456] TransferService - Processing transfer
```

`traceId` Loki/Elasticsearch search'üyle Jaeger trace'e kolay link.

#### Metric → trace (exemplars)

Prometheus exemplars:

```
http_server_requests_seconds_bucket{le="0.5"} 12345 # {trace_id="abc123"} 0.488 1716000000
```

Grafana panel'ında dot tıkla → trace ID → Jaeger.

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
```

Banking observability: Bir Grafana panel'da p99 spike → exemplar → Jaeger → root cause.

### 14. Baggage — cross-cutting context

**Trace context** (traceId, spanId) — auto-propagate.

**Baggage** — application-defined cross-cutting data.

```java
Baggage.builder()
    .put("tenant", "tr")
    .put("user-tier", "premium")
    .put("feature-flag-x", "enabled")
    .build()
    .makeCurrent();
```

Downstream service'lerde:

```java
String tenant = Baggage.current().getEntryValue("tenant");
```

**Banking örnek:** Multi-tenant context propagation.

**Tehlike:** Baggage **tüm span'lara propagate** + HTTP header'a yazılır. **PII koyma**, payload büyütme.

Sadece **temel context** (tenant ID, user tier — internal ID'ler).

### 15. Banking örnek — full transfer trace

```
traceId: 4a1b2c3d5e6f7a8b9c0d1e2f3a4b5c6d

[Gateway] POST /v1/transfers (totalDuration: 280ms)
├── auth-filter (8ms)
│   └── jwt.validate (3ms)
├── rate-limit (1ms)
│   └── redis.GET (1ms)
└── route to transfer-service (270ms)
    └── transfer-service.HTTP POST /transfers (260ms)
        ├── transfer.execute (250ms)
        │   ├── idempotency.check (5ms)
        │   │   └── postgres SELECT (3ms)
        │   ├── fraud.score (45ms)
        │   │   ├── fraud-service.HTTP POST /score (40ms)
        │   │   │   └── fraud.compute_score (35ms)
        │   │   │       └── redis.GET account-info (2ms)
        │   ├── account.debit (60ms)
        │   │   ├── account-service.HTTP POST /accounts/A/debit (55ms)
        │   │   │   ├── account.validate (3ms)
        │   │   │   ├── postgres SELECT FOR UPDATE (8ms)
        │   │   │   ├── account.withdraw (1ms)
        │   │   │   ├── postgres UPDATE (10ms)
        │   │   │   └── journal_line.insert (8ms)
        │   │   │       └── postgres INSERT (3ms)
        │   ├── account.credit (55ms)
        │   ├── transfer.save (15ms)
        │   │   └── postgres INSERT (12ms)
        │   ├── outbox.publish (10ms)
        │   │   └── postgres INSERT outbox (6ms)
        │   └── http_response (1ms)
        └── http_response_to_gateway (5ms)

Async (separate trace? parent linked):
[outbox publisher] (linked to original trace)
├── postgres SELECT FOR UPDATE outbox (10ms)
└── kafka.publish (15ms)

[notification-service] consumer (linked)
├── kafka.consume (1ms)
├── notification.send (200ms)
│   └── sms-gateway.HTTP POST (180ms)
└── postgres INSERT processed_event (5ms)
```

**Banking diagnostic:**
- `account.debit` 60ms — DB SELECT FOR UPDATE 8ms (lock contention?)
- `sms-gateway.HTTP POST` 180ms — external dependency slow
- `notification.send` async — user'a transfer 280ms'te döndü, SMS sonra

### 16. Banking anti-pattern'leri

**Anti-pattern 1: %100 sampling production'da**

Storage + cost patlama. %1-10 head-based + tail-based for errors.

**Anti-pattern 2: Every method'da manual span**

Auto-instrumentation yeter. Manual sadece **business critical**.

```java
public void someMethod() {
    Span span = tracer.spanBuilder("someMethod").startSpan();   // ❌ otomatik zaten var
    // ...
}
```

**Anti-pattern 3: PII span attribute'inda**

KVKK + PCI-DSS violation. **YASAK.**

**Anti-pattern 4: Span name high-cardinality**

```java
span.setSpanName("GET /accounts/123e4567-e89b-12d3-a456");   // ❌ her ID unique span name
```

Cardinality patlama. **Template:**

```java
span.setSpanName("GET /accounts/{id}");
span.setAttribute("account.id", id.toString());
```

**Anti-pattern 5: Sync span uzun async chain'de**

Reactive Mono içinde sync `Span.makeCurrent()` — context kaybolabilir.

Banking pratiği: Reactive'da `Mono.deferContextual` + `MicrometerObservation`.

**Anti-pattern 6: Trace context cross-service propagate edilmiyor**

Service A'ya gelen request'in `traceparent` header'ı Service B'ye gitmiyor → trace **kopuk**.

Spring Boot 3 + Micrometer otomatik. Custom HTTP client'ta manuel:

```java
String traceparent = Span.current().getSpanContext().getTraceId() + ...;
httpClient.addHeader("traceparent", traceparent);
```

**Anti-pattern 7: Async operation parent span lost**

`@Async` veya Kafka publish/consume → parent span context kayıp.

Spring Kafka + Tracing built-in. Manual async için context propagation şart.

---

## Önemli olabilecek araştırma kaynakları

- OpenTelemetry official documentation
- "Mastering Distributed Tracing" (Yuri Shkuro — Jaeger author)
- Spring Boot 3 Tracing reference
- Micrometer Tracing migration from Sleuth
- W3C Trace Context spec
- Jaeger / Tempo / Zipkin comparisons

---

## Mini task'ler

### Task 7.6.1 — Spring Boot 3 tracing setup (30 dk)

`pom.xml`, `application.yml` config. OTLP exporter to Jaeger.

```yaml
management:
  tracing:
    sampling.probability: 1.0
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
```

### Task 7.6.2 — Jaeger docker-compose (15 dk)

Jaeger all-in-one container. UI port 16686.

Spring Boot app boot, GET /actuator/health, Jaeger UI'da trace görmeli.

### Task 7.6.3 — End-to-end trace (60 dk)

4 servis chain: Gateway → transfer-service → account-service → DB. Bir transfer POST yap. Jaeger'da full trace görünür mü?

Waterfall view'da her span timing'i, attribute'ları incele.

### Task 7.6.4 — Manual @Observed span (30 dk)

`TransferService.execute()` üzerine `@Observed("transfer.execute")`. Jaeger'da bu özel span'i gör.

Attribute ekle: `transfer.amount`, `transfer.currency`.

### Task 7.6.5 — Kafka trace propagation (45 dk)

transfer-service Kafka publish → notification-service consume. Trace **tek bir trace** olarak görünmeli (linked).

### Task 7.6.6 — Log MDC ile correlation (30 dk)

Logback pattern + traceId, spanId. Log line'da görünür.

Loki veya Elasticsearch'te `traceId="..."` ile log filter.

### Task 7.6.7 — Custom sampling (30 dk)

`transfer.*` operations %100, diğer %10 sampling. Custom `Sampler` bean.

### Task 7.6.8 — PII check (15 dk)

Mevcut span'lerde PII var mı? `setAttribute("user.tc_no", ...)` veya `setAttribute("card.pan", ...)` gibi → bul ve mask et.

---

## Test yazma rehberi

```java
@SpringBootTest
@Testcontainers
class TracingIntegrationTest {
    
    @Container
    static GenericContainer<?> jaeger = new GenericContainer<>("jaegertracing/all-in-one:1.55")
        .withExposedPorts(16686, 4318);
    
    @DynamicPropertySource
    static void configureTracing(DynamicPropertyRegistry registry) {
        registry.add("management.otlp.tracing.endpoint", 
            () -> "http://" + jaeger.getHost() + ":" + jaeger.getMappedPort(4318) + "/v1/traces");
        registry.add("management.tracing.sampling.probability", () -> "1.0");
    }
    
    @Test
    void shouldCreateTraceForRequest() throws Exception {
        // Make request
        webClient.post().uri("/v1/transfers")
            .header("Authorization", "Bearer ...")
            .bodyValue(createTransferRequest())
            .exchange()
            .expectStatus().isCreated();
        
        // Wait for trace export (async)
        Thread.sleep(2000);
        
        // Query Jaeger API
        String response = restTemplate.getForObject(
            "http://" + jaeger.getHost() + ":" + jaeger.getMappedPort(16686) +
            "/api/traces?service=transfer-service&operation=POST%20/v1/transfers",
            String.class
        );
        
        assertThat(response).contains("transfer.execute");
        assertThat(response).contains("traceID");
    }
    
    @Test
    void shouldPropagateTraceAcrossServices() {
        // Make request
        // Assert downstream call had traceparent header
        // Assert child span linked to parent
    }
    
    @Test
    void shouldNotIncludePiiInSpanAttributes() {
        // Make request with sensitive data
        // Query trace
        // Assert no PII fields
    }
}
```

---

## Claude-verify prompt

```
Distributed tracing setup'ımı banking-grade kriterlere göre değerlendir:

1. OpenTelemetry setup:
   - Micrometer Tracing bridge + OTLP exporter?
   - Spring Boot 3 native config?
   - OTLP collector endpoint configurable?

2. Auto-instrumentation:
   - HTTP server (incoming) trace'liyor mu?
   - HTTP client (WebClient) child span yaratıyor mu?
   - JDBC database calls span'liyor mu?
   - Kafka producer/consumer span'liyor mu?

3. Trace propagation:
   - W3C traceparent header cross-service?
   - Kafka header trace propagation?
   - @KafkaListener parent span linked?

4. Manual span:
   - Critical business operation (@Observed)?
   - Span attribute'lar semantic conventions?
   - Custom domain attribute'lar (banking.transfer.amount)?

5. Sampling:
   - Production sampling rate makul (1-10%)?
   - Critical operation'lar %100 (custom sampler)?
   - Tail-based sampling alternative düşünülmüş mü?

6. PII protection:
   - Span attribute'lerinde PII YOK mu?
   - User.tc_no, card.pan, balance hiçbir yerde?
   - Internal ID (UUID) OK?

7. Cardinality:
   - Span name template'li (path param yok)?
   - High-cardinality attribute'ler tag'a koyulmamış mı?

8. Log correlation:
   - Logback pattern'inde traceId + spanId?
   - Loki/ELK'da traceId filter?

9. Metric exemplars:
   - Prometheus exemplars enabled?
   - Grafana panel'dan Jaeger'a link?

10. Anti-pattern:
    - %100 sampling production'da?
    - Every method manual span?
    - PII leak?
    - Cross-service propagation eksik?
    - Async parent span lost?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Spring Boot 3 + Micrometer Tracing + OTLP setup
- [ ] Jaeger docker-compose
- [ ] End-to-end trace 4 servis (Gateway → transfer → account → DB)
- [ ] Manual @Observed business span
- [ ] Kafka trace propagation
- [ ] Log MDC traceId entegrasyonu
- [ ] Custom sampler (transfer %100, diğer %10)
- [ ] PII scan + mask
- [ ] Metric exemplars Grafana'da görünür
- [ ] 5+ integration test

---

## Defter notları (10 madde)

1. "Trace + Span + Context kavramları + waterfall view: ____."
2. "OpenTelemetry vs eski (OpenTracing, OpenCensus, vendor): ____."
3. "W3C traceparent header format ve cross-service propagation: ____."
4. "Auto-instrumentation vs manual span ne zaman hangisi: ____."
5. "Sampling head-based vs tail-based banking için karar: ____."
6. "PII span attribute'larında tehlikesi (KVKK + PCI-DSS): ____."
7. "Span name template (low-cardinality) — high-cardinality tag tuzağı: ____."
8. "Exemplars Grafana → Jaeger link banking observability: ____."
9. "Kafka trace propagation header üzerinden async chain: ____."
10. "Baggage cross-cutting context — tenant propagation banking: ____."
