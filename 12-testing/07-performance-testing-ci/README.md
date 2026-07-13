# Topic 12.7 — Performance Testing in CI

## Hedef

CI/CD pipeline'da performance regression yakalama: Gatling/k6 integration, JMH benchmark CI, baseline comparison, threshold gate, dedicated runner, banking SLO-driven test plan, PR comment automation, multi-environment perf strategy.

## Süre

Okuma: 1.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 9.6 (JMH Deep) bitti
- Topic 9.7 (Load Testing) bitti
- Topic 11.4 (CI/CD) bitti

---

## Kavramlar

### 1. Perf testing — pipeline placement

```
PR-time:
  - Smoke test (1 user, 10 sec) — sanity
  - JMH micro benchmark (changed code)

Pre-merge:
  - Quality gate (Sonar)
  - SAST/SCA

Nightly:
  - Load test (steady 30 min)
  - Soak test (background)

Weekly:
  - Stress test (breakpoint)
  - Spike test (resilience)

Pre-release:
  - Full perf suite
  - Comparison to last GA
```

### 2. JMH in CI — PR regression detect

(Topic 9.6'dan tekrar)

```yaml
jmh-benchmark:
  runs-on: banking-benchmark-runner   # Dedicated, stable
  steps:
    - uses: actions/checkout@v4
    
    - name: Run benchmarks (PR branch)
      run: |
        ./mvnw -B clean package -DskipTests
        java -jar target/benchmarks.jar \
          -rf json -rff pr-result.json \
          -wi 5 -i 10 -f 2
    
    - name: Get baseline (main)
      run: |
        git fetch origin main
        git checkout origin/main
        ./mvnw -B clean package -DskipTests
        java -jar target/benchmarks.jar \
          -rf json -rff main-result.json \
          -wi 5 -i 10 -f 2
    
    - name: Compare & gate
      run: |
        python scripts/compare-benchmark.py \
          --baseline main-result.json \
          --pr pr-result.json \
          --threshold 0.10 \
          --output diff.md
        cat diff.md
    
    - name: PR comment
      uses: actions/github-script@v7
      with:
        script: |
          const fs = require('fs');
          const diff = fs.readFileSync('diff.md', 'utf8');
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '## Benchmark Result\n\n' + diff
          });
```

`compare-benchmark.py`:

```python
#!/usr/bin/env python3
import json, sys, argparse

ap = argparse.ArgumentParser()
ap.add_argument('--baseline', required=True)
ap.add_argument('--pr', required=True)
ap.add_argument('--threshold', type=float, default=0.10)
ap.add_argument('--output')
args = ap.parse_args()

baseline = json.load(open(args.baseline))
pr = json.load(open(args.pr))

baseline_by_name = {b['benchmark']: b for b in baseline}
pr_by_name = {p['benchmark']: p for p in pr}

table = "| Benchmark | Baseline (ns/op) | PR (ns/op) | Δ% | Status |\n"
table += "|---|---|---|---|---|\n"

failures = []

for name, pr_data in pr_by_name.items():
    pr_score = pr_data['primaryMetric']['score']
    baseline_data = baseline_by_name.get(name)
    
    if not baseline_data:
        table += f"| {name} | NEW | {pr_score:.2f} | - | ⊕ new |\n"
        continue
    
    baseline_score = baseline_data['primaryMetric']['score']
    delta = (pr_score - baseline_score) / baseline_score
    
    status = "✓"
    if delta > args.threshold:
        status = "❌ REGRESSION"
        failures.append((name, delta * 100))
    elif delta < -0.05:
        status = "⚡ improved"
    
    table += f"| {name} | {baseline_score:.2f} | {pr_score:.2f} | {delta*100:+.1f}% | {status} |\n"

if args.output:
    with open(args.output, 'w') as f:
        f.write(table)

print(table)

if failures:
    print("\nRegressions detected:")
    for name, pct in failures:
        print(f"  {name}: +{pct:.1f}%")
    sys.exit(1)
```

PR comment example:
```
## Benchmark Result

| Benchmark | Baseline (ns/op) | PR (ns/op) | Δ% | Status |
|---|---|---|---|---|
| moneyAdd | 45.23 | 47.10 | +4.1% | ✓ |
| moneyMultiply | 120.45 | 119.80 | -0.5% | ✓ |
| ledgerPost | 8500.12 | 9800.34 | +15.3% | ❌ REGRESSION |
```

### 3. k6 smoke test in CI

```yaml
smoke-test:
  runs-on: ubuntu-latest
  needs: [build, deploy-pr-preview]
  steps:
    - uses: actions/checkout@v4
    
    - name: k6 smoke
      uses: grafana/k6-action@v0.3.1
      with:
        filename: tests/perf/smoke.js
        flags: --env BASE_URL=${{ env.PR_PREVIEW_URL }}
    
    - name: Archive results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: k6-smoke-${{ github.sha }}
        path: results.json
```

```javascript
// tests/perf/smoke.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 1,
  duration: '30s',
  thresholds: {
    http_req_duration: ['p(95)<1000'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const baseUrl = __ENV.BASE_URL;
  const health = http.get(`${baseUrl}/actuator/health`);
  check(health, { 'health up': r => r.status === 200 });
}
```

### 4. Nightly load test workflow

```yaml
# .github/workflows/nightly-load-test.yml
name: Nightly Load Test

on:
  schedule:
    - cron: '0 2 * * *'   # 02:00 UTC
  workflow_dispatch:

jobs:
  load-test:
    runs-on: banking-perf-runner
    timeout-minutes: 60
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy staging refresh
        run: kubectl apply -k k8s/staging
      
      - name: Wait healthy
        run: kubectl wait --for=condition=ready pod -l app=banking-api -n staging --timeout=300s
      
      - name: Seed test data
        run: ./scripts/seed-perf-data.sh staging
      
      - name: Run k6 load test
        run: |
          docker run --rm -v $(pwd):/scripts grafana/k6 run \
            -e BASE_URL=https://staging.mavibank.com \
            --out json=results.json \
            /scripts/tests/perf/load.js
      
      - name: Analyze + threshold
        run: |
          python scripts/analyze-load-test.py results.json
      
      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: load-test-${{ github.run_number }}
          path: |
            results.json
            report.html
      
      - name: Compare to baseline (last successful run)
        run: |
          python scripts/compare-load.py \
            --baseline baseline.json \
            --current results.json \
            --threshold 0.10
      
      - name: Slack
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#banking-perf'
```

### 5. Banking SLO-driven test plan

Tests align with SLO definitions:

```yaml
# slo.yml
slos:
  transfer-service:
    - name: transfer_p99_latency
      target: 1000ms
      query: histogram_quantile(0.99, ...)
    - name: transfer_error_rate
      target: 0.005
      query: rate(...{status=~"5.."}[5m]) / rate(...[5m])
    - name: transfer_throughput
      target: 1000 rps
      query: rate(...[5m])
```

```javascript
// tests/perf/load.js
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const transferDuration = new Trend('banking_transfer_duration');
const transferErrors = new Rate('banking_transfer_errors');

export const options = {
  scenarios: {
    transfer_load: {
      executor: 'ramping-arrival-rate',
      startRate: 0,
      timeUnit: '1s',
      preAllocatedVUs: 200,
      maxVUs: 1000,
      stages: [
        { duration: '5m', target: 1000 },   // ramp to 1000 RPS
        { duration: '30m', target: 1000 },  // sustain
        { duration: '5m', target: 0 },
      ],
    },
  },
  thresholds: {
    banking_transfer_duration: ['p(99)<1000'],   // SLO match
    banking_transfer_errors: ['rate<0.005'],     // SLO match
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const start = Date.now();
  const res = http.post('https://staging/v1/transfers/havale', body, headers);
  
  transferDuration.add(Date.now() - start);
  transferErrors.add(res.status >= 400);
  
  check(res, {
    'status 200/201': r => r.status === 200 || r.status === 201,
  });
}
```

### 6. Banking realistic load model

```javascript
import { exec } from 'k6/execution';
import { randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

function bankingThinkTime() {
  // 70% quick (1-3 sec), 25% medium (5-15), 5% slow (30+)
  const r = Math.random();
  if (r < 0.70) return randomIntBetween(1, 3);
  if (r < 0.95) return randomIntBetween(5, 15);
  return randomIntBetween(30, 90);
}

function bankingAmount() {
  // Pareto-like: most small, few large
  const r = Math.random();
  if (r < 0.60) return randomIntBetween(50, 500);
  if (r < 0.90) return randomIntBetween(500, 5000);
  if (r < 0.99) return randomIntBetween(5000, 50000);
  return randomIntBetween(50000, 200000);
}

function bankingEndpointMix() {
  // Real production traffic mix
  const r = Math.random();
  if (r < 0.50) return 'account-balance';
  if (r < 0.75) return 'transaction-history';
  if (r < 0.90) return 'transfer';
  if (r < 0.97) return 'card-info';
  return 'profile';
}
```

### 7. Continuous benchmarking dashboard

Push benchmark results to long-term store:

```python
# scripts/push-to-grafana.py
import json, requests

results = json.load(open('pr-result.json'))

for r in results:
    name = r['benchmark']
    score = r['primaryMetric']['score']
    error = r['primaryMetric']['scoreError']
    
    requests.post('http://victoria-metrics:8428/api/v1/import',
        json={
            'metric': {'__name__': 'jmh_benchmark', 'benchmark': name},
            'values': [score],
            'timestamps': [int(time.time() * 1000)]
        })
```

Grafana panel: benchmark trend over months. Identify silent regressions.

### 8. Pre-release perf gate

Critical release (major version):

```yaml
- name: Full perf suite
  run: |
    # 1. Smoke
    k6 run tests/perf/smoke.js
    
    # 2. Load 30 min
    k6 run tests/perf/load.js
    
    # 3. Stress to find breakpoint
    k6 run tests/perf/stress.js
    
    # 4. Spike resilience
    k6 run tests/perf/spike.js
    
    # 5. Soak 4 hours
    k6 run tests/perf/soak.js

- name: Compare to last GA
  run: python scripts/compare-perf.py --baseline ga-1.0.json --current run.json

- name: Release decision
  run: |
    if [ "$REGRESSION" = "true" ]; then
      echo "❌ Performance regression — release blocked"
      exit 1
    fi
```

### 9. Dedicated CI runner — stable env

```yaml
# self-hosted runner config
labels: [self-hosted, banking-benchmark-runner]

# Setup:
# - bare metal (no virtualization noise)
# - CPU pinning (taskset)
# - turbo boost disabled
# - no other workloads
# - same JVM version
# - same OS

# Pre-run script
cpu_setup() {
  echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
  sudo cpupower frequency-set -g performance
  for cpu in 0-3; do
    echo 1 | sudo tee /sys/devices/system/cpu/cpu${cpu}/online
  done
  for cpu in 4-7; do
    echo 0 | sudo tee /sys/devices/system/cpu/cpu${cpu}/online
  done
}
```

Banking dedicated runner critical — shared CI = noise = false positives.

### 10. Banking — perf testing CI anti-pattern'leri

**Anti-pattern 1: Performance test only manual**
- Regression slips. CI gate.

**Anti-pattern 2: Shared CI runner**
- Noise. Dedicated.

**Anti-pattern 3: Hardcoded threshold**
- Code improves → threshold lowered → no detection. Adaptive (% of baseline).

**Anti-pattern 4: Smoke test only**
- Not enough. Load + soak + stress.

**Anti-pattern 5: Test in production**
- Real customer impact. Staging realistic clone.

**Anti-pattern 6: Compare absolute numbers**
- Env-dependent. Compare relative to baseline same env.

**Anti-pattern 7: No baseline storage**
- Each run isolated. Trend lost. Time-series store.

**Anti-pattern 8: Single warmup iteration**
- JIT not stable. Min 5 warmup iterations.

**Anti-pattern 9: Test in main thread**
- Non-deterministic. Dedicated JMH config.

**Anti-pattern 10: Threshold too lax**
- 50% regression still passes. Banking: 10% threshold standard.

---

## Önemli olabilecek araştırma kaynakları

- "Java Performance" — Scott Oaks
- JMH official samples
- k6 documentation
- Grafana Cloud k6 (managed)
- Continuous benchmark patterns

---

## Mini task'ler

### Task 12.7.1 — JMH PR vs main comparison (60 dk)

GitHub Actions workflow. Compare script. PR comment.

### Task 12.7.2 — k6 smoke in CI (30 dk)

PR preview deploy → k6 smoke → pass/fail.

### Task 12.7.3 — Nightly load test schedule (45 dk)

Cron 02:00 UTC. Staging → k6 load → results → Slack.

### Task 12.7.4 — Banking realistic load model (45 dk)

Endpoint mix + think time + amount distribution.

### Task 12.7.5 — SLO-driven threshold (30 dk)

`http_req_duration: p(99)<1000` (SLO match).

### Task 12.7.6 — Baseline comparison script (60 dk)

Python compare. Threshold 10% regression fail.

### Task 12.7.7 — Long-term trend store (30 dk)

Push to Prometheus / Victoria Metrics. Grafana benchmark panel.

### Task 12.7.8 — Dedicated runner script (45 dk)

CPU pinning + turbo off + governor performance.

### Task 12.7.9 — Pre-release full perf suite (60 dk)

Smoke + load + stress + spike + soak chained.

### Task 12.7.10 — Banking SLO config-as-code (30 dk)

slo.yml + threshold extraction + dashboard.

---

## Test yazma rehberi

```yaml
# .github/workflows/perf-gate.yml
name: Performance Gate

on:
  pull_request:
    branches: [main]

jobs:
  jmh:
    runs-on: banking-benchmark-runner
    steps:
      - uses: actions/checkout@v4
      - name: Setup
        run: |
          ./scripts/cpu-setup.sh
      
      - name: Run PR benchmarks
        run: |
          ./mvnw -B clean package -DskipTests
          java -XX:+UnlockDiagnosticVMOptions -XX:-TieredCompilation \
            -jar target/benchmarks.jar \
            -wi 5 -i 10 -f 3 \
            -rf json -rff pr.json
      
      - name: Get baseline
        run: |
          git fetch origin main
          git checkout origin/main
          ./mvnw -B clean package -DskipTests
          java -XX:+UnlockDiagnosticVMOptions -XX:-TieredCompilation \
            -jar target/benchmarks.jar \
            -wi 5 -i 10 -f 3 \
            -rf json -rff main.json
      
      - name: Compare
        run: python scripts/compare.py main.json pr.json --threshold 0.10
      
      - name: Comment PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const diff = require('fs').readFileSync('diff.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: diff
            });

  smoke:
    runs-on: ubuntu-latest
    needs: jmh
    steps:
      - uses: actions/checkout@v4
      - run: docker run --rm -v $(pwd):/scripts grafana/k6 run /scripts/tests/perf/smoke.js
```

---

## Claude-verify prompt

```
Performance testing CI setup'ımı banking-grade kriterlere göre değerlendir:

1. JMH CI:
   - PR vs main baseline?
   - Regression threshold 10%?
   - PR comment auto?
   - Dedicated runner?

2. Smoke test:
   - Per PR sanity?
   - SLO-matching threshold?

3. Nightly load:
   - Schedule (02:00 UTC)?
   - Staging refresh + seed?
   - 30-60 min sustained?
   - Slack on fail?

4. Pre-release:
   - Full suite (smoke + load + stress + spike + soak)?
   - Comparison to last GA?
   - Release block on regression?

5. Banking realistic:
   - Endpoint mix realistic?
   - Think time exponential?
   - Amount Pareto?
   - SLO-driven thresholds?

6. Baseline management:
   - Long-term store (Prometheus/Victoria)?
   - Trend visible?
   - Adaptive threshold (% of baseline)?

7. CI runner:
   - Dedicated (no shared)?
   - CPU pinning?
   - Turbo off?
   - Performance governor?

8. Documentation:
   - SLO config-as-code?
   - Runbook for failure?

9. Anti-pattern:
   - Shared runner YOK?
   - Hardcoded absolute threshold YOK?
   - Smoke only (no load) YOK?
   - No baseline storage YOK?
   - Single warmup YOK?
   - Test in production YOK?

10. Banking domain:
    - Currency BigDecimal benchmark?
    - Ledger throughput benchmark?
    - Concurrent transfer (lock) benchmark?
    - Cache hit rate?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] JMH PR vs main comparison CI
- [ ] Regression threshold 10% gate
- [ ] PR comment with diff table
- [ ] k6 smoke test in CI
- [ ] Nightly load test (schedule)
- [ ] Banking realistic load model
- [ ] SLO-driven k6 thresholds
- [ ] Baseline storage (Prometheus/file)
- [ ] Pre-release full suite
- [ ] Dedicated CI runner setup
- [ ] Slack on regression

---

## Defter notları (10 madde)

1. "Perf testing pipeline placement (PR smoke + nightly load + pre-release suite): ____."
2. "JMH PR vs main regression % gate + PR comment automation: ____."
3. "k6 smoke test SLO-matching threshold + PR preview deploy: ____."
4. "Nightly load test schedule + staging refresh + baseline compare: ____."
5. "Banking realistic load (endpoint mix + think time + Pareto amount): ____."
6. "SLO-driven k6 thresholds (p99 < 1s, error < 0.5%) banking: ____."
7. "Baseline long-term store + Grafana trend + silent regression detect: ____."
8. "Dedicated CI runner (CPU pin + turbo off + performance governor): ____."
9. "Pre-release full suite (smoke + load + stress + spike + soak) gate: ____."
10. "Anti-pattern (shared runner + absolute threshold + hardcoded baseline): ____."
