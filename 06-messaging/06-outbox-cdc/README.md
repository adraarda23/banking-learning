# Topic 6.6 — Outbox Pattern & CDC

## Hedef

Banking için en kritik integration pattern'lerinden biri: **dual-write problem**'i ve **outbox pattern** ile çözümü. CDC (Change Data Capture) ile Debezium kullanımı, polling publisher pattern, eventual consistency trade-offs. Transfer + Kafka event publish'in **atomicity garantisi**.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Topic 6.1-6.5 bitti (Kafka core + Streams)
- Phase 1-2: transfer service, transaction management, JPA
- Phase 4'teki Oracle migration (PostgreSQL veya Oracle outbox tablosu)

---

## Kavramlar

### 1. Dual-write problem — kabul edilemez senaryo

Banking transfer endpoint:

```java
@Service
public class TransferService {
    
    private final AccountRepository accountRepo;
    private final TransferRepository transferRepo;
    private final KafkaTemplate<String, TransferEvent> kafkaTemplate;
    
    @Transactional
    public Transfer execute(TransferRequest req) {
        // 1. Domain operation
        Account from = accountRepo.findById(req.fromAccountId).orElseThrow();
        Account to = accountRepo.findById(req.toAccountId).orElseThrow();
        from.withdraw(req.amount);
        to.deposit(req.amount);
        accountRepo.save(from);
        accountRepo.save(to);
        
        Transfer transfer = new Transfer(req);
        transferRepo.save(transfer);
        
        // 2. Kafka publish (dual-write)
        kafkaTemplate.send("banking.transfers", transfer.getId(), 
            new TransferCompletedEvent(transfer));
        
        return transfer;
    }
}
```

**Üç patolojik senaryo:**

#### Senaryo A: DB commit OK, Kafka publish FAIL

```
1. DB commit ✓
2. Network glitch / Kafka down
3. Method exception → caller'a error
4. Müşteri retry yapar → idempotency yoksa double transfer
```

**Sonuç:** Para hareket etti ama event yayınlanmadı. Notification gönderilmez, audit eksik. Banking audit'te **kayıp event** ciddi sorun.

#### Senaryo B: Kafka publish OK, DB commit FAIL

```
1. Spring @Transactional açık
2. kafkaTemplate.send() — Kafka transactional değil → ASYNC publish
3. Domain logic devam, son adımda exception (constraint violation)
4. @Transactional rollback → DB unchanged
5. AMA Kafka'ya event GİTTİ
```

**Sonuç:** Event yayınlandı, transfer DB'de yok. Notification "transfer tamamlandı" der ama transfer **yok**. Müşteri panik, support yığılır.

#### Senaryo C: Both OK, ack lost in transit

```
1. DB commit ✓
2. Kafka publish ✓
3. Kafka ack network'te kayboldu
4. Producer retry → duplicate publish
```

**Idempotent producer ile** çözülür (Topic 6.2). Ama dual-write'ın diğer iki senaryosu çözülmez.

**Banking için kabul edilemez.** Atomicity şart.

### 2. Outbox pattern — atomicity garantisi

**Prensip:** DB transaction'ında **2 tabloya birden yaz**:
1. Domain table (`transfers`)
2. Outbox table (`outbox_events`)

Ayrı bir **publisher process** outbox'ı poll'lar, Kafka'ya gönderir, mark as published.

```
┌─────────────────────────────────────────────┐
│ Service @Transactional                       │
│   INSERT transfers ... (domain)              │
│   INSERT outbox_events ... (event)           │
│ COMMIT ← atomic                              │
└─────────────────────────────────────────────┘
              ↓
        Outbox table
              ↓
┌─────────────────────────────────────────────┐
│ Outbox Publisher (polling job)               │
│   SELECT * FROM outbox WHERE PENDING         │
│   FOR EACH event:                             │
│     kafka.send(...)                          │
│     UPDATE outbox SET status='PUBLISHED'     │
└─────────────────────────────────────────────┘
              ↓
            Kafka
```

**Garantiler:**
- DB commit ✓ → outbox event ✓ → eventually Kafka'ya gider
- DB commit FAIL → outbox event yok → Kafka'ya GİDEMEZ
- Kafka publish fail → outbox PENDING kalır → bir sonraki poll'da tekrar dene

Bu **atomicity DB transaction'ından** gelir. Kafka publish eventual consistency (saniyeler).

### 3. Outbox schema

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

-- Partial index — sadece PENDING kayıtlar (publisher hızlı bulsun)
CREATE INDEX idx_outbox_pending 
    ON outbox_events(status, created_at) 
    WHERE status = 'PENDING';

-- Cleanup için
CREATE INDEX idx_outbox_published_at 
    ON outbox_events(published_at) 
    WHERE status = 'PUBLISHED';
```

**Önemli kolonlar:**
- `aggregate_type` (e.g. "Transfer"): Hangi domain entity
- `aggregate_id`: Entity'nin ID'si (Kafka partition key olarak da kullan)
- `event_type` (e.g. "TransferCompleted"): Event tipi
- `payload`: Serialized event (JSON, Avro)
- `failure_count`: Retry sayısı
- `status`: Lifecycle

### 4. Service tarafı — same transaction

```java
@Service
@Slf4j
public class TransferService {
    
    private final AccountRepository accountRepo;
    private final TransferRepository transferRepo;
    private final OutboxRepository outboxRepo;
    private final ObjectMapper objectMapper;
    
    @Transactional   // Spring tx
    public Transfer execute(TransferRequest req) {
        // 1. Domain operation
        Account from = accountRepo.findById(req.fromAccountId).orElseThrow();
        Account to = accountRepo.findById(req.toAccountId).orElseThrow();
        from.withdraw(req.amount);
        to.deposit(req.amount);
        accountRepo.save(from);
        accountRepo.save(to);
        
        Transfer transfer = transferRepo.save(new Transfer(req));
        
        // 2. Outbox event — same transaction
        TransferCompletedEvent event = TransferCompletedEvent.from(transfer);
        outboxRepo.save(new OutboxEvent(
            "Transfer",
            transfer.getId().toString(),
            "TransferCompleted",
            serializeAsJson(event),
            OutboxStatus.PENDING
        ));
        
        // 3. Audit event — same transaction
        AuditEvent audit = AuditEvent.builder()
            .actorId(req.getUserId())
            .action("TRANSFER_EXECUTED")
            .resourceId(transfer.getId().toString())
            .occurredAt(Instant.now())
            .build();
        outboxRepo.save(new OutboxEvent(
            "Audit",
            transfer.getId().toString(),
            "AuditLog",
            serializeAsJson(audit),
            OutboxStatus.PENDING
        ));
        
        log.info("Transfer executed: id={}, outbox events queued: 2", transfer.getId());
        return transfer;
    }
    
    private String serializeAsJson(Object event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize event", e);
        }
    }
}
```

**Atomicity:** Hem `transfers` insert hem 2 outbox event tek `@Transactional`. Biri fail → tüm rollback.

### 5. Polling publisher

Scheduled task, outbox'ı poll'la:

```java
@Component
@Slf4j
public class OutboxPublisher {
    
    private final OutboxRepository outboxRepo;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final MeterRegistry registry;
    private static final int BATCH_SIZE = 100;
    
    @Scheduled(fixedDelay = 1000)   // her 1 sn
    @SchedulerLock(name = "outboxPublisher", lockAtMostFor = "5m")
    @Transactional
    public void publishPending() {
        List<OutboxEvent> pending = outboxRepo.findPendingForUpdate(BATCH_SIZE);
        
        if (pending.isEmpty()) return;
        
        log.debug("Publishing {} pending outbox events", pending.size());
        
        for (OutboxEvent event : pending) {
            String topic = "banking." + event.getAggregateType().toLowerCase() + 
                          "." + camelToKebab(event.getEventType());
            String partitionKey = event.getAggregateId();
            
            try {
                SendResult<String, String> result = kafkaTemplate
                    .send(topic, partitionKey, event.getPayload())
                    .get(10, TimeUnit.SECONDS);   // sync send
                
                event.setStatus(OutboxStatus.PUBLISHED);
                event.setPublishedAt(Instant.now());
                outboxRepo.save(event);
                
                registry.counter("outbox.published", "topic", topic).increment();
                
                log.debug("Published outbox event: id={}, topic={}, partition={}, offset={}",
                    event.getId(), topic,
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
                
            } catch (Exception e) {
                log.error("Failed to publish outbox event: id={}", event.getId(), e);
                event.setFailureCount(event.getFailureCount() + 1);
                event.setLastError(e.getMessage());
                
                if (event.getFailureCount() >= 10) {
                    event.setStatus(OutboxStatus.FAILED);
                    notifier.alertOps("Outbox event permanent failure: " + event.getId());
                }
                outboxRepo.save(event);
                
                registry.counter("outbox.failed", "topic", topic).increment();
            }
        }
    }
}
```

**Önemli detaylar:**
- `FOR UPDATE SKIP LOCKED` (Topic 4.6) ile **multi-instance safe** — concurrent publisher'lar farklı event'leri işler
- `@SchedulerLock` (Topic 5.6) — gereksiz birden fazla publisher run'ı engelle
- `@Transactional` — outbox lock + update tek tx
- Failure count + permanent failure threshold

Repository:

```java
public interface OutboxRepository extends JpaRepository<OutboxEvent, UUID> {
    
    @Query(value = """
        SELECT * FROM outbox_events 
        WHERE status = 'PENDING' 
        ORDER BY created_at 
        FOR UPDATE SKIP LOCKED 
        LIMIT :batchSize
        """, nativeQuery = true)
    List<OutboxEvent> findPendingForUpdate(@Param("batchSize") int batchSize);
}
```

### 6. CDC — Debezium alternatifi

Polling publisher manuel, 1 saniye latency. **Debezium** otomatik low-latency:

- PostgreSQL **WAL** (Write-Ahead Log) okur
- MySQL **binlog** okur
- Oracle **redo log** okur
- Her INSERT/UPDATE/DELETE → Kafka event yayınla

```
[Service: INSERT outbox_event] → [PostgreSQL] → [WAL]
                                                  ↓
                                         [Debezium Connector]
                                                  ↓
                                            [Kafka topic]
```

**Avantaj:** Latency ~ms (polling 1 sn'ye karşı). Polling overhead yok.

**Disadvantage:** Kafka Connect cluster operate etmek gerekli.

### 7. Debezium setup — PostgreSQL

#### Logical replication aktive et

`postgresql.conf`:

```
wal_level = logical
max_wal_senders = 4
max_replication_slots = 4
```

PostgreSQL restart. Publication yarat:

```sql
CREATE PUBLICATION dbz_outbox_publication FOR TABLE outbox_events;
```

#### Kafka Connect + Debezium connector

`docker-compose.yml`:

```yaml
services:
  kafka-connect:
    image: debezium/connect:2.5
    ports: ["8083:8083"]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: connect-cluster
      CONFIG_STORAGE_TOPIC: connect-configs
      OFFSET_STORAGE_TOPIC: connect-offsets
      STATUS_STORAGE_TOPIC: connect-status
```

Connector config (POST to /connectors):

```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "...",
    "database.dbname": "banking",
    "topic.prefix": "banking-dbz",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_outbox_publication",
    "table.include.list": "public.outbox_events",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "banking.${routedByValue}"
  }
}
```

`EventRouter` outbox satırlarını **Kafka topic'lerine route eder**:

```
outbox_events.aggregate_type='Transfer' → banking.transfer
outbox_events.aggregate_type='Audit' → banking.audit
```

Otomatik. Polling publisher'a gerek yok.

### 8. Polling vs CDC trade-off

| Kriter | Polling Publisher | CDC (Debezium) |
|---|---|---|
| Latency | 1-5 sn (poll interval) | ~ms (WAL streaming) |
| Setup complexity | Düşük (Spring scheduled) | Yüksek (Kafka Connect) |
| Operational overhead | DB poll | Connector monitoring |
| DB load | Sürekli poll (light) | WAL read (very light) |
| Recovery | Manuel restart | Built-in connector |
| Multi-instance safe | ShedLock / SKIP LOCKED | Otomatik (Connect cluster) |
| Banking adoption | Yaygın (basit) | Modern banking trend |

**Pratik tavsiye:**
- **Start with polling** — 1 günde implement
- **Scale issue olunca CDC'ye geç** — modern infra

### 9. Eventual consistency — banking impact

Outbox pattern eventual consistency:
- DB commit → outbox INSERT (sync, atomic)
- Outbox → Kafka publish (~1 sn polling, ~ms CDC)
- Consumer process (~ms)

**Banking impact:**

```
User: POST /transfers (transfer 100 TL)
Server: 201 Created (DB commit ~50ms)
User: GET /accounts/A/balance (50ms sonra)
Server: balance düşmüş (immediate consistency)

Notification SMS: ~2-5 saniye sonra gelir (eventual)
Audit log: ~2-5 saniye sonra (eventual)
```

**UI/UX:**
- Transfer ekranı: immediate success (DB)
- Notification badge: 2-5 sn sonra (optimistic UI + final reconcile)

Banking kullanıcı bunu hisseder. UX tasarımı buna göre.

### 10. Multi-event same transaction

Banking transfer → 3 farklı consumer ihtiyaç:
- Notification (SMS)
- Audit (regulatory log)
- Fraud check (real-time)

3 ayrı outbox event:

```java
@Transactional
public Transfer execute(TransferRequest req) {
    Transfer t = doTransfer(req);
    
    // 3 farklı event — all same TX
    outboxRepo.save(toOutbox("Transfer", t.getId(), "TransferCompleted", new TransferCompletedEvent(t)));
    outboxRepo.save(toOutbox("Audit", t.getId(), "AuditLog", new AuditLogEvent(t)));
    outboxRepo.save(toOutbox("Notification", t.getToAccountId(), "NotificationRequest", new NotificationRequest(t)));
    
    return t;
}
```

3 ayrı consumer 3 ayrı topic'i tüketir. Hepsi DB commit'ine atomic bağlı.

### 11. Outbox event ID = idempotency key

Consumer side:

```java
@KafkaListener(topics = "banking.transfer")
@Transactional
public void consume(@Payload TransferCompletedEvent event, 
                    @Header("eventId") String outboxEventId) {
    if (processedRepo.existsByEventId(UUID.fromString(outboxEventId))) {
        return;   // duplicate
    }
    
    notificationService.send(event);
    processedRepo.save(new ProcessedEvent(UUID.fromString(outboxEventId), "notification-service"));
}
```

Outbox `id` her event'in unique idempotency key'i. Consumer side dedup.

### 12. Cleanup — outbox büyümesin

Published event'leri sil veya archive:

```java
@Scheduled(cron = "0 0 4 * * *")
@Transactional
public void cleanupOldEvents() {
    int deleted = outboxRepo.deleteByStatusAndPublishedAtBefore(
        OutboxStatus.PUBLISHED,
        Instant.now().minus(7, ChronoUnit.DAYS)
    );
    log.info("Cleaned up {} old outbox events", deleted);
}
```

**Banking retention:**
- PUBLISHED → 7-30 gün, sonra delete (event Kafka'da + audit DB'de)
- FAILED → manuel review, indefinite

### 13. Multi-tenant outbox

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,
    ...
);

CREATE INDEX idx_outbox_pending_tenant 
    ON outbox_events(tenant_id, status, created_at) 
    WHERE status = 'PENDING';
```

Multi-tenant banking — her tenant'ın event'leri ayrı topic veya partition.

### 14. Idempotent producer + Outbox kombine

Outbox publisher Kafka'ya gönderirken:

```yaml
spring:
  kafka:
    producer:
      acks: all
      properties:
        enable.idempotence: true
```

Outbox event ID `messageKey` olarak. Aynı event 2 kez gönderilse bile broker dedup.

### 15. Outbox + transactional producer (advanced)

Outbox publisher Kafka transaction içinde çalışabilir:

```java
@Transactional   // BOTH DB and Kafka tx (chained)
public void publishPending() {
    List<OutboxEvent> pending = outboxRepo.findPendingForUpdate(100);
    
    for (OutboxEvent event : pending) {
        kafkaTemplate.send(...);
        event.setStatus(OutboxStatus.PUBLISHED);
        outboxRepo.save(event);
    }
}
```

ChainedKafkaTransactionManager ile DB + Kafka chained. Advanced setup.

### 16. Banking örnek — full TransferService + Outbox

```java
@Service
@Slf4j
public class TransferService {
    
    private final AccountRepository accountRepo;
    private final TransferRepository transferRepo;
    private final OutboxEventService outboxEventService;
    
    @Transactional
    public Transfer execute(TransferRequest req, UUID userId) {
        // Validate idempotency
        if (transferRepo.existsByIdempotencyKey(req.idempotencyKey())) {
            return transferRepo.findByIdempotencyKey(req.idempotencyKey()).orElseThrow();
        }
        
        // Domain
        Account from = accountRepo.findByIdAndLock(req.fromAccountId()).orElseThrow();
        Account to = accountRepo.findByIdAndLock(req.toAccountId()).orElseThrow();
        from.withdraw(req.amount(), req.currency());
        to.deposit(req.amount(), req.currency());
        
        Transfer transfer = new Transfer(req, userId);
        transfer = transferRepo.save(transfer);
        accountRepo.save(from);
        accountRepo.save(to);
        
        // Outbox events
        outboxEventService.publish(
            "Transfer", transfer.getId().toString(), "TransferCompleted",
            new TransferCompletedEvent(transfer)
        );
        
        outboxEventService.publish(
            "Audit", transfer.getId().toString(), "AuditLog",
            new AuditLogEvent("TRANSFER_EXECUTED", userId, transfer.getId(), Instant.now())
        );
        
        outboxEventService.publish(
            "Notification", transfer.getToAccountId().toString(), "NotificationRequest",
            new NotificationRequest("TRANSFER_RECEIVED", transfer.getToAccountId(), 
                transfer.getAmount(), transfer.getCurrency())
        );
        
        log.info("Transfer executed: id={}, outbox events: 3", transfer.getId());
        return transfer;
    }
}

@Service
public class OutboxEventService {
    
    private final OutboxRepository outboxRepo;
    private final ObjectMapper objectMapper;
    
    @Transactional(propagation = Propagation.MANDATORY)   // caller's tx içinde olmak ZORUNDA
    public void publish(String aggregateType, String aggregateId, 
                        String eventType, Object payload) {
        try {
            String json = objectMapper.writeValueAsString(payload);
            outboxRepo.save(new OutboxEvent(
                UUID.randomUUID(),
                aggregateType,
                aggregateId,
                eventType,
                json,
                OutboxStatus.PENDING,
                Instant.now()
            ));
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Event serialization failed", e);
        }
    }
}
```

`PROPAGATION_MANDATORY` — caller mutlaka transaction'da olmalı. Outbox SADECE business transaction içinde.

### 17. Banking anti-pattern'leri

**Anti-pattern 1: Dual-write (outbox YOK)**

Veri tutarsızlığı kaçınılmaz. Banking için **YASAK**.

**Anti-pattern 2: Outbox publish'i `@TransactionalEventListener(AFTER_COMMIT)` ile**

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(TransferEvent event) {
    kafkaTemplate.send(...);   // ❌ transaction dışı
}
```

**Sorun:** `AFTER_COMMIT` listener'ı transaction'dan sonra çalışır. Burada Kafka publish fail olursa → event kayıp. Outbox table tercih.

**Anti-pattern 3: Polling publisher tekil instance**

3 app instance, ShedLock yok → 3 publisher concurrent run → DB lock contention.

**Çözüm:** ShedLock + `FOR UPDATE SKIP LOCKED`.

**Anti-pattern 4: Outbox cleanup yok**

Tablonun büyümesi → query yavaşlar → publisher latency artar. Scheduled cleanup şart.

**Anti-pattern 5: Outbox event payload denormalized**

```java
outboxRepo.save(new OutboxEvent(...,
    "TransferCompleted",
    "{\"transferId\": \"...\"}"));   // ❌ sadece ID, consumer lookup yapacak
```

Consumer tekrar DB'ye gidip detail çekecek. Outbox payload **self-contained** (Avro schema, full event).

**Anti-pattern 6: Outbox event ordering varsayımı**

`created_at` order ile poll'lanır ama Kafka publish sırası garanti değil (idempotent producer ile dedup'lanır). Consumer ordering varsayma.

**Anti-pattern 7: Outbox event'inde sensitive data**

```java
outboxRepo.save(new OutboxEvent(..., 
    "{\"cardPan\": \"4111-1111-1111-1234\"}"));   // ❌ PCI-DSS violation
```

Outbox tablosu PII içerirse PCI-DSS scope'a dahil. Tokenize edilmiş data + encrypted DB column.

---

## Önemli olabilecek araştırma kaynakları

- "Microservices Patterns" (Chris Richardson) — Saga + Outbox chapter
- Debezium documentation
- Confluent blog — Outbox pattern deep dives
- "Designing Event-Driven Systems" (Ben Stopford)
- Apache Kafka Connect documentation
- PostgreSQL logical replication docs

---

## Mini task'ler

### Task 6.6.1 — Outbox table migration (15 dk)

`V_outbox_events.sql` migration yaz. Yukarıdaki schema + partial index. Test: SELECT FROM outbox_events çalışsın.

### Task 6.6.2 — TransferService outbox entegrasyonu (45 dk)

`TransferService.execute()` içinde 3 outbox event (Transfer, Audit, Notification). `@Transactional`'da hepsi atomic.

Test: Mid-execution exception fırlat → outbox'ta hiç event olmamalı (rollback).

### Task 6.6.3 — Polling publisher + ShedLock + SKIP LOCKED (60 dk)

```java
@Scheduled(fixedDelay = 1000)
@SchedulerLock(name = "outboxPublisher", lockAtMostFor = "5m")
@Transactional
public void publishPending() { ... }
```

`OutboxRepository.findPendingForUpdate(100)` native query `FOR UPDATE SKIP LOCKED` ile.

Multi-instance test: 3 publisher instance lokal başlat → her event sadece bir kez publish olmalı.

### Task 6.6.4 — Failure handling + dead letter (45 dk)

Publisher Kafka send fail olunca:
- `failure_count++`
- `last_error = e.getMessage()`
- failure_count >= 10 → status = FAILED + alert

Test: Kafka stop → 10 retry sonra FAILED + alert.

### Task 6.6.5 — Cleanup scheduled job (30 dk)

Her gece 04:00, PUBLISHED + 7+ gün önceki event'leri delete.

### Task 6.6.6 — Debezium Postgres setup (60 dk)

`docker-compose.yml`'a Kafka Connect ekle. Postgres logical replication aktive et. Connector config POST. Test: outbox INSERT → Kafka topic'te otomatik mesaj.

### Task 6.6.7 — Consumer idempotency outbox ID ile (30 dk)

Consumer tarafında outbox `id` header olarak gelir. `processed_events` tablosunda dedup.

### Task 6.6.8 — End-to-end test (45 dk)

TransferService execute → outbox 3 event → 3 consumer (notification, audit, fraud) tüketir → side effects (SMS, log, score).

TestContainers + Postgres + Kafka full integration.

---

## Test yazma rehberi

### Test 6.6.1 — Outbox event in same transaction

```java
@SpringBootTest
@Testcontainers
@Transactional
class TransferServiceOutboxIT {
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @Autowired TransferService transferService;
    @Autowired OutboxRepository outboxRepo;
    
    @Test
    void shouldCreate3OutboxEventsAtomically() {
        TransferRequest req = createValidRequest();
        Transfer transfer = transferService.execute(req, UUID.randomUUID());
        
        List<OutboxEvent> events = outboxRepo.findByAggregateId(transfer.getId().toString());
        assertThat(events).hasSize(3);
        
        Set<String> types = events.stream().map(OutboxEvent::getEventType).collect(Collectors.toSet());
        assertThat(types).containsExactlyInAnyOrder(
            "TransferCompleted", "AuditLog", "NotificationRequest"
        );
        
        assertThat(events).allMatch(e -> e.getStatus() == OutboxStatus.PENDING);
    }
    
    @Test
    @Rollback
    void shouldRollbackOutboxOnFailure() {
        TransferRequest req = createRequestThatWillFail();
        
        assertThatThrownBy(() -> transferService.execute(req, UUID.randomUUID()))
            .isInstanceOf(InsufficientFundsException.class);
        
        // Transfer YOK + outbox YOK (rollback)
        assertThat(transferRepo.count()).isEqualTo(0);
        assertThat(outboxRepo.count()).isEqualTo(0);
    }
}
```

### Test 6.6.2 — Publisher behavior

```java
@SpringBootTest
@Testcontainers
class OutboxPublisherIT {
    
    @Container static PostgreSQLContainer<?> postgres = ...;
    @Container static KafkaContainer kafka = ...;
    
    @Autowired OutboxPublisher publisher;
    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, String> kafkaTemplate;
    
    @Test
    void shouldPublishPendingEventsAndMarkAsPublished() throws Exception {
        OutboxEvent event = createPendingEvent();
        outboxRepo.save(event);
        
        publisher.publishPending();
        
        // Mark as published
        OutboxEvent updated = outboxRepo.findById(event.getId()).orElseThrow();
        assertThat(updated.getStatus()).isEqualTo(OutboxStatus.PUBLISHED);
        assertThat(updated.getPublishedAt()).isNotNull();
        
        // Kafka'da mesaj
        KafkaConsumer<String, String> consumer = createTestConsumer("banking.transfer.transfer-completed");
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));
        assertThat(records).isNotEmpty();
    }
    
    @Test
    void shouldRetryOnKafkaFailure() throws Exception {
        kafka.stop();
        
        OutboxEvent event = createPendingEvent();
        outboxRepo.save(event);
        
        publisher.publishPending();
        
        OutboxEvent updated = outboxRepo.findById(event.getId()).orElseThrow();
        assertThat(updated.getStatus()).isEqualTo(OutboxStatus.PENDING);
        assertThat(updated.getFailureCount()).isEqualTo(1);
        assertThat(updated.getLastError()).isNotBlank();
    }
    
    @Test
    void shouldMarkAsFailedAfter10Attempts() throws Exception {
        kafka.stop();
        
        OutboxEvent event = createPendingEvent();
        event.setFailureCount(9);
        outboxRepo.save(event);
        
        publisher.publishPending();
        
        OutboxEvent updated = outboxRepo.findById(event.getId()).orElseThrow();
        assertThat(updated.getStatus()).isEqualTo(OutboxStatus.FAILED);
    }
    
    @Test
    void multipleInstancesShouldNotPublishDuplicates() throws Exception {
        for (int i = 0; i < 100; i++) {
            outboxRepo.save(createPendingEvent());
        }
        
        // 5 concurrent publishers
        ExecutorService exec = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5; i++) {
            exec.submit(() -> publisher.publishPending());
        }
        exec.shutdown();
        exec.awaitTermination(10, TimeUnit.SECONDS);
        
        // Kafka'da tam 100 mesaj
        Long publishedCount = countKafkaMessages("banking.transfer.transfer-completed");
        assertThat(publishedCount).isEqualTo(100);   // duplicate yok (SKIP LOCKED)
    }
}
```

---

## Claude-verify prompt

```
Outbox pattern kodumu banking-grade kriterlere göre değerlendir:

1. Service tarafı:
   - Transfer execute outbox event same transaction'da mı?
   - @Transactional ile atomic guarantee?
   - 3 farklı event (Transfer/Audit/Notification) tek tx'te?
   - PROPAGATION_MANDATORY OutboxEventService'te?

2. Outbox tablosu:
   - aggregate_type + aggregate_id + event_type + payload?
   - status enum (PENDING/PUBLISHED/FAILED)?
   - Partial index on PENDING?
   - failure_count + last_error error tracking?

3. Publisher:
   - Polling @Scheduled + @SchedulerLock?
   - FOR UPDATE SKIP LOCKED multi-instance safe?
   - Batch size makul (100)?
   - Failure retry + permanent failure threshold?

4. Anti-pattern:
   - Dual-write (kafka publish service'te direct) yok mu?
   - @TransactionalEventListener AFTER_COMMIT pattern kullanılmamış mı?
   - Outbox payload sensitive data içeriyor mu? (Olmamalı)
   - Outbox cleanup job var mı?

5. CDC (Debezium):
   - Debezium alternative değerlendirildi mi?
   - PostgreSQL logical replication + publication setup yapıldı mı?
   - EventRouter ile topic routing?

6. Consumer side:
   - Outbox event ID idempotency key olarak kullanılıyor mu?
   - processed_events tablosunda dedup?

7. Banking specific:
   - 3 farklı consumer için 3 ayrı outbox event?
   - Eventual consistency UX impact düşünülmüş mü?

8. Test:
   - Same transaction outbox creation test?
   - Rollback → outbox empty test?
   - Multi-instance publisher → no duplicates test?
   - Kafka down → retry + failure count test?

9. Performance:
   - Outbox tablosu cleanup scheduled?
   - Index PENDING-only?
   - Publisher batch size optimization?

10. Banking ops:
    - FAILED event'ler için alert ops?
    - Manuel resolution workflow?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `outbox_events` tablosu migration
- [ ] TransferService 3 outbox event same TX
- [ ] OutboxPublisher polling + ShedLock + SKIP LOCKED
- [ ] Failure handling + permanent failure threshold
- [ ] Cleanup scheduled job (7 gün retention)
- [ ] Debezium Postgres setup denedim
- [ ] Consumer outbox ID idempotency
- [ ] End-to-end test: service → outbox → Kafka → consumer
- [ ] Multi-instance publisher test (no duplicates)
- [ ] Rollback test (TX fail → outbox empty)

---

## Defter notları (10 madde)

1. "Dual-write problem 3 patolojik senaryosu: ____."
2. "Outbox pattern atomicity DB transaction'ından gelir — neden: ____."
3. "Polling vs CDC trade-off + banking için seçim: ____."
4. "FOR UPDATE SKIP LOCKED multi-instance publisher için neden gerekli: ____."
5. "PROPAGATION_MANDATORY OutboxEventService'te neden: ____."
6. "@TransactionalEventListener AFTER_COMMIT anti-pattern: ____."
7. "Outbox event ID = consumer idempotency key: ____."
8. "Banking eventual consistency UX impact (transfer immediate, SMS delayed): ____."
9. "Debezium EventRouter Kafka topic routing pattern: ____."
10. "Outbox payload self-contained (denormalized DEĞİL) neden: ____."
