# Faz 7 — PHASE TEST

```admonish question title="Bu test ne işe yarar?"
Faz 8'e geçmeden önce kendini sına. Bu bir sınav değil, **dürüstlük kontrolü**:
işaretleyemediğin her madde, hangi topic'e geri döneceğini gösterir.
Hepsine "evet" diyebiliyorsan hazırsın.
```

## Pratik test

- [ ] `core-banking` 4 microservice'e bölündü (Maven multi-module)
- [ ] Database per service prensibi uygulandı (4 schema)
- [ ] API Gateway + 5 route + JWT + Redis rate limiter + CB + IdempotencyKey
- [ ] K8s Service discovery (veya Eureka) çalışıyor
- [ ] Resilience4j: CB + Retry + Bulkhead + TimeLimiter + RateLimiter + Fallback per backend
- [ ] OpenTelemetry end-to-end trace Jaeger'da görünür (gateway → 3 service → DB)
- [ ] Cross-bank Saga + TCC orchestration + compensation
- [ ] 5 kasten kırma senaryosu reproduced + fixed
- [ ] Observability stack (Prometheus + Grafana + Jaeger + Loki)
- [ ] 25+ integration test passing

## Konsept testi

### DDD Strategic
- [ ] Bounded context kavramı + banking 4 servis örneği
- [ ] Ubiquitous language glossary + TR/İng karışıklığı çözümü
- [ ] Context map relationships (S-K, Customer-Supplier, ACL, vb.)
- [ ] Aggregate cross-bounded context tuzağı
- [ ] Strategic vs tactical DDD farkı

### Service Decomposition
- [ ] Microservice ayır vs birleşik tut karar matrisi
- [ ] Banking 4-servis rasyoneli
- [ ] Database per service 4 avantajı
- [ ] Distributed monolith 7 belirtisi
- [ ] Strangler fig pattern banking regulatory
- [ ] Decomposition strategy (capability/subdomain/aggregate/team)
- [ ] Conway's Law banking org

### API Gateway
- [ ] Gateway 4 sorumluluğu (entry, cross-cutting, hiding, evolution)
- [ ] Reactive (Netty) — servlet mix uyumsuzluğu
- [ ] Predicate vs Filter
- [ ] JWT gateway'de + backend'de duplicate olmama
- [ ] Redis rate limiter token bucket (replenishRate + burstCapacity)
- [ ] CB fallback URI + ProblemDetail
- [ ] IdempotencyKey banking POST/PUT zorunluluk
- [ ] Public vs internal vs admin endpoint ayrımı

### Service Discovery
- [ ] Client-side vs server-side vs DNS-based
- [ ] Eureka register + heartbeat lifecycle
- [ ] K8s Service DNS auto-discovery
- [ ] Spring Cloud LoadBalancer + @LoadBalanced
- [ ] Zone-aware load balancing
- [ ] Health check auto-eviction
- [ ] Service-to-service auth (user token vs client credentials)

### Resilience4j
- [ ] Cascading failure + retry storm patolojileri
- [ ] CB state machine (CLOSED → OPEN → HALF_OPEN)
- [ ] recordExceptions vs ignoreExceptions banking karar
- [ ] Retry exponential backoff
- [ ] Bulkhead semaphore vs ThreadPool
- [ ] Banking timeout hierarchy (DB → service → gateway)
- [ ] RateLimiter external API quota
- [ ] Composition order (Retry → CB → TL → Bulkhead)
- [ ] Fallback strategy (fail-fast vs cached vs queue vs degraded)

### Distributed Tracing
- [ ] Trace + Span + Context kavramları
- [ ] OpenTelemetry vs eski (OpenTracing, OpenCensus)
- [ ] W3C traceparent header format
- [ ] Auto-instrumentation vs manual span
- [ ] Sampling head-based vs tail-based
- [ ] PII span attribute tehlikesi
- [ ] Span name template (low-cardinality)
- [ ] Exemplars Grafana → Jaeger
- [ ] Kafka trace propagation
- [ ] Baggage cross-cutting context

### Distributed Locks & Transactions
- [ ] Distributed lock 5 kullanım senaryosu
- [ ] Redis SETNX + Lua atomic release
- [ ] Redlock controversy + Kleppmann critique
- [ ] pg_advisory_lock vs Redis vs ZooKeeper
- [ ] TCC Try-Confirm-Cancel banking örneği
- [ ] TCC Cancel implementation eksik anti-pattern
- [ ] Idempotency cross-service pattern
- [ ] Saga + TCC kombine cross-bank
- [ ] Lock granularity (resource vs service-wide)
- [ ] Network partition split-brain + DB-level guarantee

## Banking domain anlama

- [ ] Bir transfer request'inin 4 servis arası flow'unu beyaz tahtaya çizebilirim
- [ ] Cross-bank transfer Saga + TCC orchestration
- [ ] Banking için CB fallback strategies (critical vs non-critical)
- [ ] mTLS + JWT propagation service-to-service auth
- [ ] Banking PII propagation kontrolü (sadece internal ID)
- [ ] Outbox pattern (Phase 6) + microservices (Phase 7) entegrasyonu

## Soft skills

- [ ] 25 defter notu mini-project sonunda yazılmış
- [ ] Distributed system patolojilerini (cascading failure, retry storm) tanıyabilirim
- [ ] Trace okumayı öğrendim (waterfall, span attribute)
- [ ] CB state ve fallback davranışını canlı gözlemledim
- [ ] Saga compensation Jaeger'da takip ettim

## Kaç gün?

Tahmin: 25-30 gün (günde 2-3 saat). Phase 7 **en complex** faz — bir önceki + bir sonraki seviyede karmaşıklık.

---

Hepsine "evet" → **Faz 8'e geç → 08-security/**

---

## Bonus — mid+ seviyenin işareti

Phase 7'yi bitirdiğinde şu cümleleri rahatça söyleyebilirsen **mid+ / senior'a yakın** seviyedesin microservice architecture'da:

- "Monolitten 4 microservice'e Strangler Fig pattern ile kademeli migration tasarlayabilirim."
- "Spring Cloud Gateway production-grade setup (JWT + rate limit + CB + fallback) kurabilirim."
- "Resilience4j 6 pattern'ini composition order ile birlikte uygulayabilirim."
- "OpenTelemetry ile end-to-end distributed tracing setup'ı yapabilirim."
- "Cross-bank Saga + TCC orchestration design + implementation."
- "Distributed monolith anti-pattern'ini tespit edip refactor edebilirim."
- "Banking için CB fallback stratejisinin (fail-fast vs cached) kararını gerekçeli verebilirim."
- "K8s Service discovery + zone-aware load balancing kurabilirim."

TR bankalarında **mid+ seviye microservice architect** rolüne hazırsın.

---

## Banking için Phase 7'nin yeri

Phase 7 = **modern banking backend'in omurgası**. TR bankaları:

- **Open Banking** (BDDK 2020+) → API-first, gateway zorunlu
- **Microservice migration** → legacy core banking'den uzaklaşma
- **Real-time payment** (FAST) → low-latency microservice
- **PCI-DSS compliance** → service isolation
- **Multi-region resiliency** → distributed system patterns

Mid+ Java developer bu fazda öğrendiği **distributed system patterns**'i her gün kullanır. CV'de "Spring Cloud Gateway, Resilience4j, OpenTelemetry, Saga pattern, TCC, distributed tracing" bulundurmak TR bank mid+ pozisyonu için **rekabetçi profil**.

```admonish success title="Sonraki durak: Faz 8"
Phase 8 (Security) bunun üstüne kurulur — gateway'in JWT'sini Keycloak ile bağlamak, mTLS,
encryption, OWASP compliance. Phase 7 → Phase 8 doğal devam.
→ [Faz 8 — Security](../08-security/index.md)
```
