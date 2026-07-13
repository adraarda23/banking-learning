# Topic 9.3 — Distributed Tracing (Deep): OpenTelemetry + Exemplars

## Hedef

Topic 7.6'daki distributed tracing temelinin üstüne **observability stack** olarak derinlemesine kurmak. OpenTelemetry Java SDK (manual + auto), Spring Boot integration via Micrometer Tracing, exemplars (metrics ↔ trace linking), custom span + attribute, baggage propagation, sampling stratejileri (head-based + tail-based), Kafka trace propagation, banking PII protection, trace storage (Jaeger / Tempo / Datadog APM), trace analysis patterns.

## Süre

Okuma: 2.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 7.6 (Distributed Tracing temel) bitti
- Topic 9.1, 9.2 (Logging + Metrics) bitti
- Span / Trace / Context kavramları biliniyor

---

## Kavramlar

### 1. Recap — Trace anatomy

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736  (16 bytes hex)

Trace
└── Span "GET /v1/transfers"              [SpanId: 00f067aa0ba902b7, Parent: -]
    ├── Span "transferService.transfer"   [SpanId: a3ce929d..., Parent: 00f067aa...]
    │   ├── Span "DB: account_lookup"     [...]
    │   ├── Span "DB: tx_insert"
    │   ├── Span "POST kafka transfer-events"
    │   └── Span "POST risk-service /assess"
    │       └── Span "DB: risk_check"     [propagated to risk-service]
```

W3C `traceparent` header HTTP üzerinde context taşır.

### 2. OpenTelemetry stack

```
Application
   ↓
[OpenTelemetry SDK]    (instrumentation + context)
   ↓
[OTLP exporter]        (OTLP gRPC/HTTP protocol)
   ↓
[OTel Collector]       (batch, filter, sample, route)
   ↓
[Jaeger / Tempo / Datadog APM / Honeycomb / ...]
```

**Micrometer Tracing** (Spring Boot 3+) = abstraction. Underneath OpenTelemetry veya Brave.

```xml
<dependencies>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
    <dependency>
        <groupId>net.ttddyy.observation</groupId>
        <artifactId>datasource-micrometer-spring-boot</artifactId>   <!-- DB query span -->
    </dependency>
</dependencies>
```

```yaml
spring:
  application:
    name: transfer-service
management:
  tracing:
    sampling:
      probability: 1.0   # dev için %100; prod %1-10
    propagation:
      type: w3c
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
      timeout: 10s
      compression: gzip
otel:
  resource:
    attributes:
      service.name: transfer-service
      service.version: 1.0.0
      deployment.environment: production
```

### 3. Auto-instrumentation — what's covered

Out-of-the-box span generation for:
- Spring Web (HTTP server + client)
- WebClient / RestTemplate
- JDBC (datasource-micrometer)
- Spring Data
- Kafka producer/consumer
- Redis
- gRPC
- HTTP filters (Spring Security)

Banking pratiği: Auto + manuel span business operasyonlar için.

### 4. Manual span — banking domain

```java
@Service
public class TransferService {
    
    private final ObservationRegistry observationRegistry;
    
    public Transfer transfer(TransferRequest req) {
        return Observation.createNotStarted("banking.transfer", observationRegistry)
            .lowCardinalityKeyValue("transfer.type", req.type().name())
            .lowCardinalityKeyValue("currency", req.currency())
            .lowCardinalityKeyValue("tenant", currentTenant())
            .highCardinalityKeyValue("transfer.id", req.transferId().toString())   // NO PII
            .observe(() -> {
                // Business logic
                validateLimit(req);
                executeTransfer(req);
                publishEvent(req);
                return result;
            });
    }
    
    private void validateLimit(TransferRequest req) {
        Observation.createNotStarted("banking.transfer.limit_check", observationRegistry)
            .lowCardinalityKeyValue("strategy", "rolling-window")
            .observe(() -> limitService.check(req));
    }
}
```

OpenTelemetry direct API:

```java
@Service
public class TransferService {
    
    private final Tracer tracer;
    
    public TransferService(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer("com.bank.transfer", "1.0.0");
    }
    
    public Transfer transfer(TransferRequest req) {
        Span span = tracer.spanBuilder("banking.transfer")
            .setAttribute("transfer.type", req.type().name())
            .setAttribute("currency", req.currency())
            .setAttribute("amount.bucket", bucketOf(req.amount()))   // bucketed, not raw
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // business
            Transfer result = doTransfer(req);
            span.setStatus(StatusCode.OK);
            return result;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    private String bucketOf(BigDecimal amount) {
        if (amount.compareTo(new BigDecimal("100")) < 0) return "small";
        if (amount.compareTo(new BigDecimal("10000")) < 0) return "medium";
        if (amount.compareTo(new BigDecimal("100000")) < 0) return "large";
        return "very_large";
    }
}
```

### 5. Span attributes — banking semantic conventions

Standard attributes (OpenTelemetry semantic conventions):
- `http.method`, `http.status_code`, `http.url`
- `db.system`, `db.statement` (banking için statement maskeleme)
- `messaging.system`, `messaging.destination`
- `rpc.method`, `rpc.service`
- `exception.type`, `exception.message`, `exception.stacktrace`

Banking custom:
- `tenant` — TR/DE/UK
- `branch` — branch code
- `transfer.type` — EFT/FAST/SWIFT
- `currency` — TRY/USD/EUR
- `amount.bucket` — small/medium/large (NOT raw amount)
- `saga.id`, `saga.type`, `saga.step`
- `transaction.id`
- `idempotency.key`
- `risk.score.bucket` — low/medium/high
- `channel` — mobile/web/branch/atm

**PII protection — banking:**
- `tc_kimlik` ASLA
- `card_pan` ASLA (`card.last4` OK)
- `email` masked
- `user.id` — internal UUID OK (PII değil ama high cardinality)
- `account.id.hash` — hashed account ID OK
- `amount` raw değil — `amount.bucket` ile

### 6. Span events — punctual events

```java
span.addEvent("limit_check_started");
// ...
span.addEvent("limit_check_completed", Attributes.of(
    AttributeKey.stringKey("result"), "approved",
    AttributeKey.longKey("duration_ms"), 45L
));
```

Events span süresinde "moments" — log entries gibi ama span'a bağlı.

Banking örnekler:
- `mfa_challenge_sent`
- `fraud_score_calculated`
- `saga_step_completed`
- `retry_attempt`
- `circuit_breaker_opened`

### 7. Exemplars — metric ↔ trace bağlantısı

Histogram bucket'a span ID embed. Grafana panel'de "p99 spike" gör → ilgili trace'e drill-down.

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 100ms, 500ms, 1s, 2s
    enable:
      all: true
    export:
      prometheus:
        enabled: true
        prometheus.exemplars: true   # ✓ Exemplars
```

Prometheus storage exemplars on:
```yaml
prometheus:
  storage:
    tsdb:
      exemplars-enabled: true
```

Grafana panel "Exemplars" toggle → spike noktasında traceId görünür → tıkla → Jaeger/Tempo'ya.

Banking pratiği: SLO breach gör → exemplar trace incele → root cause.

### 8. Baggage — cross-service context

Trace context'in ötesinde **business context** propagate:

```
HTTP header: baggage: tenant=TR,branch=istanbul-1,customer.tier=premium
```

```java
Baggage.current().toBuilder()
    .put("tenant", "TR")
    .put("branch", "istanbul-1")
    .put("customer.tier", "premium")
    .build()
    .makeCurrent();
```

Downstream service'lar read:

```java
String tier = Baggage.current().getEntryValue("customer.tier");
```

**Banking örnekler:**
- `tenant` — tüm service'lerde aynı (multi-tenancy)
- `customer.segment` — pricing, routing
- `feature.flag` — A/B test variant
- `incident.id` — investigation çağrılarını trace

**Tehlike:**
- Baggage HTTP header → büyürse header overhead
- PII baggage'a koymak (TC, email) → ASLA

### 9. Sampling stratejileri

#### Head-based sampling

Request başlangıcında karar (root span):

```java
SdkTracerProvider.builder()
    .setSampler(TraceIdRatioBasedSampler.create(0.1))   // %10
    .build();
```

**Avantaj:** Cheap, decision once.
**Dezavantaj:** Critical error trace'i kaybedebilir.

#### Tail-based sampling

Trace tamamlandıktan sonra karar (errors, slow, sampling rule).

```yaml
# otel-collector config
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow
        type: latency
        latency: {threshold_ms: 1000}
      - name: random
        type: probabilistic
        probabilistic: {sampling_percentage: 5}
      - name: banking-high-value
        type: numeric_attribute
        numeric_attribute: 
          key: amount.value
          min_value: 100000
```

**Avantaj:** Critical trace mutlaka kaydedilir.
**Dezavantaj:** Collector RAM (10-30s buffer), karmaşık.

Banking pratiği: **Tail-based** + örnekleme önerilir.

#### Banking sampling stratejisi

```
- All errors: 100%
- p99 slow (> 2s): 100%
- High-value transfers (> 100k TL): 100%
- Saga compensation: 100%
- Fraud alerts: 100%
- Login: 10%
- Account view (read-heavy): 1%
- Health checks: 0%
```

### 10. Kafka trace propagation

Kafka producer record header'a inject:

```java
// Producer
ProducerRecord<String, byte[]> record = new ProducerRecord<>(topic, key, value);
TextMapPropagator propagator = openTelemetry.getPropagators().getTextMapPropagator();
propagator.inject(Context.current(), record.headers(), HEADER_SETTER);
producer.send(record);
```

Consumer record header'dan extract:

```java
// Consumer
Context parent = propagator.extract(Context.current(), record.headers(), HEADER_GETTER);
Span span = tracer.spanBuilder("kafka.consume")
    .setParent(parent)
    .setSpanKind(SpanKind.CONSUMER)
    .startSpan();
```

Spring Kafka auto-instrumentation bunu yapar.

Banking trace: REST API → DB → Kafka publish → ... → Kafka consume (başka service) → DB → REST response. **End-to-end**.

### 11. Trace storage backends

#### Jaeger
- Open source
- ElasticSearch / Cassandra storage
- UI iyi
- Banking yaygın

#### Tempo (Grafana)
- Cheap object storage (S3)
- Tag-based search değil (recent: limited)
- Trace-to-metrics integration mükemmel (Grafana)
- Cost-effective

#### Datadog APM / New Relic / Honeycomb
- Managed
- High signal
- Cost premium

#### Banking trade-off:
- Cost: Tempo
- Feature: Datadog
- Open: Jaeger

### 12. Trace analysis — banking patterns

**Pattern 1: Slow request investigation**

```
1. Grafana dashboard: p99 transfer spike son 1 saat
2. Exemplar tıkla → traceId
3. Jaeger trace: waterfall view
4. Span "DB: account_lookup" → 2.5s (normal 50ms)
5. DB metric: hikaricp_connections_active 100% → pool exhausted
6. Root cause: connection leak (find via heap dump)
```

**Pattern 2: Cross-service error propagation**

```
1. Frontend report: "Transfer failed for some users"
2. Grafana error rate panel → transfer-service: 5% errors
3. Click exemplar → traceId
4. Jaeger: transfer-service → calls risk-service → risk-service returns 503
5. risk-service span has error: "DB connection refused"
6. Root cause: risk-service DB down. Circuit breaker should have triggered (didn't)
```

**Pattern 3: Saga stuck investigation**

```
1. Alert: saga_stuck_count > 0
2. Saga ID from saga state DB
3. Search Jaeger: saga.id="abc-123"
4. Trace: cross-bank transfer saga steps
5. Step "callRemoteBank" span: no response (timeout)
6. Saga state: awaiting compensation, but compensation logic depends on bank response → infinite loop
7. Fix: timeout-bounded compensation logic
```

### 13. Distributed tracing performance impact

**Overhead:**
- Span creation: ~1-10 μs
- Context propagation: ~1 μs
- Network export: async, batched
- Sampling: ~50 ns

**Typical impact:** < 1% latency, < 5% CPU.

**Optimization:**
- BatchSpanProcessor (not Simple)
- gRPC over HTTP (faster export)
- Sampling (head + tail combo)
- Limit attribute count per span (< 128)
- Limit event count per span (< 128)

### 14. Banking — distributed tracing anti-pattern'leri

**Anti-pattern 1: PII in span attributes**

```java
span.setAttribute("user.tc_kimlik", "12345678901");   // ❌ KVKK
span.setAttribute("card.pan", "4532-...");             // ❌ PCI-DSS
```

Trace backend'i SIEM gibi sıkı korunmuyor → leak.

**Anti-pattern 2: Sampling %100 production'da**

Trace storage exhaust + cost explosion. Head 1-10% + tail-based on errors/slow.

**Anti-pattern 3: Span explosion**

```java
for (item in list) {
    Span s = tracer.spanBuilder("item.process").startSpan();
    // ...
    s.end();
}
```

10k item → 10k span → trace overload. Aggregate or sample.

**Anti-pattern 4: High-cardinality attribute**

```java
span.setAttribute("request.body", req.toString());   // ❌ unbounded
```

Trace size bloat. Use bucketed / hashed values.

**Anti-pattern 5: Span name template-style**

```java
spanBuilder("/v1/accounts/12345/transfers/abc-def")   // ❌ unique per request
```

**Fix:** `/v1/accounts/{id}/transfers/{txId}` (route pattern).

**Anti-pattern 6: Context leak**

```java
span.makeCurrent();   // Scope return değil
// ... code
span.end();
// Subsequent code'da hala bu span "current" → leak
```

**Fix:** Try-with-resources `Scope`.

**Anti-pattern 7: Async context lost**

```java
CompletableFuture.supplyAsync(() -> {
    // Parent context yok burada
});
```

**Fix:** `Context.current().wrap(executor)` or `Observation`.

**Anti-pattern 8: Span event vs attribute kafa karışıklığı**

- Span attribute: span'in başında/sonunda known
- Span event: span süresinde happened (point-in-time)

**Anti-pattern 9: Manual instrumentation without auto check**

Auto-instrumentation bunu zaten yapıyor → duplicate span.

**Anti-pattern 10: Production'da `simpleSpanProcessor`**

Sync export her span'i. Latency spike. **BatchSpanProcessor** kullan.

---

## Önemli olabilecek araştırma kaynakları

- OpenTelemetry Specification
- OpenTelemetry Java SDK docs
- Micrometer Tracing
- W3C Trace Context
- Jaeger / Tempo / Datadog APM
- Distributed Systems Observability — Cindy Sridharan
- Honeycomb best practices
- Banking PCI-DSS span attribute guidance

---

## Mini task'ler

### Task 9.3.1 — OpenTelemetry + Micrometer Tracing setup (45 dk)

Dependency. otel-collector docker. Jaeger UI bağla. Health endpoint trace görüntü.

### Task 9.3.2 — Auto-instrumentation verify (30 dk)

HTTP + JDBC + Kafka producer/consumer span'ler otomatik. Jaeger'da gör.

### Task 9.3.3 — Manual span banking domain (60 dk)

`Observation` API ile transfer flow span'leri. Attribute (tenant, type, amount.bucket).

### Task 9.3.4 — Exemplars Prometheus → Jaeger (60 dk)

Metrics endpoint exemplars on. Grafana panel exemplar toggle. p99 spike → trace drill-down.

### Task 9.3.5 — Baggage cross-service (45 dk)

Gateway → service A → service B. Tenant baggage propagate. Service B'de read.

### Task 9.3.6 — Sampling head + tail (60 dk)

OTel Collector tail-sampling config. All errors + slow > 1s + random 5%.

### Task 9.3.7 — Kafka trace propagation verify (45 dk)

Producer record header inject. Consumer side trace continue. Jaeger end-to-end.

### Task 9.3.8 — Banking PII protection audit (30 dk)

Mevcut span'leri tara. Hiçbir attribute'ta TC/PAN/email plain olmasın. Test.

### Task 9.3.9 — Async context propagation (30 dk)

`@Async` method veya CompletableFuture. Trace continue (Spring autowire çoğunlukla halleder).

### Task 9.3.10 — Trace-based debugging (45 dk)

Intentional slow query simulate. Grafana → exemplar → trace → root cause flow.

---

## Test yazma rehberi

```java
@SpringBootTest
@Import(TestConfig.class)
class TracingTest {
    
    @Autowired ObservationRegistry observationRegistry;
    
    @MockBean(answer = Answers.CALLS_REAL_METHODS)
    Tracer tracer;
    
    @Test
    void transferShouldCreateSpan() {
        InMemorySpanExporter exporter = ...;
        
        transferService.transfer(testRequest);
        
        List<SpanData> spans = exporter.getFinishedSpanItems();
        SpanData transferSpan = spans.stream()
            .filter(s -> s.getName().equals("banking.transfer"))
            .findFirst().orElseThrow();
        
        Map<String, Object> attrs = transferSpan.getAttributes().asMap();
        assertThat(attrs).containsEntry(AttributeKey.stringKey("tenant"), "TR");
        assertThat(attrs).containsEntry(AttributeKey.stringKey("currency"), "TRY");
    }
    
    @Test
    void transferShouldNotIncludePii() {
        TransferRequest req = new TransferRequest(...);
        transferService.transfer(req);
        
        List<SpanData> spans = exporter.getFinishedSpanItems();
        
        spans.forEach(span -> {
            String all = span.toString();
            assertThat(all).doesNotContain("12345678901");   // No TC
            assertThat(all).doesNotContain(req.fromAccountIban());
        });
    }
    
    @Test
    void errorShouldRecordException() {
        when(externalApi.call()).thenThrow(new RuntimeException("simulated"));
        
        assertThatThrownBy(() -> transferService.transfer(req))
            .isInstanceOf(RuntimeException.class);
        
        SpanData span = exporter.getFinishedSpanItems().stream()
            .filter(s -> s.getStatus().getStatusCode() == StatusCode.ERROR)
            .findFirst().orElseThrow();
        
        assertThat(span.getEvents()).anyMatch(e -> e.getName().equals("exception"));
    }
    
    @Test
    void traceContextShouldPropagateAcrossServices() throws Exception {
        // Wire two test services
        String traceId = service1.callService2();
        
        SpanData service2Span = exporter.getFinishedSpanItems().stream()
            .filter(s -> s.getName().contains("service2"))
            .findFirst().orElseThrow();
        
        assertThat(service2Span.getTraceId()).isEqualTo(traceId);   // Same trace
    }
    
    @Test
    void kafkaShouldPropagateTraceHeader() {
        Span parentSpan = tracer.spanBuilder("test.parent").startSpan();
        try (Scope s = parentSpan.makeCurrent()) {
            kafkaProducer.send("topic", event);
        }
        parentSpan.end();
        
        // Consumer side
        ConsumerRecord<?, ?> record = consumeOne("topic");
        Iterable<Header> headers = record.headers().headers("traceparent");
        assertThat(headers).isNotEmpty();
        
        String traceparent = new String(headers.iterator().next().value());
        assertThat(traceparent).contains(parentSpan.getSpanContext().getTraceId());
    }
}
```

---

## Claude-verify prompt

```
Distributed tracing implementation'ımı banking-grade kriterlere göre değerlendir:

1. SDK + Backend:
   - OpenTelemetry SDK + Micrometer Tracing bridge?
   - OTLP exporter (gRPC veya HTTP)?
   - OTel Collector (batch + sample)?
   - Jaeger / Tempo / Datadog backend?

2. Auto-instrumentation:
   - HTTP server + client?
   - JDBC?
   - Kafka producer/consumer?
   - Spring Security filter?

3. Manual spans:
   - Banking domain (banking.transfer, banking.fraud_check)?
   - Span kind (INTERNAL, CLIENT, SERVER, PRODUCER, CONSUMER)?
   - Low cardinality attribute (tenant, currency, type)?
   - High cardinality NOT in span name?

4. Attributes:
   - tenant, branch, transfer.type, currency, channel?
   - amount.bucket (NOT raw amount)?
   - saga.id, transaction.id (UUID OK)?
   - PII (TC, PAN, email) YOK?

5. Span events:
   - Punctual moment (mfa_sent, retry_attempt, cb_opened)?
   - Error recordException()?

6. Exemplars:
   - Prometheus exemplars enabled?
   - Histogram metric + traceId link?
   - Grafana panel exemplar toggle?

7. Baggage:
   - tenant cross-service?
   - PII baggage'a YOK?

8. Sampling:
   - Head-based 1-10% prod?
   - Tail-based all errors + slow + high-value?
   - Health check 0%?

9. Kafka propagation:
   - Producer header inject?
   - Consumer header extract?
   - End-to-end trace continuity?

10. Performance:
    - BatchSpanProcessor?
    - Span attribute limit (< 128)?
    - Async export?
    - Sampling reduces volume?

11. Banking domain:
    - End-to-end transfer trace (gateway → 4 service → DB → Kafka)?
    - Saga trace continuity?
    - Compensation trace?
    - Cross-bank scenario?

12. Anti-pattern:
    - PII in span YOK?
    - Span explosion (loop'ta) YOK?
    - High cardinality attribute (request body) YOK?
    - Template-style span name YOK?
    - Context leak (Scope close) YOK?
    - SimpleSpanProcessor prod YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] OpenTelemetry + Micrometer Tracing setup
- [ ] OTel Collector + Jaeger backend (or Tempo)
- [ ] Auto-instrumentation HTTP + JDBC + Kafka
- [ ] 5+ manual span banking domain
- [ ] PII-free attribute audit pass
- [ ] Exemplars Prometheus → Grafana → Jaeger drill-down
- [ ] Baggage propagation 2+ service
- [ ] Tail-based sampling (errors + slow + high-value)
- [ ] Kafka end-to-end trace
- [ ] BatchSpanProcessor + performance check
- [ ] 5+ integration test
- [ ] 1 root-cause investigation walkthrough (exemplar → trace → fix)

---

## Defter notları (10 madde)

1. "OpenTelemetry + Micrometer Tracing bridge banking abstraction: ____."
2. "Auto vs manual instrumentation balance + ne zaman manuel: ____."
3. "Span attribute banking semantic convention (tenant, currency, bucket): ____."
4. "PII span attribute leak risk (TC, PAN) — banking trace backend SIEM olmayabilir: ____."
5. "Exemplars Prometheus histogram + traceId → Jaeger drill-down workflow: ____."
6. "Baggage cross-service business context (tenant, segment, feature_flag): ____."
7. "Head vs tail-based sampling banking trade-off: ____."
8. "Tail-based: all errors + slow + high-value + low-cardinality random: ____."
9. "Kafka trace propagation producer/consumer header W3C: ____."
10. "Trace-based root cause workflow (alert → exemplar → trace → fix): ____."
