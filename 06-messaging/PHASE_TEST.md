# Faz 6 — PHASE TEST

Faz 7'ye geçmeden önce kendini sına.

## Pratik test

- [ ] Kafka KRaft stack docker-compose ile çalışıyor (Schema Registry + Kafka UI)
- [ ] Transfer endpoint → outbox → Kafka publish chain end-to-end
- [ ] 3 consumer (notification, audit, fraud) bağımsız tüketiyor
- [ ] Idempotent consumer (processed_events) → duplicate event tek kez işlendi
- [ ] DLT setup ile failed message recovery + ops alert
- [ ] Kafka Streams real-time fraud detection (1 dk window 5+ tx)
- [ ] Cross-bank Saga orchestration + compensation
- [ ] Multi-instance OutboxPublisher (3 instance, no duplicates)
- [ ] 5 kasten kırma reproduction + fix (dual-write, mid-crash, exactly-once, publisher race, compensation idempotency)
- [ ] 15+ integration test geçiyor

## Konsept testi

### Kafka Architecture
- [ ] Broker, topic, partition, replica, ISR kavramları
- [ ] Replication factor vs min.insync.replicas
- [ ] Partition key + ordering garantisi
- [ ] KRaft (no ZooKeeper) modern Kafka

### Producer
- [ ] acks=0 / 1 / all banking için karar
- [ ] enable.idempotence broker (PID, sequence) dedup
- [ ] max.in.flight + idempotence ordering
- [ ] Transactional producer multi-topic atomicity
- [ ] Async send + callback hot path için
- [ ] Outbox fallback eventual delivery
- [ ] Compression (lz4/zstd) banking JSON için
- [ ] Avro + Schema Registry banking için 3 avantaj
- [ ] Partition key entity ID (UUID DEĞİL)

### Consumer
- [ ] Manuel offset commit (MANUAL_IMMEDIATE)
- [ ] At-least-once + idempotent → effective exactly-once
- [ ] Cooperative rebalancing vs eager
- [ ] max.poll.interval.ms processing süresine göre
- [ ] CooperativeStickyAssignor banking standard
- [ ] DefaultErrorHandler retry + DLT pattern
- [ ] ErrorHandlingDeserializer poison message için
- [ ] processed_events tablo + cleanup retention
- [ ] Header propagation (X-Trace-Id) distributed tracing

### Spring Kafka
- [ ] ContainerFactory full config (manual ack, concurrency, error handler)
- [ ] Batch listener vs per-record karar
- [ ] @RetryableTopic non-blocking retry avantajı
- [ ] Transactional listener read-process-write atomicity
- [ ] Lifecycle pause/resume maintenance window
- [ ] Custom DLT routing (poison vs transient)

### Kafka Streams
- [ ] KStream / KTable / GlobalKTable kullanım farkı
- [ ] Stateless vs stateful operations
- [ ] Windowing (tumbling / hopping / session)
- [ ] State store local RocksDB + changelog backup
- [ ] Interactive query state store'a access
- [ ] exactly_once_v2 atomic read-process-write
- [ ] Processor API vs DSL trade-off

### Outbox & CDC
- [ ] Dual-write problem 3 patolojik senaryo
- [ ] Outbox pattern atomicity DB transaction'ından
- [ ] Polling vs CDC (Debezium) trade-off
- [ ] FOR UPDATE SKIP LOCKED multi-instance safe
- [ ] @TransactionalEventListener AFTER_COMMIT anti-pattern
- [ ] Outbox event ID = consumer idempotency key
- [ ] Outbox cleanup retention 7-30 gün
- [ ] Eventual consistency UX impact

### Saga Pattern
- [ ] 2PC neden banking için yasak (5 sebep)
- [ ] Saga vs 2PC karar matrisi
- [ ] Orchestration vs choreography
- [ ] Compensating action 5 kuralı
- [ ] Semantic compensation (SMS gibi geri alınamaz)
- [ ] Saga state DB-persistent
- [ ] Stuck saga timeout + auto-compensation
- [ ] TCC pattern (Try-Confirm-Cancel) booking için
- [ ] Spring State Machine ile orchestrator
- [ ] Async HTTP 202 + status polling

## Banking domain anlama

- [ ] TransferCompleted event publish chain (Transfer → Outbox → Kafka → 3 consumer)
- [ ] Idempotent consumer SMS duplicate sorunu
- [ ] Real-time fraud detection Kafka Streams ile mümkün
- [ ] Cross-bank Saga 4-step (Bank A → FX → Bank B → Audit) + compensation
- [ ] Banking eventual consistency (transfer immediate, SMS delayed) UX

## Soft skills

- [ ] Notification consumer log MDC traceId görünür
- [ ] Grafana dashboard producer/consumer/saga metrics
- [ ] DLT alert Slack/PagerDuty entegre
- [ ] Defterimde 20+ defter notu var
- [ ] GitHub'a 8+ commit, anlamlı mesajlar

## Kaç gün?

Tahmin: 20-25 gün (günde 2-3 saat). Çok yoğun faz, sabırlı ol.

---

Hepsine "evet" → **Faz 7'ye geç → 07-microservices/**

---

## Bonus — mid-junior'a geçtiğinin işareti

Phase 6'yı bitirdiğinde **mid-junior+** seviyedesin event-driven architecture tarafında:

- "Event-driven banking platform kurabilirim — outbox + idempotent consumer + DLT pattern."
- "Real-time fraud detection Kafka Streams ile tasarlayıp deploy edebilirim."
- "Cross-bank Saga orchestration design + implementation yapabilirim."
- "Dual-write problem'i identify edip outbox pattern ile fix edebilirim."
- "Exactly-once delivery'nin 3 koşulunu uygulayabilirim."
- "TR bank batch + event-driven hybrid mimari tasarlayabilirim."

TR bankalarında **modern stack'in en kritik fazı**. Eski IBM MQ legacy projelerinden modern Kafka projelerine geçiş ekibi sürekli senin gibi developer arıyor.

---

## Banking'de event-driven trendi

TR bankaları 2020+:
- Open Banking regülasyonu → API-first
- Microservice transition → event-driven backbone
- Real-time fraud → Kafka Streams
- Modern stack: Kafka, Spring Cloud, Resilience4j, OpenTelemetry

Phase 6 → modern banking core competency. CV'nde "Apache Kafka, Spring Kafka, Kafka Streams, Outbox pattern, Saga pattern" yazılı olunca mid-level mülakatları için **güçlü** profil.

Phase 7 (Microservices) bunun üstüne — monolit'ten 4 servise bölme, API Gateway, Resilience4j, distributed tracing. Phase 6 sağlam ise Phase 7 doğal evrim.
