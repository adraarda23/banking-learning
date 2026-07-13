# Faz 7 — Microservices: Monolitten Servis-Bazlı Mimariye Geçiş

## Hedef

Faz 1-6 boyunca tek bir `core-banking` monolitinde geliştirdiğin domain'i (account, transfer, ledger, scheduling, batch, messaging) **iş yetkinliği** bazında 4 microservice'e böleceksin:

- **account-service** — hesap CRUD, balance, status, double-entry ledger sahibi
- **transfer-service** — para transferi orchestration, idempotency, saga koordinasyonu
- **fraud-service** — risk skorlama, transfer onay/red kararı, kural motoru
- **notification-service** — SMS/email/push gönderim, retry, template management

Bu fazda **microservice'in yaratacağı yeni problemleri** öğreneceksin:

1. Servisler arası iletişim nasıl olacak (HTTP, gRPC, event-driven)?
2. Bir servis düştüğünde diğeri nasıl davranacak (circuit breaker, fallback)?
3. Veri birden fazla DB'ye dağıldı, **transaction nasıl yönetilecek** (saga)?
4. Servisleri nasıl bulacaksın (service discovery)?
5. Tek bir kullanıcı isteği 4 servisi gezdiğinde **trace nasıl tutulacak**?
6. Bir transfer ortasında fraud-service down → ne olacak (Saga compensation)?
7. Tüm bunlar production'da gerçekten çalışacak mı (Toxiproxy ile felaket testi)?

Bu, junior'dan **mid-level**'a geçişin teknik kırılma noktasıdır. TR bankalarında microservice mimarisi son 7 yıldır defacto standart — Garanti, İşbank, Akbank, Yapı Kredi'de microservice uzmanlık beklenir.

## Süre

Tahmini: 25-32 gün (günde 2-3 saat). Faz 1-6'ya göre **daha yavaş** gidebilirsin — bu fazda kavram yoğunluğu çok yüksek.

| Topic | Tahmini süre |
|---|---|
| 01 DDD Strategic | 3-4 gün |
| 02 Service Decomposition | 2-3 gün |
| 03 API Gateway | 3 gün |
| 04 Service Discovery | 2-3 gün |
| 05 Resilience4j | 4 gün |
| 06 Distributed Tracing | 3 gün |
| 07 Distributed Locks & Transactions (Saga) | 5-6 gün |
| Mini-project: 4 servise bölme | 5-7 gün |

## Önbilgi (mutlaka tamam olmalı)

- Faz 1 (Foundation), Faz 2 (JPA), Faz 3 (Concurrency), Faz 6 (Messaging — Kafka) bitirilmiş
- Hexagonal architecture pratiğin var, domain saf Java yazabiliyorsun
- Idempotency-Key implementasyonunu en az bir kez yazdın
- Outbox pattern'i biliyorsun (Faz 6'da uygulandı)
- Docker compose ile multi-container stack ayağa kaldırabiliyorsun
- Kafka topic, partition, consumer group kavramları net

---

## Faz 7'de öğreneceğin somut beceriler

### Stratejik

- DDD bounded context tespiti — "bu kod hangi servisin?"
- Context map ilişki tipleri (shared kernel, ACL, customer-supplier, conformist)
- Service-per-aggregate ile service-per-bounded-context kararı
- "Neyi bölmemeli" — ortak referans data, sıkı bağlı domain'ler
- Conway's Law gerçek hayatta nasıl çalışır

### Teknik patterns

- API Gateway pattern (Spring Cloud Gateway, reactive Netty)
- Service discovery (Eureka client-side vs Kubernetes DNS server-side)
- Resilience4j: Circuit Breaker, Retry, Bulkhead, RateLimiter, TimeLimiter, Fallback
- Pattern composition order: Retry inside CircuitBreaker
- OpenTelemetry W3C traceparent propagation
- Saga orchestration (Spring State Machine) vs choreography (event-driven)
- Compensating action design
- Distributed lock (Redis Redlock kritiği, ZooKeeper, ShedLock)

### Banking-spesifik

- Cross-service transfer'in saga olarak modellenmesi (fraud check + balance check + ledger write + notification)
- Compensating action'lar: transfer reverse, hold release, notification cancellation
- Fraud-service down olduğunda transfer ne yapar (fail-closed vs fail-open kararı)
- Notification-service yavaş olduğunda **transfer geciktirilmemeli** (asenkron event)
- Cross-bank transfer (FAST sistemi) için tek bir saga; uzun süren işlem timeout yönetimi
- Audit trail her servisten merkezi log'a (correlation id ile)

### Operasyonel

- Network gecikmesi (Toxiproxy ile latency injection)
- Circuit breaker'ın trip etme senaryosunu görmek
- Eventual consistency'nin müşteri deneyimini nasıl etkilediği
- Distributed system'de "test edilebilirlik" zorlukları

---

## Faz 7'nin yapısı (klasör ve sıra)

```
07-microservices/
├── README.md                              ← buradasın
├── 01-ddd-strategic/
│   └── README.md                          ← bounded context, context map, aggregate
├── 02-service-decomposition/
│   └── README.md                          ← monolit → 4 servis stratejisi
├── 03-api-gateway/
│   └── README.md                          ← Spring Cloud Gateway
├── 04-service-discovery/
│   └── README.md                          ← Eureka vs K8s DNS
├── 05-resilience4j/
│   └── README.md                          ← 6 pattern, composition
├── 06-distributed-tracing/
│   └── README.md                          ← OpenTelemetry + Jaeger
├── 07-distributed-locks-transactions/
│   └── README.md                          ← Redlock kritiği + Saga deep
├── mini-project/
│   └── README.md                          ← 4 servise bölme, kasten kırma
└── PHASE_TEST.md                          ← faz sonu özdeğerlendirme
```

**Sıra önemli:** Önce stratejik (1-2), sonra connectivity (3-4), sonra resilience (5-6), sonra dağıtık tutarlılık (7), sonra proje. Atlama.

---

## Hangi monolit kısımları ayrılacak

Faz 1-6 boyunca tek repo'da bulunan modüller şu şekilde paketlenecek:

| Monolit modülü | Yeni servis | Sahip olduğu DB schema |
|---|---|---|
| `account/` (Account aggregate + ledger) | **account-service** | `accounts`, `journal_entries`, `journal_lines`, `idempotency_keys` |
| `transfer/` (Transfer orchestration, idempotency) | **transfer-service** | `transfers`, `transfer_saga_state`, `outbox` |
| `fraud/` (Risk skoru, kural motoru — Faz 5/6'da temel atıldı) | **fraud-service** | `fraud_rules`, `risk_scores`, `blacklist` |
| `notification/` (SMS, email, template) | **notification-service** | `notification_log`, `templates`, `delivery_attempts` |

**Ayrılmayan kısımlar (ortak):**
- `common/domain/Money`, `Currency`, `ErrorCodes` — **shared kernel** olarak küçük bir kütüphane (`banking-common`) içinde
- ProblemDetail handler — her serviste kopya (DRY ihlali kabul, bağımsızlık öncelikli)
- Customer/owner referansı — sadece **ID** olarak taşınır; customer servisi bu faz kapsamında yok (Faz 8'de eklenebilir)

---

## Mimari hedef diyagramı (yüksek seviye)

```
                         ┌───────────────────┐
                         │   Mobile / Web    │
                         │      Client       │
                         └─────────┬─────────┘
                                   │ HTTPS
                                   ↓
                         ┌───────────────────┐
                         │   API Gateway     │  ← rate limit, JWT validate, routing
                         │ (Spring Cloud GW) │
                         └─┬─────┬─────┬─────┘
                  /accounts │     │ /transfers
                            ↓     ↓
              ┌─────────────────┐   ┌─────────────────┐
              │ account-service │←──│ transfer-service│
              │  (own DB)       │   │  (own DB + saga)│
              └────────┬────────┘   └────┬─────┬──────┘
                       │                 │     │
                       │            sync REST  │ async (Kafka)
                       │                 ↓     ↓
                       │      ┌─────────────────┐    ┌──────────────────┐
                       │      │  fraud-service  │    │notification-svc  │
                       │      │   (own DB)      │    │   (own DB)       │
                       │      └─────────────────┘    └──────────────────┘
                       │
                       │ async events (Kafka)
                       ↓
                  ┌─────────────────────────────────────┐
                  │  Kafka (transfer.events, fraud.*)   │
                  └─────────────────────────────────────┘

           Cross-cutting:
           - Service registry (Eureka veya K8s DNS)
           - OpenTelemetry collector → Jaeger (tracing)
           - Loki / ELK (logs, traceId ile join)
           - Redis (distributed lock, idempotency cache)
```

---

## Bu fazın bitiminde elinizde ne olacak

- 4 ayrı Spring Boot servisi (her biri ayrı repo veya tek monorepo'da ayrı module)
- Her servisin kendi `application.yml`, kendi DB schema'sı (data sovereignty)
- API Gateway docker-compose ile ayağa kalkıyor
- Eureka (veya K8s lokal: `minikube`/`kind`) servis kaydı çalışıyor
- Resilience4j ile transfer→account, transfer→fraud çağrılarında circuit breaker
- Jaeger UI'da end-to-end transfer trace'i: client → gateway → transfer → fraud → account → notification
- Saga orchestrator ile cross-service transfer (fraud rejection → reverse compensation testi yazılı)
- Toxiproxy ile fraud-service'e 5sn latency ekleyince transfer-service'in **patlamadığını** test ettin
- Ledger double-entry invariant'ı hala korunuyor (account-service'in tek sorumluluğu)
- ADR'lar (en az 5 yeni mimari karar): bounded context kararı, service-per-context kararı, sync vs async iletişim kararı, saga orchestration vs choreography kararı, Eureka vs K8s kararı

---

## Bu fazda nelere DOKUNMAYACAĞIZ

Faz 7 zaten yoğun. Aşağıdakileri **bilinçli olarak ertelemek** lazım:

- **Service mesh (Istio, Linkerd):** Bilmen iyi ama TR bankalarının %80'i hâlâ K8s + Spring Cloud bağımsız çalışıyor. Faz 11 DevOps'ta dokunulacak.
- **gRPC:** REST yeterli. Sınıra dayanan performans gereksinimi olduğunda öğren.
- **GraphQL:** Banking back-office'inde nadir. Skip.
- **CQRS event sourcing tam:** Faz 6'da Outbox + Kafka var, event-sourcing tam değil. İleride özel bir faz konusu.
- **Multi-region active-active:** Bu mid-senior konusudur. Skip.
- **Chaos engineering tam suite (Chaos Monkey, Gremlin):** Toxiproxy ile minimum dokunuş yapacağız, profesyonel chaos engineering'e girmiyoruz.

---

## Bu fazda **dikkat** — junior'ın en sık yaptığı hatalar

1. **"Microservice = küçük servis" zannetmek.** Microservice **bağımsız deploy edilebilir** birimdir, küçüklük tesadüfidir. 50 satırlık bir servis de yanlış olabilir, 50K satırlık da olabilir.

2. **Distributed monolith yaratmak.** Her servis kendi DB'si olmalı. Servisler ortak DB'ye bağlanırsa, sen aslında monolit yazdın, sadece kodu parçaladın — en kötü dünya.

3. **Sync REST ile her şeyi çözmeye çalışmak.** Notification, audit, analytics gibi şeyler **asenkron event** ile gitmeli. Sync zincir = ilk servis düştüğünde tüm zincir düşer.

4. **Distributed transaction (2PC) aramak.** Saga öğren, 2PC bankada uygulanabilir değil (CAP teoremi, performans, kilitlenme).

5. **Saga'yı her yere uygulamak.** Bir aggregate içinde yapılabilen operasyon için saga = over-engineering. Sadece **çoklu aggregate** veya **çoklu servis** olduğunda.

6. **Circuit breaker'ı yanlış konfigüre etmek.** %50 failure rate threshold çoğu zaman çok düşük, %5 traffic'te bile trip eder. Production benchmark'ı şart.

7. **Trace ID'yi propagate etmeyi unutmak.** Bir servis trace context'ini başka servise iletmezse, distributed tracing kırılır. Spring Cloud Sleuth/Micrometer otomatik yapar — ama header'ları manuel bir HttpClient kullanıyorsan elle eklemen lazım.

8. **Service discovery'i compile-time hardcode'lamak.** `http://localhost:8081/accounts` production'da çalışmaz. Service name kullan, discovery client çözsün.

9. **Saga state'ini in-memory tutmak.** Servis restart edildi → saga state kayıp → tutarsız state. Saga state DB'ye yazılmalı, durability garantili.

10. **"Eventually consistent"ı müşteriye yansıtmak.** Müşteri transfer yaptı, balance hemen güncellenmiyor → şikayet. Front-end **optimistic UI** veya **polling** ile maskelemelidir. Eventual consistency back-end gerçeği, müşteri deneyimi gerçeği değil.

---

## Defter (Phase 7 baş notu)

Bu fazı bitirdiğinde aşağıdaki cümleleri **kendi kelimelerinle** doldurabilmen şart:

1. "Bir kodun ayrı bir microservice olması gerektiğini anlamam için baktığım kriterler şunlar: ____."
2. "Bounded context ile microservice arasındaki ilişki: ____. Her bounded context bir microservice midir? ____."
3. "Distributed transaction yerine Saga kullanma sebebi: ____."
4. "Bir transfer 4 servis gezerken bir tanesi düşerse, sistemin davranışını şöyle tasarlarım: ____."
5. "Circuit breaker'ın 3 state'i ve geçişlerini şöyle açıklarım: ____."

---

## Başla

→ [01-ddd-strategic/](./01-ddd-strategic/index.md)
