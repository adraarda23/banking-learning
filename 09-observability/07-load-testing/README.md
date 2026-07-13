# Topic 9.7 — Load Testing: Gatling, k6, JMeter

## Hedef

Banking endpoint'lerini production-realistic load test etmek: Gatling Scala DSL, k6 JavaScript, JMeter karşılaştırma. Test type'lar (smoke, load, stress, spike, soak), ramp-up/steady/ramp-down, banking workload modelleme, observability stack ile entegrasyon, breaking point analizi, capacity planning.

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 9.1-9.6 bitti (load test gerçek değer ancak observable system'de)
- HTTP, REST, JSON
- Statistical p50/p95/p99 anlamı

---

## Kavramlar

### 1. Load test types

| Test type | Hedef | Süre | Banking use case |
|---|---|---|---|
| **Smoke** | Build doğrulama | 1-5 dk | CI/CD her PR |
| **Load** | Beklenen yük altında perf | 30-60 dk | Pre-release |
| **Stress** | Breaking point bul | 1-2 saat | Capacity planning |
| **Spike** | Ani burst (FAST kampanyası, maaş günü) | 5-15 dk | Resilience |
| **Soak** | Uzun süre, memory leak | 4-24 saat | Production-readiness |
| **Volume** | Big data input | Datadependent | Batch (Phase 5) |
| **Chaos** | Failure injection | Variable | Resilience (Phase 7) |

Banking için **soak test özellikle önemli** — overnight batch periodları, memory leak detection.

### 2. Workload modeling — banking realistic

Banking traffic patterns:
- **Closed model**: Sabit kullanıcı sayısı (login session) → think time + interaction
- **Open model**: Sabit request rate (per-second TPS) → bağımsız arrival

Production banking genelde **open model** (REST API, event-driven).

**Mix realistic:**
```
50% account.read (balance check)
20% transactions.read
15% transfer.write
10% card.read
3% login
2% other
```

**Peak patterns:**
- Maaş günü (ayın 1-15. günü) → 5x TPS
- Festival eve (kurban bayramı arefe) → spike
- 9-17 iş saatleri → 3x off-hours
- Tax deadline → batch + interactive mix

### 3. Gatling — Scala DSL

Industry banking favorite. High-performance (NIO).

```scala
// BankingSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class BankingSimulation extends Simulation {
  
  val httpProtocol = http
    .baseUrl("https://api.mavibank.com")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .userAgentHeader("Gatling/Banking-LoadTest")
  
  val authFeeder = csv("users.csv").circular
  // username,password
  
  val tokens = scenario("Login").exec(
    feed(authFeeder),
    http("Login")
      .post("/v1/auth/login")
      .body(StringBody("""{"username":"${username}","password":"${password}"}"""))
      .check(jsonPath("$.access_token").saveAs("token"))
  )
  
  val accountRead = scenario("Account Read")
    .exec(tokens)
    .repeat(10) {
      exec(
        http("Account Balance")
          .get("/v1/accounts/me/balance")
          .header("Authorization", "Bearer ${token}")
          .check(status.is(200))
          .check(jsonPath("$.balance").exists)
      ).pause(1, 3)
    }
  
  val transfer = scenario("Transfer")
    .exec(tokens)
    .exec(
      http("Initiate Transfer")
        .post("/v1/transfers")
        .header("Authorization", "Bearer ${token}")
        .header("X-Idempotency-Key", "${randomUuid()}")
        .body(StringBody("""{
          "fromAccount": "${fromAccount}",
          "toIban": "TR${randomNumeric(24)}",
          "amount": "${randomFloat(100,5000)}",
          "currency": "TRY",
          "description": "Load test"
        }"""))
        .check(status.in(200, 201))
        .check(jsonPath("$.transferId").saveAs("transferId"))
    )
    .pause(1, 5)
    .exec(
      http("Check Transfer Status")
        .get("/v1/transfers/${transferId}")
        .header("Authorization", "Bearer ${token}")
        .check(status.is(200))
    )
  
  setUp(
    accountRead.inject(
      // Ramp-up: 0 → 200 users in 60 seconds
      rampUsersPerSec(0).to(200).during(60.seconds),
      // Steady: 200 RPS for 5 minutes
      constantUsersPerSec(200).during(5.minutes),
      // Ramp-down
      rampUsersPerSec(200).to(0).during(30.seconds)
    ),
    transfer.inject(
      rampUsersPerSec(0).to(50).during(60.seconds),
      constantUsersPerSec(50).during(5.minutes)
    )
  ).protocols(httpProtocol)
    .assertions(
      global.responseTime.percentile3.lt(500),   // p95 < 500 ms
      global.responseTime.percentile4.lt(2000),  // p99 < 2000 ms
      global.failedRequests.percent.lt(0.5)      // error rate < 0.5%
    )
}
```

Run:
```bash
mvn gatling:test
# or
./gatling.sh -s BankingSimulation
```

Output: HTML report (response times, distribution, error breakdown).

### 4. k6 — JavaScript DSL (modern favorite)

```javascript
// banking-load.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Trend, Rate, Counter } from 'k6/metrics';
import { randomString, randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

const errorRate = new Rate('errors');
const transferDuration = new Trend('transfer_duration');
const transfersTotal = new Counter('transfers_total');

export const options = {
  scenarios: {
    account_read: {
      executor: 'ramping-arrival-rate',
      startRate: 0,
      timeUnit: '1s',
      preAllocatedVUs: 100,
      maxVUs: 500,
      stages: [
        { duration: '60s', target: 200 },
        { duration: '5m', target: 200 },
        { duration: '30s', target: 0 },
      ],
    },
    transfer: {
      executor: 'ramping-arrival-rate',
      startRate: 0,
      timeUnit: '1s',
      preAllocatedVUs: 50,
      maxVUs: 200,
      stages: [
        { duration: '60s', target: 50 },
        { duration: '5m', target: 50 },
        { duration: '30s', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<2000'],
    errors: ['rate<0.005'],
    transfer_duration: ['p(99)<3000'],
  },
};

const BASE = 'https://api.mavibank.com';

function login() {
  const res = http.post(`${BASE}/v1/auth/login`, JSON.stringify({
    username: 'user' + randomIntBetween(1, 1000),
    password: 'TestPass123!'
  }), { headers: { 'Content-Type': 'application/json' }});
  
  check(res, { 'login success': r => r.status === 200 });
  return res.json('access_token');
}

export default function () {
  const token = login();
  const headers = {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  };
  
  group('account', () => {
    const balance = http.get(`${BASE}/v1/accounts/me/balance`, { headers });
    check(balance, {
      'balance 200': r => r.status === 200,
      'balance has field': r => r.json('balance') !== undefined,
    });
    errorRate.add(balance.status !== 200);
    sleep(1);
  });
  
  group('transfer', () => {
    const start = Date.now();
    const body = JSON.stringify({
      fromAccount: 'acc-' + randomIntBetween(1, 1000),
      toIban: 'TR' + randomString(24, '0123456789'),
      amount: randomIntBetween(100, 5000),
      currency: 'TRY',
      description: 'Load test',
    });
    const transfer = http.post(`${BASE}/v1/transfers`, body, { 
      headers: { ...headers, 'X-Idempotency-Key': randomString(36) }
    });
    
    transferDuration.add(Date.now() - start);
    transfersTotal.add(1);
    
    check(transfer, {
      'transfer 200/201': r => r.status === 200 || r.status === 201,
      'has transferId': r => r.json('transferId') !== undefined,
    });
    errorRate.add(transfer.status >= 400);
    sleep(2);
  });
}
```

Run:
```bash
k6 run banking-load.js

# With output to Prometheus
k6 run --out experimental-prometheus-rw=http://prometheus:9090/api/v1/write banking-load.js

# Distributed (k6 cloud or operator)
k6 cloud banking-load.js
```

### 5. JMeter — legacy but ubiquitous

GUI-based (XML), Java. Banking için legacy systems'te yaygın.

**Strengths:**
- Plugin ecosystem
- Distributed mode
- Visual test plan

**Weaknesses:**
- Heavy memory
- GUI ≠ load test runner (CLI mode)
- JS DSL yok (XML)

Modern preferences: **k6 (script-first), Gatling (high perf), JMeter (legacy compat)**.

### 6. Test scenario design — banking

**Smoke** — CI'da her PR:
```
1 user, 1 request per endpoint, 10 second
Assert: 200 OK, < 1s
Goal: Build sanity
```

**Load** — pre-release:
```
Ramp 0 → 500 RPS in 5 min
Steady 500 RPS for 30 min
Ramp 500 → 0 in 2 min
Assert: p95 < 500ms, error rate < 0.5%
Goal: Performance regression check
```

**Stress** — find breakpoint:
```
Ramp 100 RPS → 5000 RPS over 1 hour
Continue until: error rate > 5% OR p99 > 10s
Record: breaking point
Goal: Capacity planning
```

**Spike** — sudden burst:
```
Steady 100 RPS for 5 min
Spike to 2000 RPS for 1 min
Drop to 100 RPS for 5 min
Assert: Recover within 30s, no data loss
Goal: Resilience under burst (FAST campaign, maaş)
```

**Soak** — overnight:
```
Constant 200 RPS for 8 hours
Watch: Memory, GC, connection pool over time
Goal: Memory leak detection (Topic 9.5)
```

### 7. Observability tie-in

Load test alone = numbers. **Observability ile birlikte** = root cause.

```
Run k6 load test
   ↓
Live monitoring:
  - Grafana: TPS, error rate, p99 latency, CPU, memory, DB pool
  - Jaeger: trace exemplars (slow requests)
  - JFR: continuous profile (Topic 9.4)
   ↓
Identify bottleneck:
  - "p99 spike at 800 RPS"
  - Profile shows BigDecimal.divide hot
  - Trace shows DB query 1.5s
  - Metric shows hikari pool exhausted
   ↓
Fix → re-run load test
```

### 8. Banking realistic data

**Pre-seed test data:**
```sql
-- 1M users
INSERT INTO customers SELECT generate_series, ... FROM generate_series(1, 1_000_000);
-- 5M accounts
INSERT INTO accounts ...;
-- 100M transactions (historical)
INSERT INTO transactions ...;
```

**User credentials feed:**
```csv
# users.csv (Gatling) — 100k pre-created users
username,password
user1,TestPass123!
user2,TestPass123!
...
```

**Random data — realistic distribution:**

```javascript
// k6 — exponential think time
function thinkTime() {
  return Math.random() < 0.7 ? randomIntBetween(1, 3) : randomIntBetween(5, 30);
  // 70% fast, 30% slow user
}

// Banking amount distribution (long-tail)
function amount() {
  if (Math.random() < 0.6) return randomIntBetween(50, 500);     // 60% small
  if (Math.random() < 0.3) return randomIntBetween(500, 5000);   // 30% medium
  if (Math.random() < 0.09) return randomIntBetween(5000, 50000); // 9% large
  return randomIntBetween(50000, 200000);                          // 1% very large
}
```

### 9. Banking specific — idempotency under load

Load test idempotency'i de test eder.

```javascript
const idempotencyKey = randomString(36);

// Send 3x same request (simulating client retry)
for (let i = 0; i < 3; i++) {
  http.post(`${BASE}/v1/transfers`, body, {
    headers: { 'X-Idempotency-Key': idempotencyKey }
  });
}

// Verify: Only 1 transfer record created
const check = http.get(`${BASE}/v1/transfers?idempotency_key=${idempotencyKey}`);
check.equals(check.json('count'), 1);
```

### 10. Banking session lifecycle

Realistic user flow (not isolated endpoint):

```javascript
export default function () {
  // Login
  const token = login();
  
  // Browse
  http.get('/accounts/me', { headers });
  sleep(2);
  http.get('/accounts/me/transactions', { headers });
  sleep(3);
  
  // Transfer
  http.post('/transfers', body, { headers });
  sleep(5);
  
  // Logout
  http.post('/auth/logout', null, { headers });
}
```

100% transfer endpoint alone = unrealistic. Banking real users browse + occasional action.

### 11. Result interpretation

**Gatling report:**
- Response time distribution histogram
- p50, p75, p95, p99
- Active users over time
- Errors per second
- Response codes breakdown

**Banking signals:**

| Signal | Interpretation |
|---|---|
| p50 stable, p99 spike | Tail latency — GC pause, lock contention |
| Error rate climb after threshold | Resource exhaustion (DB pool, thread) |
| Latency stable but TPS plateau | Throughput ceiling — vertical scale |
| Memory grow over time | Memory leak (run soak) |
| GC frequency 10x | Allocation pressure (Topic 9.4) |
| 504 timeout | Upstream service slow |
| 503 | Service unavailable (deliberate or kub eviction) |

### 12. CI integration

```yaml
# .github/workflows/load-test.yml
name: Nightly Load Test

on:
  schedule:
    - cron: '0 2 * * *'   # 02:00 UTC nightly
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: kubectl apply -k k8s/staging
      
      - name: Wait healthy
        run: kubectl wait --for=condition=ready pod -l app=banking-api --timeout=300s
      
      - name: Run k6 load test
        run: |
          docker run --rm -v $(pwd):/scripts grafana/k6 run /scripts/banking-load.js
      
      - name: Upload result
        uses: actions/upload-artifact@v3
        with:
          name: load-test-result
          path: results/
      
      - name: Compare to baseline
        run: python compare_load.py baseline.json result.json
      
      - name: Slack notification
        if: failure()
        uses: 8398a7/action-slack@v3
```

### 13. Banking — load test anti-pattern'leri

**Anti-pattern 1: Load test on shared infrastructure**

Other apps interfere. **Dedicated staging environment** banking için.

**Anti-pattern 2: Localhost load test**

NIC limits, no network latency. Distant runner (realistic).

**Anti-pattern 3: Synthetic clean data**

100k same user = cache hit ratio gerçek değil. Realistic distribution.

**Anti-pattern 4: No think time**

Open hammer → unrealistic. Banking user 2-30 sn think.

**Anti-pattern 5: Single endpoint hammer**

`POST /transfers` only. Real users browse first.

**Anti-pattern 6: Run once, declare done**

Variance. 3 run + statistical.

**Anti-pattern 7: Ignore observability**

Number only. Profile/trace ile root cause.

**Anti-pattern 8: Production load test without coordination**

Banking için coordinate'siz production load = customer impact + chaos. **Synthetic traffic** marker'lı, isolated path.

**Anti-pattern 9: No baseline**

Comparison'sız "fast enough" subjektif. Baseline + CI track.

**Anti-pattern 10: Load test = correctness test**

Load test perf'i ölçer. Correctness ayrı (integration test). Phase 12'de.

---

## Önemli olabilecek araştırma kaynakları

- Gatling docs
- k6 docs
- JMeter user manual
- "Web Scalability for Startup Engineers" — Artur Ejsmont
- Brendan Gregg load test methodology
- Google SRE workbook — load testing

---

## Mini task'ler

### Task 9.7.1 — k6 smoke test (30 dk)

Banking API health endpoint k6 ile 1 user, 10 sec. CI'da PR check.

### Task 9.7.2 — Gatling banking simulation (60 dk)

Login + account read + transfer scenario. Ramp + steady + ramp-down.

### Task 9.7.3 — Load test pre-release (60 dk)

500 RPS steady, 30 min. SLO assert (p99 < 1s, error < 0.5%).

### Task 9.7.4 — Stress test breakpoint (60 dk)

100 → 5000 RPS ramp. Breaking point bul.

### Task 9.7.5 — Spike test (45 dk)

100 → 2000 → 100 RPS. Recovery check. CB / autoscale verify.

### Task 9.7.6 — Soak test 4 saat (overnight) (background)

Constant 200 RPS, 4-8 saat. Memory grow check. Topic 9.5'ten heap dump compare.

### Task 9.7.7 — Idempotency under load (45 dk)

Same idempotency_key 3x → 1 transfer assert.

### Task 9.7.8 — Observability tie (60 dk)

Load test sırasında Grafana monitor + Jaeger trace + JFR profile. Root cause walkthrough.

### Task 9.7.9 — Realistic data distribution (45 dk)

Pareto / log-normal amount distribution. Think time exponential. Verify.

### Task 9.7.10 — CI nightly load test (60 dk)

GitHub Actions schedule + staging deploy + k6 + result archive + Slack notify.

---

## Test yazma rehberi

```java
// JUnit-driven k6 invocation for CI
@Test
@Tag("load")
void smokeTest() throws Exception {
    Process k6 = new ProcessBuilder("k6", "run", "smoke.js")
        .inheritIO()
        .start();
    int exitCode = k6.waitFor();
    assertThat(exitCode).isEqualTo(0);   // k6 exits non-zero on threshold fail
}

@Test
@Tag("load")
void loadTestWithThresholds() throws Exception {
    // Trigger Gatling via Maven plugin
    InvocationRequest req = new DefaultInvocationRequest();
    req.setPomFile(new File("pom.xml"));
    req.setGoals(List.of("gatling:test"));
    req.addArg("-Dgatling.simulationClass=com.bank.BankingLoadSimulation");
    
    Invoker invoker = new DefaultInvoker();
    InvocationResult result = invoker.execute(req);
    
    assertThat(result.getExitCode()).isEqualTo(0);
}
```

CI:
```bash
mvn test -Dgroups=load
```

---

## Claude-verify prompt

```
Load testing setup'ımı banking-grade kriterlere göre değerlendir:

1. Tool selection:
   - Gatling / k6 / JMeter pick rationale?
   - CI integration?

2. Test types:
   - Smoke (CI per PR)?
   - Load (pre-release)?
   - Stress (capacity planning)?
   - Spike (resilience)?
   - Soak (memory leak)?
   - Banking için tümü kapsanmış?

3. Workload modeling:
   - Realistic endpoint mix (50/20/15/10/5)?
   - Peak pattern (maaş günü, festival)?
   - Open model (TPS) banking için?
   - Think time exponential?
   - Amount distribution Pareto?

4. Banking specific:
   - Idempotency under load test?
   - Login → browse → action flow?
   - 100k pre-seeded users?
   - X-Idempotency-Key header?

5. SLO assertions:
   - p95 < 500ms threshold?
   - p99 < 2s threshold?
   - Error rate < 0.5%?
   - CI fail on regression?

6. Observability tie:
   - Live Grafana monitor during test?
   - Jaeger exemplar trace?
   - JFR continuous profile?
   - Root cause workflow documented?

7. Environment:
   - Dedicated staging (shared YOK)?
   - Realistic network latency?
   - Production-similar data volume?
   - CI runner dedicated?

8. CI integration:
   - Smoke per PR?
   - Load nightly?
   - Soak weekly?
   - Result archive + baseline diff?
   - Slack notification?

9. Stress / spike:
   - Breaking point recorded?
   - Recovery from spike measured?
   - Circuit breaker verify (Phase 7)?
   - Autoscale trigger verify?

10. Anti-pattern:
    - Localhost test YOK?
    - Synthetic clean data YOK?
    - No think time YOK?
    - Single endpoint hammer YOK?
    - Production load test uncoordinated YOK?
    - No baseline comparison YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] k6 / Gatling kurulu + simulation script
- [ ] Smoke + Load + Stress + Spike + Soak script
- [ ] Banking realistic workload (endpoint mix, think time, distribution)
- [ ] Idempotency-under-load scenario
- [ ] SLO assertion (p95, p99, error rate)
- [ ] Observability tie (Grafana monitor)
- [ ] Trace exemplar drill-down workflow
- [ ] CI integration (PR smoke + nightly load + weekly soak)
- [ ] Baseline + regression detection
- [ ] 1 root-cause analysis walkthrough
- [ ] 10 mini-task done

---

## Defter notları (10 madde)

1. "Load test type'ları (smoke/load/stress/spike/soak/volume) banking use case: ____."
2. "Open vs closed workload model + banking REST API açık: ____."
3. "Realistic endpoint mix + think time + amount distribution banking: ____."
4. "Gatling Scala DSL strength + k6 JavaScript modern + JMeter legacy: ____."
5. "SLO threshold assertion CI gate (p95, p99, error rate): ____."
6. "Load test + observability tie (Grafana + Jaeger + JFR): ____."
7. "Banking session lifecycle (login → browse → action → logout) realistic: ____."
8. "Idempotency under load (same key 3x = 1 record): ____."
9. "Stress breaking point + spike recovery + CB/autoscale verify: ____."
10. "CI integration nightly + baseline comparison + Slack notify: ____."
