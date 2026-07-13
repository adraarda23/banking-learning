# Faz 9 — PHASE TEST

Faz 10'a geçmeden önce kendini sına.

## Pratik test

- [ ] Full observability stack docker-compose (Loki + Prometheus + Alertmanager + Jaeger + OTel Collector + Pyroscope + Grafana)
- [ ] 4 service: JSON structured log + MDC + sensitive data masking
- [ ] 4 service: 10+ custom banking metric + dashboards
- [ ] 4 service: distributed tracing + manual span + baggage propagation
- [ ] 4 service: Pyroscope continuous profile agent attached
- [ ] OTel Collector tail-based sampling (errors + slow + high-value)
- [ ] Grafana banking dashboard 4 panel set (RED + business + distributed + SLO)
- [ ] Alertmanager rules 8+ (HighErrorRate, P99Slow, DbPool, Saga, OutboxStale, KafkaLag, ErrorBudget, BruteForce)
- [ ] JMH benchmark suite + CI regression detection
- [ ] k6 load test scripts (smoke + load + stress + spike + soak)
- [ ] 10 incident runbook
- [ ] 5 kasten kırma senaryo reproduce + diagnose + fix + verify
- [ ] Banking retention: audit 5-10 yıl, log tiered storage

## Konsept testi

### Structured Logging
- [ ] JSON structured vs plain text banking trade-off
- [ ] SLF4J parameterized lazy evaluation
- [ ] MDC per-thread context + RequestContextFilter pattern
- [ ] Banking MDC keys (transactionId, accountId, channel)
- [ ] Sensitive data masking regex (TC, PAN, Bearer)
- [ ] Log levels banking strategy (INFO prod, dynamic DEBUG)
- [ ] StructuredArguments kv() explicit fields
- [ ] AsyncAppender queue + neverBlock banking choice
- [ ] Log aggregation (ELK/Loki/Datadog) Fluent Bit K8s
- [ ] Banking retention BDDK 5-10 yıl tiered storage

### Metrics
- [ ] 3 pillar log/metric/trace farkı + retention
- [ ] Prometheus 4 type (Counter, Gauge, Histogram, Summary)
- [ ] Histogram vs Summary banking pick
- [ ] Cardinality budget (userId/accountId TAG OLARAK YOK)
- [ ] Timer percentile_histogram + SLO bucket banking
- [ ] RED method endpoint-level
- [ ] USE method resource-level (DB pool, thread pool)
- [ ] Banking domain (transfers, logins, sagas, outbox, kafka_lag)
- [ ] PromQL p99 latency + error rate + burn rate
- [ ] SLO/SLI/SLA + error budget burn rate
- [ ] Alert critical/high/warning + Alertmanager routing

### Distributed Tracing
- [ ] OpenTelemetry + Micrometer Tracing bridge banking abstraction
- [ ] Auto-instrumentation (HTTP, JDBC, Kafka) vs manual span
- [ ] Span attribute semantic convention banking (tenant, currency, bucket)
- [ ] PII span attribute leak risk (TC, PAN, email)
- [ ] Exemplars Prometheus + Jaeger drill-down workflow
- [ ] Baggage cross-service business context
- [ ] Head vs tail-based sampling trade-off banking
- [ ] Tail sampling rules (errors + slow + high-value)
- [ ] Kafka trace propagation producer/consumer header
- [ ] Trace root cause workflow (alert → exemplar → trace → fix)

### Profiling
- [ ] Sampling vs instrumentation profiler banking overhead
- [ ] JFR continuous + on-demand dump workflow
- [ ] async-profiler 4 mode (cpu, alloc, lock, wall) banking
- [ ] Flame graph reading (genişlik = zaman)
- [ ] Wall-clock profile banking I/O sebebi
- [ ] Continuous profiling (Pyroscope/Parca) diff workflow
- [ ] BigDecimal allocation hotspot banking fix patterns
- [ ] Lock contention reduce — granularity + optimistic
- [ ] JVM tuning from profile (G1GC, ZGC banking)
- [ ] Incident workflow (alert → trace → profile → fix → verify)

### Heap & Thread Dumps
- [ ] JVM memory layout (heap, metaspace, native, off-heap)
- [ ] MAT Leak Suspects → Dominator Tree → Histogram workflow
- [ ] Common Java leak (unbounded cache, ThreadLocal, inner class)
- [ ] Thread state interpretation banking (RUNNABLE, BLOCKED, WAITING)
- [ ] Deadlock detect (jstack + ThreadMXBean) + lock ordering fix
- [ ] FastThread.io thread pool exhaustion + repeating pattern
- [ ] GC log + GCEasy throughput + pause analysis banking targets
- [ ] G1GC vs ZGC banking trade-off
- [ ] Native memory tracking + direct buffer leak
- [ ] Idempotency cache OOM case + Redis fix

### JMH Deep
- [ ] Naive benchmark bozuk (DCE, JIT, GC, frequency)
- [ ] Mode selection (AverageTime, Throughput, SampleTime)
- [ ] State scope (Thread vs Benchmark vs Group)
- [ ] Blackhole DCE prevention
- [ ] Parameter sweep @Param matrix
- [ ] async-profiler integration -prof
- [ ] Dedicated CI runner + CPU affinity stable env
- [ ] Score ± Error statistical significance
- [ ] PR vs baseline regression CI gate
- [ ] Production profile → JMH → fix → re-benchmark verify

### Load Testing
- [ ] Load test types (smoke/load/stress/spike/soak/volume)
- [ ] Open vs closed workload model banking
- [ ] Realistic endpoint mix + think time + amount distribution
- [ ] Gatling vs k6 vs JMeter selection
- [ ] SLO threshold CI gate
- [ ] Load test + observability tie (Grafana + Jaeger + JFR)
- [ ] Session lifecycle (login → browse → action → logout)
- [ ] Idempotency under load
- [ ] Stress breaking point + spike recovery
- [ ] CI integration nightly + baseline + Slack notify

## Banking compliance

- [ ] BDDK denetimi: audit log 5-10 yıl retention
- [ ] KVKK: PII log masking + tracing PII removal
- [ ] PCI-DSS: card data hiçbir log/metric/trace'de
- [ ] MASAK: transaction audit + monitoring metric'leri
- [ ] Open Banking: API performance SLO

## Soft skills

- [ ] 15 mini-project defter notu yazılmış
- [ ] Incident runbook 10 banking specific yazıldı
- [ ] 5 kasten kırma senaryo workflow recorded
- [ ] Slow request investigation pratik yapıldı
- [ ] OOM, deadlock, cascading failure scenario tanıdı
- [ ] Capacity planning load test'ten data ile yapabilirim

## Kaç gün?

Tahmin: 30-35 gün (günde 3 saat). Phase 9 = **production operability**. Bu phase'i bitiren bir engineer 3 AM oncall'da işe yarar.

---

Hepsine "evet" → **Faz 10'a geç → 10-domain-mastery/**

---

## Bonus — senior banking SRE / Backend engineer işareti

Phase 9 bittikten sonra şu cümleleri söyleyebiliyorsan:

- "Banking microservice observability stack'i sıfırdan (logs + metrics + traces + profile) kurabilirim."
- "Production incident 3 AM aldığımda: alert → dashboard → exemplar → trace → profile → root cause → fix flow'unu canlı uygulayabilirim."
- "Banking için SLO tanımlayıp error budget burn rate alert kurabilirim."
- "OOM, deadlock, lock contention, memory leak senaryolarını reproduce + diagnose + fix yapabilirim."
- "JMH ile performance regression CI guard kurabilirim."
- "k6 load test ile capacity planning + breaking point + spike resilience verify edebilirim."
- "BDDK + KVKK retention + masking compliance ile observability'i design edebilirim."
- "Tail-based sampling + cost vs signal trade-off karar verebilirim."
- "Continuous profiling diff'i ile performance regression spotlama yapabilirim."
- "10 incident runbook + team handover yazabilirim."

TR bankalarında **senior backend / SRE** rolüne hazır profilsin. Banking için ek **MASAK alert rules + KVKK retention** bilmek bonus.

---

## Banking için Phase 9'un yeri

Phase 9 = **banking operational excellence**. TR bankaları için:

- **24/7 SLA:** Banking app 99.95% uptime. Phase 9 olmadan tutturulamaz.
- **BDDK denetim:** Audit log + retention + incident report.
- **KVKK denetim:** PII masking observability stack'inde olmalı.
- **PCI-DSS:** Card data observability'i sıkı isolation.
- **Open Banking SLO:** 3rd party API performance contract.
- **Incident response:** RTO < 1 saat, RPO < 5 dk banking target.

Mid+ engineer artık **production operations** sorumluluğu da alabilir. SRE roller TR banking sektöründe rekabetçi maaş, çok arananan profil.

Phase 9 → Phase 10 (Domain Mastery): observability + business domain kombinasyonu = senior banking engineer.
