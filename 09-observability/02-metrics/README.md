# Topic 9.2 — Metrics: Micrometer + Prometheus + Grafana

## Hedef

Banking servisinde **metric** ölçümünü production-grade kurmak: Micrometer abstraction, Prometheus scrape, Grafana dashboards, RED/USE methodology, banking-specific business metrics, SLI/SLO/SLA, alert rules.

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 9.1 (Structured Logging) bitti
- Prometheus, Grafana ne işe yarar duy
- Quantile, percentile, histogram temel

---

## Kavramlar

### 1. 3 pillar — log vs metric vs trace

| | Log | Metric | Trace |
|---|---|---|---|
| **Veri tipi** | Event (discrete) | Time-series numeric | Distributed call chain |
| **Storage** | Yüksek (TB/gün) | Düşük (compact) | Orta |
| **Query** | Search (full-text) | Aggregation (rate, sum) | Waterfall |
| **Retention** | 30 gün - 10 yıl | 1-2 yıl | 7-30 gün |
| **Banking örnek** | "Transfer initiated for user X" | `transfer_count_total{result="success"} 1234` | "GET /transfers → DB query → Kafka publish" |

**Metric** = "ne kadar / ne sıklıkta / ne kadar süre". Log/Trace'in **özetidir**.

### 2. Metric types — Prometheus 4 type

#### Counter
Monotonically increasing. Reset = process restart.

```
http_requests_total{service="account", endpoint="/v1/accounts", status="200"} 12345
```

Banking örnek:
- `transfers_total{tenant="TR", currency="TRY", result="success"}` → sayı
- `logins_failed_total{reason="invalid_password"}` → sayı

PromQL:
```promql
rate(transfers_total[5m])    # 5dk window'da TPS
sum(rate(transfers_total[5m])) by (tenant)
```

#### Gauge
Up/down. Anlık değer.

```
jvm_memory_used_bytes{area="heap"} 524288000
db_connections_active{pool="HikariCP"} 12
```

Banking:
- `account_balance_total{tenant="TR"}` (toplam müşteri bakiyesi)
- `loan_portfolio_value{currency="TRY"}`
- `active_sessions_count`

#### Histogram
Distribution. Bucket-based.

```
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.5"} 250
http_request_duration_seconds_bucket{le="1.0"} 280
http_request_duration_seconds_bucket{le="+Inf"} 300
http_request_duration_seconds_sum 45.6
http_request_duration_seconds_count 300
```

PromQL:
```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))    # p99
```

Banking:
- `transfer_processing_duration_seconds`
- `db_query_duration_seconds{query="account_lookup"}`
- `external_api_call_duration_seconds{api="bkm"}`

#### Summary
Pre-calculated quantiles. Client-side calc.

Histogram'a karşı: aggregate edilemez (multiple service quantile'lerini birleştirmek matematiksel olarak hatalı). **Banking için Histogram tercih.**

### 3. Micrometer — vendor-neutral metric facade

SLF4J'in metric karşılığı.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Spring Boot Actuator auto-config:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 100ms, 200ms, 500ms, 1s, 2s, 5s
    tags:
      application: ${spring.application.name}
      environment: ${ENV:dev}
      region: ${REGION:eu-central-1}
  prometheus:
    metrics:
      export:
        enabled: true
```

`GET /actuator/prometheus`:
```
http_server_requests_seconds_count{method="GET",status="200",uri="/v1/accounts"} 1234
http_server_requests_seconds_sum{method="GET",status="200",uri="/v1/accounts"} 45.6
http_server_requests_seconds_bucket{le="0.01", ...} 100
...
```

### 4. Micrometer code patterns

#### Counter

```java
@Service
public class TransferService {
    
    private final Counter transfersSuccess;
    private final Counter transfersFailed;
    
    public TransferService(MeterRegistry registry) {
        this.transfersSuccess = Counter.builder("banking.transfers")
            .description("Number of transfers processed")
            .tag("result", "success")
            .baseUnit("transfers")
            .register(registry);
        this.transfersFailed = Counter.builder("banking.transfers")
            .tag("result", "failed")
            .register(registry);
    }
    
    public void transfer(...) {
        try {
            // ... business
            transfersSuccess.increment();
        } catch (Exception e) {
            transfersFailed.increment();
            throw e;
        }
    }
}
```

#### Counter with dynamic tags

Dynamic tag = high cardinality risk. Allowlist:

```java
Counter.Builder builder = Counter.builder("banking.transfers")
    .description("...")
    .tag("currency", currency)    // OK: TRY, USD, EUR (low cardinality)
    .tag("type", type)            // OK: EFT, FAST, SWIFT
    .tag("tenant", tenant);       // OK: TR, DE
    // .tag("userId", userId)     // ❌ HIGH cardinality (millions)
```

Banking cardinality limits:
- `currency`: ~10
- `tenant`: ~5
- `transfer_type`: ~5
- `result`: 2-3 (success/failed/timeout)
- `endpoint`: ~50
- ❌ `userId`, `accountId`, `transferId`: unbounded → DON'T tag

#### Gauge

```java
@Component
public class ActiveSessionGauge {
    
    private final SessionRepository sessionRepo;
    
    public ActiveSessionGauge(MeterRegistry registry, SessionRepository sessionRepo) {
        this.sessionRepo = sessionRepo;
        Gauge.builder("banking.sessions.active", this, g -> g.sessionRepo.countActive())
            .description("Currently active user sessions")
            .register(registry);
    }
}
```

Gauge **lazy evaluation** — Prometheus scrape ettiğinde execute.

#### Timer + Histogram

```java
@Service
public class TransferService {
    
    private final Timer transferTimer;
    
    public TransferService(MeterRegistry registry) {
        this.transferTimer = Timer.builder("banking.transfer.duration")
            .description("Transfer processing time")
            .publishPercentiles(0.5, 0.95, 0.99, 0.999)
            .publishPercentileHistogram()
            .serviceLevelObjectives(
                Duration.ofMillis(100),
                Duration.ofMillis(500),
                Duration.ofSeconds(1),
                Duration.ofSeconds(2))
            .minimumExpectedValue(Duration.ofMillis(1))
            .maximumExpectedValue(Duration.ofSeconds(30))
            .register(registry);
    }
    
    public Transfer transfer(...) {
        return transferTimer.record(() -> {
            // ... business
            return result;
        });
    }
}
```

#### `@Timed` annotation

```java
@RestController
public class TransferController {
    
    @PostMapping("/transfers")
    @Timed(value = "banking.transfer.api", 
           description = "Transfer API duration",
           percentiles = {0.5, 0.95, 0.99})
    public Transfer transfer(...) { ... }
}
```

### 5. RED method — request-driven services

3 metric core:
- **R**ate — request per second
- **E**rrors — error rate
- **D**uration — request latency (p50/p99)

Banking transfer endpoint:
```promql
# Rate
rate(http_server_requests_seconds_count{uri="/v1/transfers"}[5m])

# Errors
rate(http_server_requests_seconds_count{uri="/v1/transfers", status=~"5.."}[5m])
/ rate(http_server_requests_seconds_count{uri="/v1/transfers"}[5m])

# Duration (p99)
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{uri="/v1/transfers"}[5m]))
```

### 6. USE method — resource-driven systems

3 metric core:
- **U**tilization — % busy
- **S**aturation — queue depth
- **E**rrors — error count

Banking DB pool:
```promql
# Utilization
hikaricp_connections_active / hikaricp_connections_max

# Saturation
hikaricp_connections_pending

# Errors
rate(hikaricp_connections_timeout_total[5m])
```

Banking thread pool:
```promql
executor_active_threads / executor_pool_max_threads
executor_queued_tasks
rate(executor_rejected_tasks_total[5m])
```

### 7. Banking domain metrics — beyond RED/USE

**Business metrics (BL):**
```
banking_transfers_total{tenant, currency, type, result}
banking_transfer_amount_total{tenant, currency}    (sum of amounts — careful)
banking_logins_total{result, mfa_used}
banking_logins_failed_total{reason}
banking_mfa_challenges_total{method, result}
banking_cards_blocked_total{reason}
banking_password_reset_total{result}
banking_fraud_alerts_total{severity, type}
```

**Saga metrics:**
```
saga_started_total{saga_type}
saga_completed_total{saga_type, result}
saga_stuck_count{saga_type}    (gauge — stuck for > X seconds)
saga_compensation_total{saga_type}
saga_duration_seconds{saga_type, result}
```

**Outbox metrics:**
```
outbox_pending_count
outbox_published_total
outbox_publish_duration_seconds
outbox_stale_count   (created > 5 min ago, not published — alert!)
```

**Kafka metrics (consumer):**
```
kafka_consumer_lag{topic, group, partition}
kafka_consumer_committed_offset
kafka_consumer_records_consumed_total
kafka_consumer_rebalance_total
```

Banking → Prometheus Kafka exporter ya da `KafkaClientMetrics`.

**External API metrics:**
```
external_api_call_total{provider, endpoint, result}
external_api_call_duration_seconds{provider, endpoint}
external_api_circuit_breaker_state{provider}
external_api_circuit_breaker_calls_total{provider, state}
```

### 8. Spring Boot pre-built metrics

Out-of-the-box:
- `http.server.requests` (timer per endpoint)
- `jvm.memory.used`
- `jvm.gc.pause`
- `jvm.threads.live`
- `process.cpu.usage`
- `system.cpu.usage`
- `hikaricp.connections.active/idle/max`
- `kafka.consumer.records-consumed-rate`
- `spring.security.filterchains` (Spring Security 6)
- `cache.gets/puts/evictions` (Spring Cache)

Banking minimal config exposes hepsi.

### 9. Prometheus setup

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ['9090:9090']
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
      - --storage.tsdb.retention.size=10GB
      - --web.enable-lifecycle    # config reload via API
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: banking-prod
    region: eu-central-1

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'banking-services'
    metrics_path: /actuator/prometheus
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: service
```

K8s annotation pattern:
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

### 10. PromQL banking örnekleri

```promql
# Transfer TPS son 5 dk
sum(rate(banking_transfers_total[5m]))

# Tenant başına
sum by (tenant) (rate(banking_transfers_total[5m]))

# Error rate %
sum(rate(banking_transfers_total{result="failed"}[5m])) 
/ sum(rate(banking_transfers_total[5m])) * 100

# p99 transfer latency
histogram_quantile(0.99, 
  sum by (le) (rate(banking_transfer_duration_seconds_bucket[5m])))

# Apdex score (200ms target, 800ms tolerable)
(
  sum(rate(http_server_requests_seconds_bucket{le="0.2"}[5m])) +
  sum(rate(http_server_requests_seconds_bucket{le="0.8"}[5m])) / 2
) / sum(rate(http_server_requests_seconds_count[5m]))

# DB pool exhaustion alert
hikaricp_connections_active / hikaricp_connections_max > 0.9

# Saga stuck — banking critical
saga_stuck_count > 0

# Outbox stale — dual-write issue
outbox_stale_count > 100

# Kafka consumer lag
sum by (topic, group) (kafka_consumer_lag)

# Top 5 slow endpoints
topk(5, histogram_quantile(0.95, 
  sum by (uri, le) (rate(http_server_requests_seconds_bucket[5m]))))

# Geçen seneye göre transfer volume
sum(rate(banking_transfers_total[7d])) 
/ sum(rate(banking_transfers_total[7d] offset 1y))
```

### 11. SLI / SLO / SLA — banking

| | Anlam | Banking örnek |
|---|---|---|
| **SLI** (Indicator) | Ne ölçüyoruz | "Transfer p99 latency" |
| **SLO** (Objective) | Hedef | "p99 < 500 ms 30 günde" |
| **SLA** (Agreement) | Kontrat (penalty) | "99.9% uptime monthly, missed → service credit" |

Banking SLO örnekleri:
- Login p95 < 1s, %99
- Transfer p99 < 2s, %99
- Account balance p99 < 500ms, %99.9
- Availability monthly > 99.95%
- Error rate < 0.1%

**Error budget:**
```
SLO: 99.9% availability monthly
Total minutes in month: 30 * 24 * 60 = 43200
Allowed downtime: 43200 * 0.001 = 43.2 min/month

→ This is your "error budget"
```

Error budget burn rate alert:
```promql
# Burn rate son 1 saat
(1 - (
  sum(rate(http_server_requests_seconds_count{status!~"5.."}[1h]))
  / sum(rate(http_server_requests_seconds_count[1h]))
)) / 0.001 > 14   # 14x burn → critical
```

### 12. Alertmanager rules — banking

```yaml
# rules/banking-alerts.yml
groups:
  - name: banking-critical
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) 
          / sum(rate(http_server_requests_seconds_count[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
          team: banking-backend
        annotations:
          summary: "Error rate > 1% for 5 minutes"
          description: "Service {{ $labels.service }} error rate: {{ $value | humanizePercentage }}"
          runbook: "https://wiki.bank/runbook/high-error-rate"
      
      - alert: TransferP99Slow
        expr: |
          histogram_quantile(0.99, 
            sum by (le) (rate(banking_transfer_duration_seconds_bucket[5m]))) > 2
        for: 10m
        labels:
          severity: high
          team: banking-backend
      
      - alert: DbPoolExhausted
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.95
        for: 2m
        labels:
          severity: high
      
      - alert: SagaStuck
        expr: saga_stuck_count > 0
        for: 5m
        labels:
          severity: critical
          team: banking-backend
        annotations:
          summary: "Saga stuck for > 5 minutes"
      
      - alert: OutboxStale
        expr: outbox_stale_count > 100
        for: 3m
        labels:
          severity: high
      
      - alert: KafkaConsumerLagHigh
        expr: sum by (topic, group) (kafka_consumer_lag) > 10000
        for: 5m
        labels:
          severity: high
      
      - alert: ErrorBudgetBurnRate
        expr: |
          (1 - (
            sum(rate(http_server_requests_seconds_count{status!~"5.."}[1h]))
            / sum(rate(http_server_requests_seconds_count[1h]))
          )) / 0.001 > 14
        for: 2m
        labels:
          severity: critical
          paging: true
      
      - alert: BruteForceAttack
        expr: rate(banking_logins_failed_total[1m]) > 50
        for: 1m
        labels:
          severity: critical
          team: security
```

Alertmanager → Slack / PagerDuty / Opsgenie.

### 13. Grafana dashboards — banking

**Dashboard 1: Service overview**
- Request rate (RED)
- Error rate %
- p50/p95/p99 latency
- Resource usage (CPU, memory, threads)
- DB pool active/max

**Dashboard 2: Banking business**
- Transfers TPS (by tenant, currency, type)
- Login success/fail
- MFA challenges
- Active sessions
- Fraud alerts
- Card operations

**Dashboard 3: Distributed system**
- Saga stuck / completed / compensated
- Outbox pending / stale
- Kafka lag per topic/group
- Circuit breaker state
- Cross-service trace count

**Dashboard 4: SLO**
- Each SLO % vs target
- Error budget remaining
- Burn rate

**Grafana provisioning** (as code):

```yaml
# dashboards.yml
apiVersion: 1
providers:
  - name: banking
    orgId: 1
    folder: 'Banking'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

JSON dashboard'lar git'te version controlled.

### 14. Banking anti-pattern'leri

**Anti-pattern 1: High cardinality tags**

```java
.tag("userId", userId)   // ❌ Millions of unique values
```

Prometheus memory explosion. **Tag cardinality < 100 (rule of thumb), <10 (banking)**.

**Anti-pattern 2: Sum sensitive amounts in Prometheus**

```java
amountCounter.increment(transferAmount.doubleValue());
```

Pro: Sum görünür.
Con: PII / aggregable data Prometheus'ta.

Banking: Sum log/DB'de, **count + bucket** Prometheus'ta.

**Anti-pattern 3: Histogram bucket'ları yanlış**

Banking transfer 1-30 sn arası:
```
buckets: [0.001, 0.01, 0.1, 1, 10, 100]   # ❌ banking için yanlış
buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5, 10, 30]   # ✓ banking
```

Bucket'lar workload'a göre.

**Anti-pattern 4: Summary aggregate edilebilir sanmak**

Multiple service quantile'leri **toplanmaz**. **Histogram + histogram_quantile** kullan.

**Anti-pattern 5: Counter reset olmaz sanmak**

Process restart counter sıfırlar. `rate()` bunu handle eder. **Ham counter farkı alma.**

**Anti-pattern 6: Push gateway misuse**

Pushgateway sadece **short-lived job** için. Long-running service'ler **pull** model. Banking batch job → pushgateway OK.

**Anti-pattern 7: Tag set production'da değişir**

```java
.tag("env", System.getenv("ENV"))   // Belki dev belki prod
```

Same metric, different tags → series chaos. Tag set sabit.

**Anti-pattern 8: Alert noise**

Critical / high / warning level mix. Banking: critical = page; high = email; warning = dashboard.

**Anti-pattern 9: SLO yok**

"Performance OK" subjektif. SLO + error budget = objective.

**Anti-pattern 10: Scrape interval çok kısa**

`scrape_interval: 1s` → Prometheus storage explode. Banking 15-30s typical.

---

## Önemli olabilecek araştırma kaynakları

- Micrometer documentation
- Prometheus best practices
- Brendan Gregg USE method
- Tom Wilkie RED method
- Google SRE book — SLO/SLI
- Grafana documentation
- BDDK operasyonel risk yönetimi

---

## Mini task'ler

### Task 9.2.1 — Micrometer + Prometheus setup (30 dk)

Dependency, actuator config. `GET /actuator/prometheus` accessible.

### Task 9.2.2 — Banking custom counter + timer (45 dk)

`banking.transfers` counter + `banking.transfer.duration` timer. Tag: tenant, currency, result.

### Task 9.2.3 — Gauge sessions count (30 dk)

Active session sayısı gauge. Lazy evaluation (per scrape).

### Task 9.2.4 — Histogram percentiles (30 dk)

Histogram buckets banking için tune. p50/p95/p99/p999 percentile publish.

### Task 9.2.5 — Prometheus + Grafana local (60 dk)

Docker compose. App scrape config. Grafana datasource Prometheus.

### Task 9.2.6 — Banking dashboard (60 dk)

5+ panel: TPS, error rate, p99 latency, DB pool, JVM memory, saga stuck.

### Task 9.2.7 — Alert rule + Alertmanager (45 dk)

3 alert: HighErrorRate, TransferP99Slow, SagaStuck. Slack webhook (mock).

### Task 9.2.8 — SLO + error budget (45 dk)

1 SLO definition (transfer p99 < 1s, 99.9%). Error budget burn rate query.

### Task 9.2.9 — Kafka consumer lag metric (30 dk)

Kafka consumer KafkaClientMetrics. Lag dashboard.

### Task 9.2.10 — Load test → dashboard observe (60 dk)

JMeter / k6 ile burst. Dashboard'da TPS rise, latency rise, alert fire görmek.

---

## Test yazma rehberi

```java
@SpringBootTest
class MetricsTest {
    
    @Autowired MeterRegistry registry;
    @Autowired TransferService transferService;
    
    @Test
    void shouldIncrementTransferCounter() {
        transferService.transfer(...);
        
        Counter counter = registry.find("banking.transfers")
            .tag("result", "success")
            .counter();
        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(1.0);
    }
    
    @Test
    void shouldRecordTransferDuration() {
        transferService.transfer(...);
        
        Timer timer = registry.find("banking.transfer.duration").timer();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isGreaterThan(0);
    }
    
    @Test
    void shouldExposePrometheusEndpoint() throws Exception {
        mockMvc.perform(get("/actuator/prometheus"))
            .andExpect(status().isOk())
            .andExpect(content().string(containsString("http_server_requests_seconds")));
    }
    
    @Test
    void shouldHavePercentilePublished() throws Exception {
        // After some traffic
        for (int i = 0; i < 100; i++) transferService.transfer(...);
        
        mockMvc.perform(get("/actuator/prometheus"))
            .andExpect(content().string(containsString("quantile=\"0.99\"")));
    }
    
    @Test
    void shouldHaveLowCardinality() {
        // Generate many requests with different userId
        for (int i = 0; i < 100; i++) {
            transferService.transfer(userId: "user-" + i, ...);
        }
        
        // The metric should NOT have user-1, user-2, ..., user-100 as labels
        Collection<Counter> counters = registry.find("banking.transfers").counters();
        assertThat(counters.size()).isLessThan(20);   // Low cardinality
    }
}
```

---

## Claude-verify prompt

```
Metric implementation'ımı banking-grade kriterlere göre değerlendir:

1. Setup:
   - Micrometer + Prometheus registry?
   - Actuator /actuator/prometheus exposed?
   - Common tags (application, environment, region)?

2. Metric types:
   - Counter (banking events)?
   - Gauge (active sessions, balance)?
   - Histogram + percentile_histogram (latency)?
   - Summary NOT preferred (aggregate edilemez)?

3. RED method (services):
   - Rate per endpoint?
   - Errors per endpoint per status?
   - Duration p50/p95/p99/p999?

4. USE method (resources):
   - DB pool utilization/saturation?
   - Thread pool utilization?
   - JVM heap, GC pause?

5. Banking business:
   - banking_transfers_total (tenant, currency, type, result)?
   - banking_logins_total (result, mfa_used)?
   - banking_logins_failed_total (reason)?
   - banking_mfa_challenges_total?
   - banking_fraud_alerts_total?

6. Distributed system:
   - saga_started/completed/stuck/compensation?
   - outbox_pending / outbox_stale?
   - kafka_consumer_lag?
   - external_api_call_duration?
   - circuit_breaker_state?

7. Cardinality:
   - userId/accountId/transferId TAG OLARAK YOK?
   - Tag cardinality < 100 per metric?

8. Histogram buckets:
   - Banking workload'a göre tune (ms ile ölçek)?
   - SLO bucket'ları include (100ms, 200ms, 500ms, 1s, 2s)?

9. Prometheus + Grafana:
   - K8s service discovery scrape?
   - Retention 15-30 gün?
   - Grafana dashboard (RED + USE + business + SLO)?

10. Alerts:
    - Critical (page) vs high (email) vs warning?
    - SLO error budget burn rate?
    - Saga stuck?
    - Brute force detection?
    - Outbox stale (dual-write break)?

11. SLI/SLO:
    - Tanımlı SLO doc?
    - Error budget tracking?
    - Burn rate alert?

12. Anti-pattern:
    - High cardinality tag YOK?
    - Sum sensitive amount in metric YOK?
    - Scrape interval < 5s YOK?
    - Wrong bucket'lar YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Micrometer + Prometheus actuator exposed
- [ ] Banking custom counter + timer + gauge (5+ metric)
- [ ] Tag cardinality kontrol (no userId)
- [ ] Histogram bucket banking-tuned
- [ ] Prometheus scrape config
- [ ] Grafana dashboard banking (10+ panel)
- [ ] 5+ alert rule (HighErrorRate, P99Slow, DbPool, Saga, OutboxStale)
- [ ] SLO + error budget burn rate
- [ ] Kafka consumer lag metric
- [ ] Load test ile dashboard verify
- [ ] 6+ integration test

---

## Defter notları (10 madde)

1. "3 pillar log/metric/trace farkı + retention farkı: ____."
2. "Prometheus 4 metric type + Histogram vs Summary banking pick: ____."
3. "Counter parameter banking (tenant, currency, result, type) cardinality budget: ____."
4. "Timer percentile_histogram + SLO bucket banking: ____."
5. "RED method (Rate, Errors, Duration) banking endpoint: ____."
6. "USE method (Util, Sat, Errors) DB pool + thread pool: ____."
7. "Banking domain metrics (transfers, logins, sagas, outbox, kafka_lag): ____."
8. "PromQL p99 latency + error rate + burn rate query: ____."
9. "SLO/SLI/SLA + error budget burn rate banking: ____."
10. "Alert critical/high/warning + Alertmanager → PagerDuty/Slack: ____."
