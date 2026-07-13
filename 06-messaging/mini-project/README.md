# Phase 6 Mini-Project — Event-Driven Banking Platform

## Hedef

`core-banking`'i **event-driven architecture'a** evrilmeye. Kafka cluster lokal kur. Transfer event publish + 3 paralel consumer (notification, audit, fraud). Outbox pattern dual-write çözümü. Kafka Streams ile real-time fraud detection. Cross-bank Saga orchestrator. Tüm bunlar **production-grade**:
- acks=all + idempotence + transactional
- exactly_once_v2
- DLT + retry pattern
- Idempotent consumer + processed_events
- ShedLock multi-instance safe outbox publisher
- Async API + status polling

Sonunda elinde: TR bank backend ekibinin Kafka tarafında **kullanabileceği** referans implementation.

## Süre

8-12 gün (günde 2-3 saat).

## Önbilgi

Phase 6'nın 7 topic'i bitti. Transfer/account domain Phase 1-5'ten hazır. Outbox pattern (Topic 6.6), Saga (Topic 6.7) detaylı bildiğin için.

---

## Görev 1 — Kafka stack docker-compose (yarım gün)

### 1.1 KRaft Kafka (no ZooKeeper)

```yaml
# docker-compose.kafka.yml
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: banking-kafka
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_LISTENERS: 'PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:9092'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:9093'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT'
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      CLUSTER_ID: 'banking-cluster-001'
    ports:
      - "9092:9092"
    healthcheck:
      test: kafka-broker-api-versions --bootstrap-server localhost:9092
      interval: 10s
      timeout: 5s
      retries: 5
  
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    container_name: banking-schema-registry
    depends_on:
      kafka:
        condition: service_healthy
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    ports:
      - "8081:8081"
  
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: banking-kafka-ui
    depends_on:
      - kafka
    environment:
      KAFKA_CLUSTERS_0_NAME: banking
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    ports:
      - "8090:8080"
```

### 1.2 Topics yarat

```bash
docker compose -f docker-compose.kafka.yml up -d

# Wait healthy
sleep 30

# Banking topics
for topic in banking.transfers banking.audit banking.notifications banking.fraud.alerts; do
    docker exec banking-kafka kafka-topics --bootstrap-server localhost:9092 \
        --create --topic $topic --partitions 10 --replication-factor 1 \
        --config min.insync.replicas=1
done

# DLT topics
for topic in banking.transfers.DLT banking.audit.DLT banking.notifications.DLT; do
    docker exec banking-kafka kafka-topics --bootstrap-server localhost:9092 \
        --create --topic $topic --partitions 3 --replication-factor 1
done

# Verify
docker exec banking-kafka kafka-topics --bootstrap-server localhost:9092 --list
```

Kafka UI: `http://localhost:8090` — topic'leri gör.

### Deliverables

- [ ] Kafka KRaft mode docker-compose
- [ ] Schema Registry container
- [ ] Kafka UI (provectuslabs)
- [ ] 7 topic yaratıldı
- [ ] Healthcheck passing

---

## Görev 2 — TransferEventPublisher + Outbox (1.5 gün)

### 2.1 Outbox migration

`V_outbox_events.sql` (Topic 6.6):

```sql
CREATE TABLE outbox_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(100) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    published_at    TIMESTAMP WITH TIME ZONE,
    failure_count   INT DEFAULT 0,
    last_error      TEXT,
    
    CONSTRAINT chk_outbox_status CHECK (status IN ('PENDING', 'PUBLISHED', 'FAILED'))
);

CREATE INDEX idx_outbox_pending ON outbox_events(status, created_at) WHERE status = 'PENDING';
```

### 2.2 TransferService refactor

```java
@Service
@Slf4j
public class TransferService {
    
    private final AccountRepository accountRepo;
    private final TransferRepository transferRepo;
    private final OutboxEventService outboxEventService;
    
    @Transactional
    public Transfer execute(TransferRequest req, UUID userId) {
        // Idempotency check
        if (transferRepo.existsByIdempotencyKey(req.idempotencyKey())) {
            return transferRepo.findByIdempotencyKey(req.idempotencyKey()).orElseThrow();
        }
        
        // Domain
        Account from = accountRepo.findByIdAndLock(req.fromAccountId()).orElseThrow();
        Account to = accountRepo.findByIdAndLock(req.toAccountId()).orElseThrow();
        from.withdraw(req.amount(), req.currency());
        to.deposit(req.amount(), req.currency());
        accountRepo.save(from);
        accountRepo.save(to);
        
        Transfer transfer = transferRepo.save(new Transfer(req, userId));
        
        // Outbox events — same transaction
        outboxEventService.publish("Transfer", transfer.getId().toString(),
            "TransferCompleted", new TransferCompletedEvent(transfer));
        
        outboxEventService.publish("Audit", transfer.getId().toString(),
            "AuditLog", new AuditLogEvent("TRANSFER_EXECUTED", userId, transfer.getId()));
        
        outboxEventService.publish("Notification", transfer.getToAccountId().toString(),
            "NotificationRequest", new NotificationRequest(transfer));
        
        return transfer;
    }
}
```

### 2.3 OutboxPublisher

Topic 6.6'da detaylı kod. Polling + ShedLock + FOR UPDATE SKIP LOCKED.

### Deliverables

- [ ] Outbox migration uygulandı
- [ ] TransferService 3 outbox event same TX
- [ ] OutboxPublisher polling + ShedLock
- [ ] Rollback test: transfer fail → outbox 0 event
- [ ] Happy path test: transfer success → outbox 3 event PENDING → publisher publish PUBLISHED

---

## Görev 3 — 3 Consumer Service (2 gün)

### 3.1 NotificationConsumer

```java
@Component
@Slf4j
public class NotificationConsumer {
    
    private final NotificationService notificationService;
    private final ProcessedEventRepository processedRepo;
    
    @KafkaListener(
        topics = "banking.transfer.transfer-completed",
        groupId = "notification-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    @Transactional
    public void consume(
        @Payload TransferCompletedEvent event,
        @Header(value = "X-Trace-Id", required = false) String traceId,
        Acknowledgment ack
    ) {
        MDC.put("traceId", traceId);
        try {
            UUID eventId = event.getId();
            
            if (processedRepo.existsByEventIdAndConsumerGroup(eventId, "notification-service")) {
                ack.acknowledge();
                return;
            }
            
            notificationService.sendTransferNotification(event);
            processedRepo.save(new ProcessedEvent(eventId, "notification-service"));
            
            ack.acknowledge();
        } finally {
            MDC.clear();
        }
    }
}
```

### 3.2 AuditConsumer

```java
@Component
public class AuditConsumer {
    
    @KafkaListener(topics = "banking.audit.audit-log", groupId = "audit-service")
    @Transactional
    public void consume(@Payload AuditLogEvent event, Acknowledgment ack) {
        if (auditRepo.existsByEventId(event.getId())) {
            ack.acknowledge();
            return;
        }
        
        auditRepo.save(AuditRecord.from(event));
        ack.acknowledge();
    }
}
```

### 3.3 FraudConsumer (Kafka Streams)

Topic 6.5'teki fraud pipeline. Real-time alert generation.

### Deliverables

- [ ] 3 consumer service
- [ ] Idempotent consumer pattern (processed_events)
- [ ] Header propagation (X-Trace-Id → MDC)
- [ ] DefaultErrorHandler + DLT
- [ ] Fraud detection: 5+ tx/dk → CRITICAL alert

---

## Görev 4 — DLT setup (yarım gün)

DefaultErrorHandler + DLT consumer + alert (Topic 6.4).

```java
@Component
public class DltMonitor {
    
    @KafkaListener(topics = {"banking.transfers.DLT", "banking.audit.DLT", "banking.notifications.DLT"})
    public void onDlt(ConsumerRecord<String, String> record) {
        log.error("DLT message: topic={}, value={}", record.topic(), record.value());
        
        dlqRepo.save(new DeadLetterRecord(record.topic(), record.value()));
        notifier.alertOps("DLT: " + record.topic());
    }
}
```

### Deliverables

- [ ] 3 DLT topic
- [ ] DltMonitor consumer
- [ ] Failure simulation: invalid event → 3 retry → DLT → alert

---

## Görev 5 — Cross-bank Saga (2 gün)

Topic 6.7'de detaylı kod. Full implementation:
- SagaStateRepository
- CrossBankTransferSaga orchestrator
- 6 Kafka listener
- Compensation flow
- Stuck saga recovery scheduler
- Async API (202 Accepted + GET status)

```java
@RestController
@RequestMapping("/v1/cross-bank-transfers")
public class CrossBankTransferController {
    
    @Autowired CrossBankTransferSaga saga;
    @Autowired SagaStateRepository sagaRepo;
    
    @PostMapping
    public ResponseEntity<SagaCreatedResponse> initiate(
            @Valid @RequestBody CrossBankTransferRequest req) {
        UUID sagaId = saga.initiate(req);
        return ResponseEntity.accepted()
            .body(new SagaCreatedResponse(sagaId, "/v1/sagas/" + sagaId));
    }
    
    @GetMapping("/{sagaId}/status")
    public SagaStatusResponse getStatus(@PathVariable UUID sagaId) {
        SagaState state = sagaRepo.findById(sagaId).orElseThrow();
        return new SagaStatusResponse(state.getCurrentState(), state.getUpdatedAt());
    }
}
```

### Deliverables

- [ ] saga_states migration
- [ ] CrossBankTransferSaga orchestrator
- [ ] 6 step + compensation
- [ ] Mock external bank servisleri (Bank A, Bank B, FX, Audit, Notification)
- [ ] Stuck saga scheduler (5 min timeout)
- [ ] Async API + status polling
- [ ] Happy path + compensation test

---

## Görev 6 — Schema Registry + Avro (1 gün, opsiyonel ileri)

Avro schema'lar tanımla, `xjc` ile Java class generate, JSON → Avro migration.

```bash
mvn avro:schema
```

### Deliverables (opsiyonel)

- [ ] 4 Avro schema (TransferCompleted, AuditLog, NotificationRequest, FraudAlert)
- [ ] Schema Registry'e register
- [ ] Producer + Consumer Avro serializer
- [ ] Schema evolution test (backward compatible field add)

---

## Görev 7 — Kasten kırma görevleri (1 gün)

### 7.1 Dual-write reproduction (outbox YOK)

İlk önce **outbox olmadan** dual-write yap:

```java
@Transactional
public void transferWithoutOutbox(req) {
    transferRepo.save(...);
    kafkaTemplate.send("banking.transfers", event);   // dual-write
}
```

Kafka stop → transfer DB'de var, event Kafka'da YOK. **Veri kaybı reproduce**.

Sonra outbox ekle → fix.

### 7.2 Consumer crash mid-processing

Consumer event aldı, process'i %50 yaptı, crash. Offset commit YOK → restart'ta yeniden process.

Idempotency olmadan → side effect 2 kez (SMS 2 kez gönder).

processed_events ekle → fix.

### 7.3 Exactly-once test

```java
// Without exactly_once_v2
producer.send(event);
// Network glitch, retry
// Consumer 2 mesaj alır
```

`processing.guarantee=exactly_once_v2` ekle → consumer tek mesaj.

### 7.4 Outbox publisher race

3 publisher instance, FOR UPDATE SKIP LOCKED **olmadan**:

```java
List<OutboxEvent> pending = outboxRepo.findByStatus(PENDING);   // ❌ no lock
```

3 publisher aynı event'i alır → Kafka'ya 3 kez gönderir.

SKIP LOCKED ile fix → her event bir kez.

### 7.5 Saga compensation idempotency

```java
public void reverseDebit(UUID txId) {
    // ❌ no check
    accountService.credit(txId, amount);
}
```

Network retry → compensation 2 kez → double credit.

`reversedTxRepo` check ekle → idempotent.

### Deliverables

- [ ] 5 kasten kırma reproduction + fix
- [ ] Her birinin **defterimde** before/after notu
- [ ] Banking için "neden bu pattern şart" anlatımı

---

## Görev 8 — Integration test suite (1 gün)

`@SpringBootTest + @Testcontainers + KafkaContainer + PostgreSQLContainer`:

- TransferService outbox test (Görev 2)
- OutboxPublisher publish test (Görev 2)
- NotificationConsumer idempotency test (Görev 3)
- AuditConsumer test
- FraudConsumer Kafka Streams TopologyTestDriver test
- DLT routing test
- CrossBankSaga happy path test
- CrossBankSaga compensation test
- End-to-end test: HTTP transfer → outbox → 3 consumer → side effects

Toplam 15+ integration test.

---

## Acceptance criteria

- [ ] Kafka KRaft stack docker-compose ile çalışıyor
- [ ] Schema Registry + Kafka UI accessible
- [ ] Transfer endpoint → outbox 3 event → Kafka publish chain
- [ ] 3 consumer service (notification, audit, fraud) mesajları işliyor
- [ ] Idempotent consumer pattern (processed_events) — duplicate skip
- [ ] DLT setup ile error recovery + ops alert
- [ ] Kafka Streams ile real-time fraud detection (5+ tx/dk)
- [ ] Cross-bank Saga orchestration (4 step + compensation)
- [ ] Outbox pattern dual-write çözüm
- [ ] Multi-instance OutboxPublisher (SKIP LOCKED no duplicates)
- [ ] exactly_once_v2 active
- [ ] 5 kasten kırma reproduction + fix
- [ ] 15+ integration test passing
- [ ] Async API + status polling cross-bank
- [ ] Stuck saga recovery scheduler

---

## Claude-verify prompt

```
Event-driven banking project'imi banking-grade kriterlere göre değerlendir:

1. Kafka setup:
   - KRaft mode (no ZooKeeper)?
   - Schema Registry mevcut?
   - acks=all + idempotence + min.insync.replicas=2?

2. Outbox pattern:
   - TransferService 3 event same TX'te?
   - OutboxPublisher polling + ShedLock + SKIP LOCKED?
   - Failure handling + permanent failure threshold?
   - Cleanup scheduled?

3. Consumer pattern:
   - Idempotent consumer (processed_events)?
   - Manuel offset commit (MANUAL_IMMEDIATE)?
   - DefaultErrorHandler + DLT?
   - Header propagation (X-Trace-Id)?
   - CooperativeStickyAssignor?

4. Kafka Streams:
   - exactly_once_v2?
   - 1 dk window fraud detection?
   - State store + interactive query?

5. Saga:
   - State persistent (DB)?
   - 6 Kafka listener (4 step + compensation)?
   - Idempotent compensation?
   - Async API (202 + status polling)?
   - Stuck saga recovery?

6. Kasten kırma reproduction:
   - Dual-write data loss reproduced + fixed?
   - Consumer mid-crash → duplicate reproduced + fixed (idempotency)?
   - Publisher race → duplicate reproduced + fixed (SKIP LOCKED)?
   - Compensation duplicate reproduced + fixed?

7. Integration tests:
   - TestContainers Kafka + Postgres + Schema Registry?
   - 15+ test passing?
   - End-to-end HTTP → outbox → Kafka → consumer test?

8. Banking specific:
   - PII (cardPan, TC No) payload'da YOK?
   - Outbox event ID = consumer idempotency key?
   - Eventual consistency UX impact düşünülmüş?

9. Monitoring:
   - Producer/consumer/saga metrics?
   - DLT alert ops?
   - Grafana dashboard?

10. Anti-pattern:
    - Dual-write var mı?
    - Auto-commit?
    - Saga state in-memory?
    - 2PC kullanılmış mı?
    - exactly_once_v2 yok?

Her madde için PASS / FAIL / EKSIK işaretle, kanıt göster (file path veya code reference).
```

---

## Defter notları (20 madde)

1. "Dual-write probleminin 3 patolojik senaryosu — canlı reproduce ettim: ____."
2. "Outbox pattern atomicity DB transaction'ından gelir — açıklama: ____."
3. "Outbox publisher FOR UPDATE SKIP LOCKED multi-instance neden gerekli: ____."
4. "@Transactional outbox event creation — caller's tx içinde MANDATORY: ____."
5. "Idempotent consumer processed_events pattern + cleanup TTL: ____."
6. "exactly_once_v2 = transactional producer + read_committed consumer + idempotent consumer: ____."
7. "CooperativeStickyAssignor STW pause kazancı: ____."
8. "DefaultErrorHandler + DLT vs @RetryableTopic non-blocking trade-off: ____."
9. "Kafka Streams real-time fraud detection — production vs Phase 4 PL/SQL alternative: ____."
10. "Saga orchestration vs choreography banking için karar matrisi: ____."
11. "2PC vs Saga — neden 2PC banking için yasak: ____."
12. "Compensating action 5 kuralı (idempotent, audit, semantic, order, local-atomic): ____."
13. "Saga stuck timeout banking SLA için ne kadar (rakam): ____."
14. "Async HTTP 202 + status polling banking UX impact: ____."
15. "Multi-instance publisher fairness (SKIP LOCKED) + integration test sonucu: ____."
16. "Kafka Streams state store local RocksDB + changelog backup: ____."
17. "Schema Registry + Avro vs JSON without schema — banking için fark: ____."
18. "Outbox event ID = consumer idempotency key — pattern detayı: ____."
19. "Eventual consistency UX (transfer immediate, notification delayed) — kullanıcı feedback'i: ____."
20. "TR bank batch (Phase 5) + event-driven (Phase 6) hybrid mimari: ____."
