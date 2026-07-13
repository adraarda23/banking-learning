# Faz 9 — Observability & Performance

## Hedef

Banking sistemini **production-grade gözlenebilir** hale getirmek. Logs, metrics, traces (Three Pillars) stack'inin tamamını kurmak, kullanmak, yorumlamak. Performance bottleneck'lerini sistematik biçimde **bulup düzeltmek**: profiling (JFR, async-profiler), heap analizi (MAT), GC log okuma, JMH ile mikro karşılaştırmalar, Gatling ile yük testi.

Bu faz, "kod yazan" junior'dan "üretimde sorunu çözen" mid-level developer'a geçişin **en kritik adımı**. Banka tarafında "production'da yavaşladı, sebep ne?" sorusuna **delil ile** cevap verebilmek bu fazda kazanılır.

## Süre

Tahmin: 14-20 gün (günde 3-4 saat). 8 topic + mini-project + phase test.

## Önbilgi

- Faz 1-8 tamamlandı (özellikle Faz 2 JPA, Faz 3 concurrency, Faz 7 microservices, Faz 8 security)
- `core-banking` projesi çalışır durumda
- Faz 1 Topic 1.7'den `traceId` filter ve MDC kavramı zaten elinde
- Faz 3'te JMH'in basit kullanımını gördün — burada derinleşeceğiz
- Faz 7'de microservices arası HTTP/Kafka iletişim var — distributed tracing burada hayat kazanır

## Why this matters — banking için observability hikâyesi

Bir TR bankasının core-banking servisinde sabah 09:00'da müşteri şikayetleri patlıyor: "EFT yavaş geliyor". Bu sabah özellikle. Ne yapacaksın?

**Observability olmadan:** Tahminle yola çıkarsın. "DB yavaşladı mı?" — bilmiyorsun. "Kafka mı tıkandı?" — bilmiyorsun. "Yeni deploy mu bozdu?" — fark etmedin bile. Saatler boyu log dosyalarında stacktrace ararsın. Müşteri kaybedersin. Manager kızar.

**Observability ile:**
1. **Grafana dashboard** açarsın: `transfer-service` p99 latency'si 200ms'den 3.5s'ye fırlamış, saat 08:55'te. Yeni deploy 08:50'de yapıldı.
2. **Prometheus query**: `rate(http_server_requests_seconds_count{uri="/v1/transfers",status="500"}[5m])` — error oranı 0.01'den 0.4'e çıkmış.
3. **Jaeger UI**: Bir slow trace'i açarsın, `transfer-service → fraud-service` span'ı 3s. Fraud servisi sebep.
4. **Fraud servis Jaeger detayı**: bir span'ın içinde `redis.get` 2.8s sürmüş. Redis cache'i çöktü.
5. **Loki / log aggregator**: `traceId=abc` arar, exception görürsün: `RedisConnectionTimeoutException`.
6. **Profil**: ek olarak async-profiler ile CPU flame graph'ında "Jackson deserialization 60% CPU" görürsün — başka bir sorun.

15 dakikada **kanıt ile** çözüm.

Banka observability'ye **çok ciddi** yatırım yapar. Junior'dan mid'e geçiş, "bu stack'i çalıştırabilirim ve sorunu bulurum" demektir.

---

## Three Pillars of Observability

Modern observability'nin **üç temel direği** vardır. Bu fazın iskeleti budur.

### 1. Logs (Loglar) — "ne oldu, ne zaman, neden"

**Tanım:** Sistemde olan olayların kayıtları. Yapılandırılmış (structured) veya yapılandırılmamış (plaintext) olabilir.

**Soru tipi:** "Bu hata neden oldu?" "Bu request'in detayı ne?" "Audit log: kullanıcı X şu saatte ne yaptı?"

**Banking örneği:**
```json
{
  "@timestamp": "2025-05-12T10:30:00.123Z",
  "level": "WARN",
  "logger": "com.mavibank.banking.transfer.application.service.ExecuteTransferService",
  "message": "Insufficient funds for transfer",
  "traceId": "abc12345",
  "spanId": "def67890",
  "fromAccountId": "550e8400-e29b-...",
  "amount": "10000.00",
  "currency": "TRY",
  "availableBalance": "5000.00",
  "userId": "user-123",
  "ip": "10.0.0.5"
}
```

**Strengths:** Detaylı bağlam, debug için ideal, audit gerekliliği (BDDK).
**Weaknesses:** Hacim büyük, expensive storage, aggregate sorgu için kötü.

### 2. Metrics (Metrikler) — "ne kadar, ne sıklıkla, ne kadar süreyle"

**Tanım:** Zaman serisine yazılan sayısal değerler. Yüksek frekansla toplanır, agrega edilir.

**Soru tipi:** "Şu anda saniyede kaç transfer geliyor?" "p99 latency'si nedir?" "Error oranı ne?"

**Banking örneği:**
```promql
# Saniyede kaç transfer (last 5 min average)
rate(banking_transfers_total[5m])

# p99 transfer latency (last 5 min)
histogram_quantile(0.99, rate(banking_transfer_duration_seconds_bucket[5m]))

# Error oranı by status code
sum by (status) (rate(http_server_requests_seconds_count[5m]))
```

**Strengths:** Çok az yer kaplar, agrega sorgu hızlı, dashboard ideal, alerting kolay.
**Weaknesses:** Bağlam yok ("hangi user yavaş aldı?" cevaplanamaz).

### 3. Traces (İz takipleri) — "request bütün sistemde nereden geçti"

**Tanım:** Bir request'in **birden fazla servisten geçerken** yaptığı tüm işlemlerin (span'ların) ağacı.

**Soru tipi:** "Bu yavaş request neden yavaştı, hangi servis sebep?"

**Banking örneği — bir EFT request'inin trace'i:**
```
TRACE traceId=abc12345 (total: 3.5s)
├── api-gateway:POST /v1/transfers (3.5s)
│   ├── auth-service:validateToken (50ms)
│   └── transfer-service:executeTransfer (3.4s)
│       ├── postgres:SELECT account (10ms)
│       ├── postgres:SELECT account (8ms)
│       ├── fraud-service:checkFraud (3.2s) ← BOTTLENECK
│       │   ├── redis:GET fraud-rules (2.9s) ← REAL CULPRIT
│       │   └── ml-service:scoreTransaction (200ms)
│       ├── postgres:UPDATE accounts (20ms)
│       └── kafka:publish TransferCompleted (50ms)
```

**Strengths:** End-to-end görünüm, microservice çağrı zincirini görür, bottleneck bulur.
**Weaknesses:** Yüksek hacim (sampling gerekir), setup karmaşık.

### Üçü beraber çalışır

- **Metric**: "p99 latency yükseldi" — alert tetikler.
- **Trace**: hangi servis yavaş, hangi span sorumlu.
- **Log**: o span'da ne hata var, detay ne.

**Modern observability'nin altın kuralı: Metric → Trace → Log akışı.** Alert metric'ten gelir, sorun trace'te lokalize edilir, kök neden log'da görülür.

---

## Bu fazın map'i

| Topic | Pillar / Konu | Tool stack |
|---|---|---|
| 9.1 Structured Logging | Logs | Logback + SLF4J + logstash-encoder + ELK/Loki |
| 9.2 Metrics | Metrics | Micrometer + Prometheus + Grafana |
| 9.3 Distributed Tracing | Traces | OpenTelemetry + Jaeger + Spring Boot 3 |
| 9.4 Profiling | Performance | JFR + JMC + async-profiler + flame graph |
| 9.5 Heap & Thread Dumps | Memory/Threading | MAT + jstack + GC log analysis |
| 9.6 JMH Deep | Microbenchmark | JMH + profilers (gc, async-profiler) |
| 9.7 Load Testing | Throughput | Gatling + k6 |
| Mini-project | Full stack | Tüm yukarıdaki + docker-compose |

---

## Tools — kurulum hazırlığı (genel rehber)

**Local stack için Docker imajları (mini-project'te detaylı):**
- `prom/prometheus` — Prometheus server
- `grafana/grafana` — Grafana UI
- `jaegertracing/all-in-one` — Jaeger (collector + query + UI)
- `grafana/loki` — Loki (log aggregation, Grafana ekosistemi)
- `grafana/promtail` — Loki log shipper
- `otel/opentelemetry-collector-contrib` — OTel collector (gateway)

**Local JVM tooling (java SDK ile gelir veya ayrı kurulum):**
- `jcmd`, `jstack`, `jmap`, `jstat` — JDK built-in
- `JFR` (Java Flight Recorder) — JDK 17+ built-in, license-free
- `JMC` (Java Mission Control) — ayrı indirilir, Eclipse Adoptium veya Oracle
- `async-profiler` — GitHub release, brew install async-profiler
- `MAT` (Eclipse Memory Analyzer) — eclipse.org/mat
- `GCViewer` veya `gceasy.io` — GC log analyzer

**Load testing:**
- `gatling` — Gatling open-source, Scala-based DSL
- `k6` — Grafana Labs, JavaScript DSL

Bu fazda her topic'in ilk bölümünde **somut kurulum komutları** verilecek.

---

## Bu fazın çıktıları (mini-project sonu)

Bitirdiğinde:
- `core-banking` Prometheus metrics expose ediyor, custom metrics tanımlı (transfer latency, fraud score, account creation rate)
- Grafana'da RED + USE method dashboard'u
- Jaeger UI'da bir transfer'in end-to-end trace'i görünür
- Structured JSON logs Loki'ye gidiyor, Grafana Explore ile sorgulanabilir
- Gatling load test 1000 req/sec hedefiyle çalışıyor, breaking point bulundu
- async-profiler ile CPU flame graph oluşturuldu, bir hotspot tespit edilip iyileştirildi
- G1 vs ZGC karşılaştırma yapıldı
- Heap dump alındı, MAT ile analyze edildi (simulated leak)

Bu çıktılar **CV-ready**. Banka interviewer'ı "Grafana dashboard'unu nasıl kurdun?" diye sorduğunda **gerçek deneyimle** cevaplayabilirsin.

---

## Faz boyunca uyulacak prensipler

1. **Her topic'i `core-banking` üzerinde uygula.** Teorik kalmayın. Mini-project birikimle yapılır.
2. **Defter notlarını yaz.** Her topic sonunda not bölümünü kendi cümlelerinle doldur.
3. **Claude-verify prompt'unu kullan.** Sadece doğrulama, kod yazdırma.
4. **Production-grade hedefi koru.** Toy değil, banka kalitesi.
5. **Sensitive data redaction.** Hiçbir zaman TC No, kart no, password log'a / metric tag'e / trace attribute'una koyma.

---

## Sıralama

Fazın bağımlılık zinciri:
```
9.1 Structured Logging   ← MDC, traceId (Faz 1.7'den geliyor)
        ↓
9.2 Metrics              ← Micrometer + Prometheus
        ↓
9.3 Distributed Tracing  ← OpenTelemetry + Jaeger; metric exemplar ile linkleniyor
        ↓
9.4 Profiling            ← JFR + async-profiler
        ↓
9.5 Heap & Thread Dumps  ← MAT + GC log
        ↓
9.6 JMH Deep             ← Faz 3 JMH'in devamı
        ↓
9.7 Load Testing         ← Gatling, full stack üzerinde
        ↓
Mini-project (full stack)
        ↓
PHASE_TEST
        ↓
Faz 10 (Domain — Türk bankacılığı: IBAN, EFT, FAST, BDDK)
```

→ Topic 9.1'e başla → [01-structured-logging/](./01-structured-logging/README.md)
