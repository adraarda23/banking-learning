# Phase 9 Mini-Project — Banking Observability Stack (End-to-End)

## Hedef

Phase 7-8 mini-project banking microservice'lerine **full observability** entegre etmek: structured logging + metrics + distributed tracing + profiling + load testing. Production-grade incident response capability.

## Süre

10-12 gün (günde 3 saat)

## Önbilgi

- Phase 8 mini-project tamam
- Topic 9.1-9.7 bitti

---

## Görev listesi

### 1. Observability stack — docker compose (0.5 gün)

```yaml
# docker-compose.observability.yml
version: '3.8'

services:
  # Logs
  loki:
    image: grafana/loki:2.9.0
    ports: ['3100:3100']
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yaml
    depends_on: [loki]

  # Metrics
  prometheus:
    image: prom/prometheus:v2.51.0
    ports: ['9090:9090']
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
      - --enable-feature=exemplar-storage
      - --web.enable-lifecycle

  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports: ['9093:9093']
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/config.yml
    command:
      - --config.file=/etc/alertmanager/config.yml

  # Traces
  jaeger:
    image: jaegertracing/all-in-one:1.55
    ports:
      - '16686:16686'   # UI
      - '14250:14250'   # gRPC
      - '14268:14268'   # HTTP collector
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.97.0
    ports:
      - '4317:4317'   # OTLP gRPC
      - '4318:4318'   # OTLP HTTP
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    depends_on: [jaeger]

  # Profile
  pyroscope:
    image: grafana/pyroscope:1.5.0
    ports: ['4040:4040']
    volumes:
      - pyroscope-data:/var/lib/pyroscope

  # Visualization
  grafana:
    image: grafana/grafana:10.4.0
    ports: ['3000:3000']
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_FEATURE_TOGGLES_ENABLE: traceqlEditor,traceToMetrics
    volumes:
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - grafana-data:/var/lib/grafana
    depends_on: [prometheus, loki, jaeger, pyroscope]

volumes:
  loki-data: {}
  prometheus-data: {}
  pyroscope-data: {}
  grafana-data: {}
```

### 2. Spring Boot integration — all 4 services (1.5 gün)

Common library: `banking-observability`:

```xml
<dependencies>
    <!-- Logging -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>
    
    <!-- Metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- Tracing -->
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
        <artifactId>datasource-micrometer-spring-boot</artifactId>
    </dependency>
    
    <!-- Profile -->
    <dependency>
        <groupId>io.pyroscope</groupId>
        <artifactId>agent</artifactId>
        <version>0.13.0</version>
    </dependency>
</dependencies>
```

`application.yml` (each service):

```yaml
spring:
  application:
    name: ${SERVICE_NAME}

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, loggers
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 100ms, 200ms, 500ms, 1s, 2s
    tags:
      application: ${spring.application.name}
      environment: ${ENV:dev}
  prometheus:
    metrics:
      export:
        enabled: true
        prometheus.exemplars: true
  tracing:
    sampling:
      probability: 0.1   # head 10%; tail-based collector
    propagation:
      type: w3c
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
      compression: gzip

logging:
  config: classpath:logback-spring.xml
```

### 3. Structured logging (1.5 gün)

(Topic 9.1)

`logback-spring.xml` ortak. RequestContextFilter MDC. Masking. Async appender. Audit + Security markers.

### 4. Banking metrics (1.5 gün)

Her service'te 8+ custom metric:
- `banking_transfers_total{tenant, currency, type, result}`
- `banking_transfer_duration_seconds{tenant, currency, type}`
- `banking_logins_total{result, mfa_used}`
- `banking_card_operations_total{operation, result}`
- `banking_balance_queries_total{tenant}`
- `banking_fraud_alerts_total{severity}`
- `banking_saga_started_total{saga_type}`
- `banking_saga_completed_total{saga_type, result}`
- `banking_saga_stuck_count{saga_type}` (gauge)
- `outbox_pending_count`
- `outbox_stale_count`

Grafana dashboards:
- **Service Overview** — RED + USE per service
- **Banking Business** — TPS, revenue proxy, MFA stats
- **Distributed System** — saga, outbox, Kafka lag
- **SLO** — error budget remaining, burn rate

Alert rules (Topic 9.2):
- HighErrorRate (> 1%, 5m)
- TransferP99Slow (> 2s, 10m)
- DbPoolExhausted (> 95%, 2m)
- SagaStuck (> 0, 5m)
- OutboxStale (> 100, 3m)
- KafkaConsumerLagHigh (> 10k, 5m)
- ErrorBudgetBurnRate (14x, 2m)
- BruteForceAttack (> 50 fail/min)

### 5. Distributed tracing (1 gün)

(Topic 9.3, 7.6)

- Auto-instrumentation: HTTP + JDBC + Kafka
- Manual spans: banking.transfer, banking.fraud_check, banking.saga.step
- Baggage: tenant, customer.segment
- Banking attribute conventions (no PII)
- Exemplars: Prometheus histogram → Jaeger
- Tail-based sampling: OTel Collector

### 6. Profile (1 gün)

- Pyroscope agent attach (each service)
- JFR continuous (default settings)
- Production-safe (1-2% overhead)
- Differential profile workflow (pre/post deploy)

### 7. JMH benchmarks (1 gün)

(Topic 9.6)

`banking-benchmarks` Maven module:
- `MoneyBenchmark` (BigDecimal ops)
- `SerializationBenchmark` (Jackson vs Avro for events)
- `LockBenchmark` (concurrent balance update)
- `CacheBenchmark` (Caffeine vs Redis lookup)

CI: PR'da regression > 10% fail.

### 8. Load testing (1.5 gün)

(Topic 9.7)

k6 scripts:
- `smoke.js` — CI per PR (1 user, 10s)
- `load.js` — nightly (500 RPS, 30m)
- `stress.js` — weekly (find breakpoint)
- `spike.js` — monthly (100 → 2000 → 100)
- `soak.js` — release (200 RPS, 4h)

Scenarios:
- Login → browse → transfer flow
- Idempotency under load
- Realistic amount distribution
- Think time exponential

### 9. Incident response runbook (1 gün)

`runbooks/` directory:

- `high-error-rate.md` — A. HighErrorRate alert → steps
- `latency-spike.md` — p99 spike → exemplar → trace → profile
- `db-pool-exhausted.md` — Hikari pool > 95%
- `oom-killed.md` — Pod OOM → heap dump → MAT
- `deadlock.md` — Stuck threads → jstack → fix
- `saga-stuck.md` — saga_stuck_count > 0 → state DB → trace → compensation
- `outbox-stale.md` — outbox lag → publish job health
- `kafka-lag.md` — consumer lag → consumer scaling / replay
- `brute-force.md` — failed login burst → Keycloak lockout
- `cascading-failure.md` — service dependency down → CB state

Runbook template:
```markdown
# Runbook: HighErrorRate

## Symptom
Error rate > 1% for 5 minutes on transfer-service.

## Likely causes
- Downstream service (risk, account) errors
- DB issue
- Recent deploy

## Investigation
1. Check Grafana dashboard "transfer-service Overview"
   - Error rate by status code
   - p99 latency
   - DB pool active/max
2. Sample errors from Loki:
   ```
   {service="transfer-service", level="ERROR"} |~ "Exception"
   ```
3. Trace exemplar from histogram panel → Jaeger
4. Profile if CPU/memory unusual

## Resolution paths
- Recent deploy → rollback (kubectl rollout undo)
- DB issue → reach DBA team
- Downstream service → check CB state
- Real customer impact → escalate to ops

## Post-mortem
After resolution: 
- Add metric/alert if signal was late
- Add test if regression preventable
- Update this runbook
```

### 10. Production-like incident simulation (1 gün)

5 kasten kırma + observe:

#### Senaryo 1: Slow DB query
1. Add 2s `pg_sleep(2)` to account lookup query
2. Run k6 load test
3. Grafana: p99 spike
4. Jaeger exemplar → trace: DB span 2s
5. Action: revert query

#### Senaryo 2: Memory leak
1. Add unbounded HashMap cache code
2. Run soak test (1 hour)
3. Pyroscope: alloc rate steady but heap grow
4. JFR / MAT: HashMap retention
5. Fix: Caffeine maxSize

#### Senaryo 3: Deadlock
1. Inconsistent lock order (account1 → account2 vs account2 → account1)
2. Concurrent load test
3. Some transfers hang
4. jstack → "Found Java-level deadlock"
5. Fix: canonical lock order

#### Senaryo 4: Kafka lag
1. Slow consumer (artificial 500ms delay per message)
2. kafka_consumer_lag metric spike
3. Alert fires
4. Investigation: log + trace → slow processor
5. Fix: parallel consumer or batch size

#### Senaryo 5: Cascading failure
1. risk-service down (kubectl delete deployment)
2. transfer-service → CB opens → fallback to default risk score
3. Alert: high fallback rate
4. Trace: transfer trace shows CB fallback span
5. Resilience verify

### 11. Defter notları (15 item)

1. "Banking observability stack (Loki + Prometheus + Jaeger + Pyroscope + Grafana) docker-compose: ____."
2. "Spring Boot + Micrometer Tracing + OTLP exporter end-to-end: ____."
3. "Structured logging masking + MDC + async appender + Markers: ____."
4. "Banking metric set (10+ custom) cardinality budget: ____."
5. "Grafana dashboards (4): RED, business, distributed, SLO + error budget: ____."
6. "Alert rules + Alertmanager + Slack/PagerDuty integration: ____."
7. "Exemplars Prometheus → Jaeger drill-down: ____."
8. "Tail-based sampling OTel Collector (errors + slow + high-value): ____."
9. "Pyroscope continuous profile + JFR auto-dump on incident: ____."
10. "JMH benchmark CI (PR vs main, regression > 10% fail): ____."
11. "k6 load test (smoke per PR, load nightly, soak weekly): ____."
12. "10 incident runbooks (banking specific): ____."
13. "5 kasten kırma senaryosu reproduce + diagnose + fix + verify: ____."
14. "BDDK + KVKK log retention 5-10 yıl + tiered storage: ____."
15. "Production-like setup: dedicated CI runner + stable env + observability tie: ____."

---

## Tamamlama kriterleri

- [ ] Docker compose: Loki + Promtail + Prometheus + Alertmanager + Jaeger + OTel Collector + Pyroscope + Grafana
- [ ] 4 service: structured JSON logging + MDC + masking
- [ ] 4 service: 10+ business metric + dashboards
- [ ] 4 service: distributed tracing + manual span + baggage
- [ ] 4 service: Pyroscope continuous profile
- [ ] OTel Collector tail-sampling
- [ ] Grafana dashboard banking (4)
- [ ] Alert rules (8+) + Alertmanager
- [ ] JMH benchmark suite + CI regression check
- [ ] k6 load test (smoke + load + stress + spike + soak)
- [ ] 10 incident runbook (markdown)
- [ ] 5 kasten kırma senaryo + diagnose workflow recorded
- [ ] 15 defter notu
- [ ] BDDK retention + tiered storage notes

---

## Önemli not

Phase 9 = observability **operational maturity**. Banking production'da:
- Olay 3 AM olur — observability ile self-service ops
- Audit + KVKK retention regulatory
- Incident response time SLA (RTO/RPO)
- Capacity planning data-driven

Senior banking engineer için bu phase **tools mastery + workflow mastery** kombosu.
