# Topic 6.2 — Kafka Producer Design

## Hedef

Kafka producer'ı banking-grade seviyede konfigüre etmek. `acks`, `enable.idempotence`, transactional producer, batch tuning, compression, partition key stratejisi, error handling. Banking event publishing (TransferCompleted, FraudDetected, AccountStatusChanged) için **veri kaybı olmayan** producer setup'ı yapabilmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 6.1 (Kafka Architecture) bitti — broker, topic, partition, replica, ISR biliyorsun
- Spring Boot + KafkaTemplate'i basit seviyede kullandın
- BigDecimal money handling Phase 1'den hazır

---

## Kavramlar

### 1. Producer'ın iç çalışması

KafkaProducer mesajı 4 aşamada gönderir:

```
1. Serializer        → byte[] (Avro/JSON/Protobuf)
2. Partitioner       → partition number (hash(key) % partitions veya custom)
3. RecordAccumulator → batch (memory buffer)
4. Sender thread     → network'e batch gönder
```

**Asenkronluk:** `producer.send(record)` **immediately** döner — accumulator'a koyar. Sender thread asenkron olarak broker'a iletir.

```java
Future<RecordMetadata> future = producer.send(record);
// future henüz ack almadı, kuyruga konmuş olabilir
```

Banking impact: `send` döndü diye "yazıldı" denemez. Callback veya `get()` ile ack bekle.

### 2. `acks` — durability vs latency

Producer broker'dan **kaç onay** beklesin?

#### `acks=0` — fire-and-forget

```yaml
spring:
  kafka:
    producer:
      acks: 0
```

Producer broker'ın ack'ini **beklemez**. En yüksek throughput, **en yüksek veri kaybı riski**.

**Banking'de YASAK.** Audit, transaction, notification event'leri kaybolamaz.

#### `acks=1` — leader ack

```yaml
acks: 1
```

Leader broker yazıldığını ack eder. Replica'lara kopyalanmadan önce.

**Tehlike:** Leader fail olur, replica henüz almamış → mesaj kayıp.

**Banking'de yetersiz.** Regulatory için durability şart.

#### `acks=all` (veya `-1`) — full ISR ack

```yaml
acks: all
```

Leader + **all in-sync replicas (ISR)** yazdı dediğinde ack. En güvenli.

Beraberinde:
- `min.insync.replicas=2` (topic config) — en az 2 replica ISR'da olmalı, yoksa producer fail
- `replication.factor=3` — tipik banking setup (1 leader + 2 replica)

**Banking standard:**

```yaml
spring:
  kafka:
    producer:
      acks: all
      properties:
        min.insync.replicas: 2
```

Topic create:

```bash
kafka-topics --bootstrap-server kafka:9092 \
  --create --topic banking.transfers \
  --partitions 10 --replication-factor 3 \
  --config min.insync.replicas=2
```

### 3. `enable.idempotence` — retry'a karşı duplicate koruması

**Senaryo:** `acks=all`, broker yazdı, ack producer'a dönerken network glitch. Producer "ack gelmedi" deyip retry → broker'a **2. kez yazılır**.

**Çözüm:** Idempotent producer.

```yaml
spring:
  kafka:
    producer:
      acks: all
      properties:
        enable.idempotence: true
```

**Nasıl çalışır:**

1. Producer her batch'e `Producer ID (PID)` + `sequence number` ekler
2. Broker per-partition `(PID, sequence)` tracking yapar
3. Aynı `(PID, sequence)` ikinci kez gelirse → broker **drop** eder

Garanti: Per-partition **exactly-once-write** (per producer session).

**Kurallar (`enable.idempotence=true` aktivasyonu):**
- `acks=all` (otomatik set olur)
- `retries >= 0` (default Integer.MAX_VALUE)
- `max.in.flight.requests.per.connection <= 5`

**Banking standardı:** Her zaman `enable.idempotence=true`. Network glitch real, duplicate notification = customer noise.

### 4. `max.in.flight.requests.per.connection` — ordering vs throughput

Producer bir broker connection'ında **kaç paralel request** uçurabilir.

```yaml
max.in.flight.requests.per.connection: 5   # default
```

**Tuzak:** `enable.idempotence=false` + `max.in.flight > 1` + `retries > 0` → **out-of-order delivery**.

Senaryo:
```
M1 gönder, M2 gönder (parallel)
M1 fail, retry
Broker'a sıra: M2, M1, M1
```

**`enable.idempotence=true` ile fix.** Broker sequence number ile sıralar.

Banking impact: Account event'lerinin sırası önemli (`Opened` → `Deposited` → `Closed`). Idempotence + max.in.flight=5 → hem ordering hem throughput.

### 5. Batching — throughput optimization

Producer mesajları batch'leyerek gönderir. Batch'ler **per-partition** memory buffer'da birikir.

```yaml
spring:
  kafka:
    producer:
      batch-size: 16384      # 16KB — buffer dolunca gönder
      linger-ms: 10          # veya 10ms bekle, dolmasa bile gönder
      buffer-memory: 33554432  # 32MB total accumulator
```

**Trade-off:**

| Config | Throughput | Latency |
|---|---|---|
| `batch-size=16384`, `linger-ms=0` | Düşük | En düşük |
| `batch-size=16384`, `linger-ms=10` | Orta | +10ms |
| `batch-size=65536`, `linger-ms=50` | Yüksek | +50ms |

**Banking pratiği:**

- **Low-volume critical events** (TransferCompleted): `batch-size=16384`, `linger-ms=0-5`
- **High-volume analytics events** (CustomerActivity): `batch-size=65536`, `linger-ms=20-50`

### 6. Compression

```yaml
spring:
  kafka:
    producer:
      compression-type: lz4   # snappy, gzip, zstd alternatives
```

Compression batch-level. CPU cost vs network/disk save.

| Codec | CPU | Ratio | Banking |
|---|---|---|---|
| `none` | 0 | 1x | ❌ |
| `lz4` | Düşük | 2-3x | ✓ default |
| `snappy` | Düşük | 2-3x | OK |
| `gzip` | Yüksek | 4-5x | High-volume analytics |
| `zstd` | Orta | 4-6x | Modern, banking trend |

**Banking JSON payloads %50-70 kompres edilir.** Always enable.

### 7. Transactional producer — multi-event atomicity

Birden fazla event birden gönderiliyor — ya hepsi yazılsın ya hiçbiri.

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("acks", "all");
props.put("enable.idempotence", "true");
props.put("transactional.id", "transfer-producer-1");   // UNIQUE per producer instance

try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
    producer.initTransactions();
    
    try {
        producer.beginTransaction();
        
        producer.send(new ProducerRecord<>("banking.transfers", transferId, transferEvent));
        producer.send(new ProducerRecord<>("banking.audit", transferId, auditEvent));
        producer.send(new ProducerRecord<>("banking.notifications", recipientId, notificationEvent));
        
        producer.commitTransaction();
    } catch (Exception e) {
        producer.abortTransaction();
        throw e;
    }
}
```

**Garanti:** Üç event ya **hepsi commit** ya **hiçbiri** consumer'a görünmez.

Consumer side:

```yaml
spring:
  kafka:
    consumer:
      isolation-level: read_committed
```

`read_committed` consumer sadece commit edilmiş transaction event'leri görür.

#### Spring Kafka transactional

```java
@Configuration
public class KafkaTransactionalConfig {
    
    @Bean
    public ProducerFactory<String, Object> transactionalProducerFactory(KafkaProperties props) {
        DefaultKafkaProducerFactory<String, Object> factory = 
            new DefaultKafkaProducerFactory<>(props.buildProducerProperties());
        factory.setTransactionIdPrefix("tx-transfer-");
        return factory;
    }
    
    @Bean
    public KafkaTransactionManager<String, Object> kafkaTransactionManager(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTransactionManager<>(producerFactory);
    }
}

@Service
public class TransferEventPublisher {
    
    @Autowired KafkaTemplate<String, Object> template;
    
    @Transactional("kafkaTransactionManager")
    public void publishMultiple(Transfer transfer, AuditEvent audit, NotificationEvent notif) {
        template.send("banking.transfers", transfer.getId(), transfer);
        template.send("banking.audit", transfer.getId(), audit);
        template.send("banking.notifications", notif.recipient(), notif);
    }
}
```

**DB + Kafka kombine transactional** mümkün değil (different resources). **Outbox pattern** (Topic 6.6) ile çözeriz.

### 8. Send modes

#### Fire-and-forget

```java
template.send(record);
// future ignored, hata feedback yok
```

**Sadece** fire-and-forget OK ise: durability gerekli olmayan event'ler (heartbeat, debug log).

**Banking için NADİREN.** Audit/transfer kaybı kabul edilemez.

#### Synchronous

```java
RecordMetadata metadata = template.send(record).get();
// blocks until ack — yavaş ama hata visibility
```

Banking pratiği: **Critical operation** (transfer event)'in immediate kontrolü için sync kullanılabilir. Ama hot path'te yavaş.

#### Async with callback

```java
CompletableFuture<SendResult<String, Object>> future = template.send(record);

future.whenComplete((result, ex) -> {
    if (ex != null) {
        log.error("Failed to publish event: {}", record, ex);
        // Outbox fallback'e yaz, alert
        outboxFallback.save(record);
        meterRegistry.counter("kafka.send.failure").increment();
    } else {
        log.debug("Published: partition={}, offset={}",
            result.getRecordMetadata().partition(),
            result.getRecordMetadata().offset());
        meterRegistry.counter("kafka.send.success").increment();
    }
});
```

**Banking standard:** Async + callback. Hot path bloklamaz, hata yakalanır.

### 9. Partition key seçimi

```java
template.send("banking.transfers", accountId.toString(), event);
```

Partition key = mesajın **partition'ını belirler** (default partitioner: `hash(key) % partitions`).

**Faydası — per-key ordering:**

Aynı `accountId` ile gönderilen tüm event'ler **aynı partition**'a gider. Consumer sıralı tüketir.

```
Partition 0: Acc-A.Opened → Acc-A.Deposited → Acc-A.Withdrawn
Partition 5: Acc-B.Opened → Acc-B.Deposited → Acc-B.Closed
```

`accountId=A` event'lerinin sırası **garanti**. `accountId=B` ile sırası **garanti değil** (farklı partition).

**Banking örneği — doğru key:**

- **TransferCompleted:** key = `fromAccountId` (account perspective)
- **CardTransaction:** key = `cardId`
- **CustomerActivity:** key = `customerId`
- **FraudAlert:** key = `accountId`

**Anti-pattern: key = random UUID veya timestamp**

Her event farklı partition'a → ordering yok. Banking event'leri yanlış sırada consume edilebilir.

#### Custom partitioner

Bazı senaryolarda hash partition yetmez:

```java
public class TenantPartitioner implements Partitioner {
    
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, 
                         Object value, byte[] valueBytes, Cluster cluster) {
        TransferEvent event = (TransferEvent) value;
        String tenant = event.getTenant();
        
        // Premium müşteriler için dedicated partition 0-2 (priority)
        if (event.isPremium()) {
            return Math.abs(tenant.hashCode()) % 3;
        }
        // Normal müşteriler 3-9
        return 3 + Math.abs(tenant.hashCode()) % 7;
    }
}
```

Banking: Multi-tenant — tenant başına partition isolation.

### 10. Schema Registry + Avro/Protobuf — banking-grade

JSON serialization OK ama:
- Schema evolution yok
- Payload büyük
- Type-safe değil

**Avro / Protobuf:**
- Generated Java classes (compile-time type safety)
- Schema Registry (Confluent veya Apicurio)
- Backward/forward compatibility check

```yaml
spring:
  kafka:
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
    properties:
      schema.registry.url: http://schema-registry:8081
```

`TransferCompleted.avsc`:

```json
{
  "type": "record",
  "name": "TransferCompleted",
  "namespace": "com.mavibank.banking.events",
  "fields": [
    {"name": "transferId", "type": {"type": "string", "logicalType": "uuid"}},
    {"name": "fromAccountId", "type": {"type": "string", "logicalType": "uuid"}},
    {"name": "toAccountId", "type": {"type": "string", "logicalType": "uuid"}},
    {"name": "amount", "type": {"type": "bytes", "logicalType": "decimal", "precision": 19, "scale": 4}},
    {"name": "currency", "type": "string"},
    {"name": "occurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
  ]
}
```

Maven plugin Java class generate:

```xml
<plugin>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro-maven-plugin</artifactId>
    <version>1.11.3</version>
    <executions>
        <execution>
            <goals><goal>schema</goal></goals>
            <configuration>
                <sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory>
                <outputDirectory>${project.build.directory}/generated-sources/avro/</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```java
TransferCompleted event = TransferCompleted.newBuilder()
    .setTransferId(transferId.toString())
    .setFromAccountId(fromId.toString())
    .setToAccountId(toId.toString())
    .setAmount(amount.unscaledValue().toByteArray())
    .setCurrency("TRY")
    .setOccurredAt(Instant.now().toEpochMilli())
    .build();

template.send("banking.transfers", fromId.toString(), event);
```

**Banking adoption:** Modern banking projects → Avro. Eski JSON-based migration trend.

### 11. Banking örnek — full producer setup

```java
@Component
@Slf4j
public class TransferEventPublisher {
    
    private static final String TOPIC = "banking.transfers";
    
    private final KafkaTemplate<String, TransferCompleted> template;
    private final MeterRegistry registry;
    private final OutboxRepository outboxRepo;
    
    public TransferEventPublisher(
            KafkaTemplate<String, TransferCompleted> template,
            MeterRegistry registry,
            OutboxRepository outboxRepo) {
        this.template = template;
        this.registry = registry;
        this.outboxRepo = outboxRepo;
    }
    
    public CompletableFuture<SendResult<String, TransferCompleted>> publish(TransferCompleted event) {
        String partitionKey = event.getFromAccountId();
        
        log.debug("Publishing transfer event: id={}, key={}", event.getTransferId(), partitionKey);
        
        CompletableFuture<SendResult<String, TransferCompleted>> future = 
            template.send(TOPIC, partitionKey, event);
        
        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to publish transfer event: id={}", event.getTransferId(), ex);
                
                // Outbox fallback — guaranteed eventual delivery
                outboxRepo.save(new OutboxEvent(
                    event.getTransferId(),
                    "Transfer",
                    "TransferCompleted",
                    serializeAvro(event),
                    OutboxStatus.PENDING
                ));
                
                registry.counter("kafka.publish.failure", "topic", TOPIC).increment();
            } else {
                log.debug("Published successfully: id={}, partition={}, offset={}",
                    event.getTransferId(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
                
                registry.counter("kafka.publish.success", "topic", TOPIC).increment();
                registry.summary("kafka.publish.latency", "topic", TOPIC)
                    .record(System.currentTimeMillis() - event.getOccurredAt());
            }
        });
        
        return future;
    }
}
```

Production config:

```yaml
spring:
  kafka:
    bootstrap-servers: kafka-1:9092,kafka-2:9092,kafka-3:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      acks: all
      retries: 2147483647   # essentially infinite (idempotence handles dedup)
      batch-size: 16384
      linger-ms: 5
      buffer-memory: 33554432
      compression-type: lz4
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        delivery.timeout.ms: 120000   # 2 dakika total timeout
        request.timeout.ms: 30000
        schema.registry.url: http://schema-registry:8081
```

### 12. Error handling matrix

| Exception | Anlamı | Strateji |
|---|---|---|
| `RetriableException` | Geçici (network, leader change) | Auto-retry (producer içinde) |
| `TimeoutException` | `delivery.timeout.ms` aşıldı | Outbox fallback |
| `RecordTooLargeException` | Mesaj `max.request.size`'tan büyük | Sızdırma — DLQ |
| `SerializationException` | Avro/JSON serialize fail | DLQ, code bug |
| `AuthenticationException` | mTLS/SASL fail | Alert ops |
| `AuthorizationException` | Topic ACL deny | Alert ops |

Banking pattern:

```java
future.whenComplete((result, ex) -> {
    if (ex instanceof TimeoutException) {
        outboxRepo.save(...);   // eventual delivery
    } else if (ex instanceof RecordTooLargeException) {
        dlqRepo.save(...);   // dev review
    } else if (ex != null) {
        log.error("Unexpected", ex);
        alerts.notifyOps(ex);
    } else {
        // success path
    }
});
```

### 13. Banking anti-pattern'leri

**Anti-pattern 1: `acks=0` veya `acks=1` banking event'inde**

Veri kaybı kabul edilemez. Hep `acks=all`.

**Anti-pattern 2: `enable.idempotence=false`**

Network glitch real. Duplicate event = duplicate notification = customer rahatsızlığı.

**Anti-pattern 3: Sync send hot path'te**

```java
@PostMapping("/transfers")
public Transfer transfer(@RequestBody TransferRequest req) {
    Transfer t = transferService.execute(req);
    kafkaTemplate.send("banking.transfers", t).get();   // ❌ blocks
    return t;
}
```

API latency Kafka publish latency'sine bağlanır. Async + outbox.

**Anti-pattern 4: Partition key = timestamp / UUID**

Ordering kaybolur.

**Anti-pattern 5: Producer per send**

```java
public void publish(Event e) {
    KafkaProducer p = new KafkaProducer(props);   // pahalı obje
    p.send(...);
    p.close();
}
```

Producer **singleton** (Spring bean). KafkaTemplate yeniden kullan.

**Anti-pattern 6: Outbox fallback yok**

Network down → Kafka publish fail → callback log'lu ama event kayboldu. Outbox eventual delivery garantisi.

**Anti-pattern 7: Schema evolution check yok**

JSON ile field rename → consumer kırılır deploy sonrası. Avro + Schema Registry'de backward compat check otomatik.

---

## Önemli olabilecek araştırma kaynakları

- Confluent producer config docs
- "Kafka: The Definitive Guide" (Narkhede, Shapira, Palino) — Chapter 3 (Producers)
- Apache Kafka KIP-98 (transactional producer)
- Confluent blog — exactly-once semantics deep dive
- Avro vs Protobuf vs JSON karşılaştırmaları
- Schema Registry documentation
- `librdkafka` (C client) docs — protocol detayları

---

## Mini task'ler

### Task 6.2.1 — Basic producer + acks=all (30 dk)

`TransferEventPublisher` yaz:

```java
@Component
public class TransferEventPublisher {
    private final KafkaTemplate<String, String> template;
    
    public CompletableFuture<SendResult<String, String>> publish(TransferCompleted event) {
        return template.send("banking.transfers", event.getFromAccountId().toString(), 
            new ObjectMapper().writeValueAsString(event));
    }
}
```

Test:
- `application.yml`'da `acks: all` + `enable.idempotence: true`
- Bir mesaj gönder, ack al
- `kafka-console-consumer` ile gör

### Task 6.2.2 — Async callback + Outbox fallback (45 dk)

`TransferEventPublisher`'a outbox fallback ekle:

```java
future.whenComplete((result, ex) -> {
    if (ex != null) {
        outboxRepo.save(...);
        meterRegistry.counter("publish.failure").increment();
    } else {
        meterRegistry.counter("publish.success").increment();
    }
});
```

Test: Kafka container'ı stop et, publish çağır → outbox tablo'sunda kayıt görmeli.

### Task 6.2.3 — Transactional producer (45 dk)

3 topic'e atomic publish:

```java
@Transactional("kafkaTransactionManager")
public void publishMultiple(Transfer t, AuditEvent a, NotificationEvent n) {
    template.send("banking.transfers", t.getId(), t);
    template.send("banking.audit", t.getId(), a);
    template.send("banking.notifications", n.recipient(), n);
}
```

Ortasında exception fırlat → tüm event'ler rollback olmalı. Consumer (read_committed) görmemeli.

### Task 6.2.4 — Partition key deneme (30 dk)

10 partition'lı topic. Aynı `accountId` ile 100 event gönder. Hepsi **aynı partition**'a gitmeli (consume ederek doğrula).

Sonra random UUID key ile 100 event → 10 partition'a yaklaşık eşit dağılım.

### Task 6.2.5 — Avro + Schema Registry (60 dk)

Schema Registry docker compose ekle. `TransferCompleted.avsc` yaz. Maven plugin ile Java class generate. Producer'ı Avro'ya çevir. Schema Registry UI'da schema'yı gör.

Sonra field ekle (`description: string`) → backward compat check OK olmalı (default value ile).

### Task 6.2.6 — Compression test (30 dk)

Aynı 10000 transfer event:
- `compression-type=none` → topic size ölç
- `compression-type=lz4` → topic size ölç
- `compression-type=gzip` → topic size ölç

Kafka tool: `kafka-log-dirs --describe` ile partition size.

**Defterine** tablo: compression ratio + CPU usage trade-off.

### Task 6.2.7 — Error handling matrix (45 dk)

Test her bir error case:
- Kafka stop → TimeoutException → outbox
- Çok büyük mesaj (>1MB) → RecordTooLargeException → DLQ
- Wrong serializer → SerializationException → DLQ

`@SpringBootTest` ile mock ProducerFactory + verify behavior.

---

## Test yazma rehberi

### Test 6.2.1 — Producer integration with TestContainers

```java
@SpringBootTest
@Testcontainers
@DirtiesContext
class TransferEventPublisherIT {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @DynamicPropertySource
    static void kafkaProps(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Autowired TransferEventPublisher publisher;
    @Autowired KafkaTemplate<String, String> template;
    
    @Test
    void shouldPublishWithCorrectPartitionKey() throws Exception {
        UUID accountId = UUID.randomUUID();
        TransferCompleted event = TransferCompleted.builder()
            .transferId(UUID.randomUUID())
            .fromAccountId(accountId)
            .toAccountId(UUID.randomUUID())
            .amount(new BigDecimal("100.00"))
            .currency("TRY")
            .build();
        
        SendResult<String, String> result = publisher.publish(event).get(5, TimeUnit.SECONDS);
        
        assertThat(result.getRecordMetadata().topic()).isEqualTo("banking.transfers");
        
        // Aynı partition key ile 10 daha gönder, hepsi aynı partition'a gitmeli
        Set<Integer> partitions = new HashSet<>();
        for (int i = 0; i < 10; i++) {
            event = event.toBuilder().transferId(UUID.randomUUID()).build();
            SendResult<String, String> r = publisher.publish(event).get(5, TimeUnit.SECONDS);
            partitions.add(r.getRecordMetadata().partition());
        }
        assertThat(partitions).hasSize(1);   // Hepsi aynı partition
    }
    
    @Test
    void shouldFallbackToOutboxWhenKafkaDown() throws Exception {
        kafka.stop();
        
        TransferCompleted event = createTestEvent();
        publisher.publish(event);
        
        await().atMost(10, TimeUnit.SECONDS).untilAsserted(() ->
            assertThat(outboxRepo.findByAggregateId(event.getTransferId())).isPresent()
        );
        
        kafka.start();
    }
}
```

### Test 6.2.2 — Transactional producer

```java
@Test
@Transactional("kafkaTransactionManager")
void transactionalSendShouldBeAtomic() {
    UUID transferId = UUID.randomUUID();
    
    try {
        publisher.publishMultiple(transfer, audit, notification);
        if (Math.random() < 0.5) throw new RuntimeException("Simulated failure");
    } catch (RuntimeException e) {
        // expected - transaction rolls back
    }
    
    // Consumer with isolation_level=read_committed should see all or nothing
    KafkaConsumer<String, String> consumer = ...;
    consumer.subscribe(List.of("banking.transfers", "banking.audit", "banking.notifications"));
    
    int found = 0;
    for (ConsumerRecord<String, String> record : consumer.poll(Duration.ofSeconds(5))) {
        if (record.key().equals(transferId.toString())) found++;
    }
    
    // Either 0 (rollback) or 3 (success), not 1 or 2
    assertThat(found).isIn(0, 3);
}
```

### Test 6.2.3 — Idempotence test

```java
@Test
void duplicateRetryShouldNotDuplicateInTopic() throws Exception {
    // Producer config'inde retries=10, idempotence=true
    
    UUID transferId = UUID.randomUUID();
    TransferCompleted event = createEvent(transferId);
    
    // Simulate network glitch — manuel retry pattern
    publisher.publish(event).get();
    // (Producer içinde retry happens otomatik)
    
    // Consume and count
    KafkaConsumer<String, String> consumer = ...;
    consumer.subscribe(List.of("banking.transfers"));
    
    int count = 0;
    for (ConsumerRecord<String, String> record : consumer.poll(Duration.ofSeconds(5))) {
        if (record.value().contains(transferId.toString())) count++;
    }
    
    assertThat(count).isEqualTo(1);   // not 2, idempotence working
}
```

---

## Claude-verify prompt

```
Kafka producer kodumu banking-grade kriterlere göre değerlendir. Eksiklerimi 
söyle, kod yazma:

1. Durability config:
   - acks=all aktif mi?
   - enable.idempotence=true mu?
   - min.insync.replicas=2 topic config'inde mi?
   - replication-factor=3 mi?

2. Async pattern:
   - send() callback ile mi yapılıyor (sync get() değil)?
   - Hot path bloklanıyor mu?
   - Hata callback'inde outbox fallback var mı?

3. Idempotence + ordering:
   - max.in.flight.requests.per.connection ≤ 5 mi?
   - idempotence aktif olduğu için retry duplicate yaratmıyor mu?
   - Partition key entity ID mi (UUID random DEĞİL)?

4. Transactional producer:
   - Multi-topic atomic publish için transactional.id var mı?
   - Spring @Transactional("kafkaTransactionManager") doğru kullanılmış mı?
   - Consumer isolation_level=read_committed mi?

5. Performance:
   - compression-type aktif mi (lz4/zstd)?
   - batch-size + linger-ms event volume'a göre seçilmiş mi?
   - buffer-memory yeterli mi?

6. Avro / Schema Registry:
   - Avro generated class kullanılıyor mu (JSON DEĞİL)?
   - Schema Registry entegre mi?
   - Schema evolution backward compatible mi?

7. Error handling:
   - TimeoutException → outbox fallback?
   - RecordTooLargeException → DLQ?
   - SerializationException → DLQ?
   - Authentication/Authorization → alert ops?

8. Banking-specific:
   - Sensitive data (PAN, TC No) event payload'da YOK mu?
   - Partition key entity ID (accountId, cardId, customerId)?
   - Outbox pattern dual-write için entegre mi?

9. Anti-pattern:
   - acks=0 veya acks=1 banking event'inde?
   - Sync .get() hot path'te?
   - Producer per send (singleton DEĞİL)?
   - JSON without schema registry?

10. Test:
    - TestContainers KafkaContainer integration?
    - Partition key test (aynı key → aynı partition)?
    - Idempotence test (duplicate retry → tek mesaj)?
    - Transactional rollback test?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `acks=all` + `enable.idempotence=true` + `min.insync.replicas=2`
- [ ] Async callback pattern + outbox fallback
- [ ] Partition key entity ID (accountId)
- [ ] Transactional producer ile multi-topic atomic publish denedim
- [ ] Compression aktif (lz4)
- [ ] Avro + Schema Registry setup (en az 1 topic için)
- [ ] Error handling matrix uygulanmış
- [ ] TestContainers ile integration test (4+ test)
- [ ] Idempotence test'le doğrulandı
- [ ] Compression ratio ölçüldü, defterimde tablo
- [ ] Banking PII payload'da YOK kontrolü

---

## Defter notları (10 madde)

1. "Producer'ın 4 aşamalı pipeline'ı (serializer → partitioner → accumulator → sender): ____."
2. "`acks=0` / `1` / `all` farkları + banking için seçim: ____."
3. "`enable.idempotence=true` ile broker (PID, sequence) dedup mekanizması: ____."
4. "`max.in.flight=5` + idempotence ordering garantisini neden bozmaz: ____."
5. "Partition key seçiminin per-key ordering'e etkisi (accountId örneği): ____."
6. "Transactional producer 3-topic atomic publish — banking örneği: ____."
7. "Async send + callback vs sync .get() banking hot path için: ____."
8. "Outbox fallback eventual delivery garantisi: ____."
9. "Avro + Schema Registry banking için 3 avantaj: ____."
10. "Compression (lz4) banking JSON payload için tipik ratio + CPU cost: ____."
