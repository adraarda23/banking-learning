# Topic 6.3 — Kafka Consumer Design

## Hedef

Kafka consumer'ı banking-grade seviyede konfigüre etmek. Consumer group dynamics, offset management strategies, rebalancing, exactly-once delivery, idempotent consumer pattern, isolation level. Banking notification/audit/fraud consumer'larını **veri kaybı + duplicate olmadan** çalıştırabilmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 6.1 + 6.2 bitti — Kafka architecture + producer biliyorsun
- Spring Kafka `@KafkaListener` basic seviyede gördün
- Phase 1'in transfer/account domain'i hazır

---

## Kavramlar

### 1. Consumer ne yapar

```
1. Subscribe topics → broker'a "ben bu topic'leri tüketeceğim"
2. Group coordinator partition assignment yapar
3. Consumer poll() ile mesaj çeker
4. Process → işle
5. Commit offset → "bu offset'e kadar işledim"
6. Repeat 3-5
```

`poll()` döngüsü kalbe atış. Heartbeat de poll çağrılarından gelir (eski Kafka), modern Kafka'da background thread.

### 2. Consumer group — partition paylaşımı

```yaml
spring:
  kafka:
    consumer:
      group-id: banking-notification-service
```

**Group ID** consumer'ları **takım** yapar. Aynı group içindeki consumer'lar partition'ları **paylaşır**.

```
Topic banking.transfers, 10 partitions
Group "notification-service", 3 instances:
  Instance 1: P0, P1, P2, P3
  Instance 2: P4, P5, P6
  Instance 3: P7, P8, P9
```

Her partition **sadece bir** consumer'a atanır. Load balancing otomatik.

**Banking örnek:** Aynı topic 3 farklı service tarafından consume edilir (notification, audit, fraud). Her biri **ayrı group**. Her event **3 farklı service'te** işlenir.

```
Group "notification-service" → SMS gönderir
Group "audit-service" → audit log yazar  
Group "fraud-service" → fraud score hesaplar
```

Her group bağımsız offset tutar. Group A'nın offset'i Group B'yi etkilemez.

### 3. Offset — consumer'ın yer imi

Her partition için her group'un kendi **offset**'i var. Bu bilgi Kafka'da **`__consumer_offsets`** internal topic'inde saklanır.

```
banking.transfers / Partition 0:
  Messages: [m1, m2, m3, m4, m5, m6, m7]
  Group "notification-service" committed offset: 4
  → Bu group m5'ten devam eder
```

Consumer **commit etmediği sürece** restart'ta tekrar consume eder.

### 4. Offset commit stratejileri

#### Auto commit — DİKKAT

```yaml
enable-auto-commit: true
auto-commit-interval-ms: 5000
```

Her 5 saniyede consumer son consume edilen offset'i otomatik commit.

**Tehlike:** Consumer crash anında commit interval'ı son 5 saniyelik mesajlar:
- Commit edilmiş ama henüz işlenmemiş → restart'ta **kayıp**
- İşlenmiş ama commit edilmemiş → restart'ta **duplicate**

**Banking için yetersiz.** Manuel commit.

#### Manual commit — synchronous

```yaml
enable-auto-commit: false
listener:
  ack-mode: MANUAL_IMMEDIATE
```

```java
@KafkaListener(topics = "banking.transfers", groupId = "notification-service")
public void consume(ConsumerRecord<String, TransferEvent> record, Acknowledgment ack) {
    try {
        TransferEvent event = record.value();
        notificationService.sendSms(event);
        
        ack.acknowledge();   // ✓ Sadece başarılı sonra commit
    } catch (Exception e) {
        log.error("Process failed", e);
        // commit YOK → restart'ta yeniden consume edilir (at-least-once)
        throw e;
    }
}
```

**At-least-once delivery:**
- Process başarılı + commit → her şey OK
- Process başarılı + commit fail → restart'ta yeniden process (idempotent değilse duplicate)
- Process fail + commit YOK → restart'ta yeniden process

#### Manual commit — async vs sync

```java
// Sync — slow ama güvenli
ack.acknowledge();   // = commitSync()

// Async — fast ama failure handling complex
// Spring Kafka built-in async commit AckMode.BATCH ile
```

Banking pratiği: `MANUAL_IMMEDIATE` (sync per record). Yavaş ama her event critical.

#### Batch ack

```yaml
listener:
  ack-mode: BATCH
```

Chunk işlemi sonrası tek commit. Daha hızlı ama partial failure recovery zor.

**Banking örneği:** High-volume non-critical event'ler (audit log) için OK. Critical (transfer notification) için per-record.

### 5. `auto.offset.reset` — yeni consumer ne yapsın

```yaml
auto-offset-reset: latest    # default
auto-offset-reset: earliest
auto-offset-reset: none
```

**Senaryo:** Yeni consumer group başlıyor. `__consumer_offsets`'te bu group için kayıt yok. Nereden başlasın?

- **`latest`:** Şu andan itibaren gelen mesajları al. Eski mesajları görme.
- **`earliest`:** Topic'in başından oku. Tüm tarihi tüket.
- **`none`:** Kayıt yoksa exception fırlat — DBA bilinçli karar.

**Banking pratiği:**
- **Notification consumer** → `latest` (eski notification gönderme)
- **Audit consumer** → `earliest` (her event kaydet)
- **Production systems** → `none` (operator karar)

**Tuzak:** Production'da `earliest` ile yeni group → milyonlarca eski mesaj process edilir. Cost + spam.

### 6. Consumer config — banking-grade

```yaml
spring:
  kafka:
    consumer:
      group-id: banking-notification-service
      bootstrap-servers: kafka-1:9092,kafka-2:9092,kafka-3:9092
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      enable-auto-commit: false
      auto-offset-reset: none
      max-poll-records: 100         # her poll'da max 100 mesaj
      max-poll-interval-ms: 300000  # 5 dk — bu süre içinde poll yoksa "dead"
      session-timeout-ms: 30000     # heartbeat timeout
      heartbeat-interval-ms: 10000  # heartbeat sıklığı (session/3)
      fetch-min-bytes: 1
      fetch-max-wait: 500
      isolation-level: read_committed   # transactional producer için
      properties:
        spring.json.trusted.packages: "com.mavibank.banking.events"
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
    listener:
      ack-mode: MANUAL_IMMEDIATE
      concurrency: 5                # 5 thread per @KafkaListener
```

#### Anahtar parametreler

**`max.poll.interval.ms`:** Poll'lar arası max süre. Bu sürede yeni poll çağrılmazsa group koordinatörü "consumer öldü" der → **rebalance**.

Banking impact: Listener method **uzun sürüyor**sa `max.poll.interval.ms`'ı artır veya işlemi **async**'e taşı.

**`session.timeout.ms`:** Heartbeat ile dead detection (background). Genelde `max.poll.interval.ms`'tan kısa.

**`heartbeat.interval.ms`:** Heartbeat sıklığı. `session.timeout.ms / 3` önerilir.

**`max.poll.records`:** Tek poll'da kaç mesaj. Yüksek = throughput, düşük = recovery granularity.

**`fetch.min.bytes` + `fetch.max.wait`:** Broker küçük partition'larda mesajları bekletip batch yapar (latency vs throughput).

### 7. Rebalancing — partition yeniden atama

Consumer add/remove/crash → partition'lar yeniden dağıtılır.

#### Eager rebalancing (default eski)

```
1. Group coordinator "stop the world" sinyali
2. Tüm consumer'lar partition'larını BIRAKIR
3. Coordinator reassign yapar
4. Hepsi yeni partition'larını alır
5. Processing tekrar başlar
```

**Banking impact:** STW duration boyunca **hiçbir consumer process yapmıyor**. Yüksek throughput sistemlerde lag patlar.

#### Cooperative rebalancing (Kafka 2.4+)

```yaml
partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

Sadece **gerekli** partition'lar reassign. Diğerleri uninterrupted continue. Incremental rebalance.

**Banking standard:** Cooperative her zaman tercih. STW azalır.

### 8. Idempotent consumer pattern — banking şart

Restart, rebalance, retry → consumer **aynı mesajı 2 kez** alabilir. Idempotency olmadan duplicate processing.

**Solution:** Processed event tablosu.

#### Schema

```sql
CREATE TABLE processed_events (
    event_id        UUID PRIMARY KEY,
    consumer_group  VARCHAR(100) NOT NULL,
    processed_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    -- compound unique olabilir: consumer_group bazlı
    UNIQUE (event_id, consumer_group)
);

CREATE INDEX idx_processed_cleanup ON processed_events(processed_at);
```

#### Consumer

```java
@Component
@Slf4j
public class TransferNotificationConsumer {
    
    private final ProcessedEventRepository processedRepo;
    private final NotificationService notificationService;
    
    @KafkaListener(
        topics = "banking.transfers",
        groupId = "notification-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    @Transactional
    public void consume(
        @Payload TransferEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack
    ) {
        log.debug("Received: id={}, partition={}, offset={}", 
            event.getId(), partition, offset);
        
        // Idempotency check
        UUID eventId = event.getId();
        if (processedRepo.existsByEventIdAndConsumerGroup(eventId, "notification-service")) {
            log.info("Duplicate event skipped: {}", eventId);
            ack.acknowledge();   // commit, skip
            return;
        }
        
        try {
            // Business logic
            notificationService.sendSms(event);
            
            // Idempotency record (same TX)
            processedRepo.save(new ProcessedEvent(eventId, "notification-service", Instant.now()));
            
            ack.acknowledge();
            log.debug("Processed: {}", eventId);
        } catch (Exception e) {
            log.error("Failed to process: {}", eventId, e);
            // NO commit → retry on next poll
            throw e;
        }
    }
}
```

**Garanti:** Per consumer-group, event_id **bir kez** işlenir.

**Cleanup:** Eski processed_events temizliği scheduled job (TTL 7-30 gün).

```java
@Scheduled(cron = "0 0 4 * * *")
public void cleanupOldProcessedEvents() {
    int deleted = processedRepo.deleteByProcessedAtBefore(Instant.now().minus(30, ChronoUnit.DAYS));
    log.info("Cleaned up {} old processed events", deleted);
}
```

### 9. Exactly-once delivery — 3 koşul

Kafka exactly-once için:

1. **Producer transactional** (Topic 6.2)
2. **Consumer `isolation.level=read_committed`**
3. **Idempotent consumer** (yukarıdaki pattern)

Üçü birden = **exactly-once end-to-end**. Banking için bu kombinasyon şart.

```yaml
spring:
  kafka:
    producer:
      transactional-id-prefix: tx-
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      isolation-level: read_committed
```

### 10. Error handling — DefaultErrorHandler + DLT

İşlenemeyen mesaj sonsuza retry → consumer takılır. **Dead Letter Topic** pattern:

```java
@Configuration
public class KafkaErrorHandlerConfig {
    
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));
        
        DefaultErrorHandler handler = new DefaultErrorHandler(
            recoverer,
            new ExponentialBackOffWithMaxRetries(3)   // 3 retry: 1s, 2s, 4s
        );
        
        // Bu exception'lar **retry'lanmaz** — direkt DLT
        handler.addNotRetryableExceptions(
            InvalidEventException.class,
            IllegalArgumentException.class,
            JsonProcessingException.class
        );
        
        // Bu exception'lar retry'lanır
        handler.addRetryableExceptions(
            TransientDataAccessException.class,
            ResourceAccessException.class
        );
        
        return handler;
    }
}
```

Listener factory'e ekle:

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
        ConsumerFactory<String, Object> consumerFactory,
        DefaultErrorHandler errorHandler) {
    
    ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setConcurrency(5);
    factory.setCommonErrorHandler(errorHandler);
    factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
    return factory;
}
```

DLT consumer (operator review):

```java
@KafkaListener(topics = "banking.transfers.DLT", groupId = "dlt-monitor")
public void onDlt(ConsumerRecord<String, String> record) {
    log.error("Message in DLT: topic={}, partition={}, offset={}, value={}",
        record.topic(), record.partition(), record.offset(), record.value());
    
    dlqRepo.save(new DeadLetterRecord(
        record.topic(),
        record.partition(),
        record.offset(),
        record.value(),
        Instant.now()
    ));
    
    // Alert ops
    notifier.alertOps("DLT message: " + record.topic());
}
```

### 11. Listener container concurrency

```yaml
listener:
  concurrency: 5
```

Tek `@KafkaListener` için 5 consumer thread. Her thread bir consumer instance.

**Kural:** `concurrency <= partition count`. Daha fazla thread = boşta consumer.

**Banking örnek:** Topic 10 partition × concurrency 5 × instance 2 = 10 paralel consumer. Her partition bir consumer'a.

### 12. Banking pattern — full notification consumer

```java
@Component
@Slf4j
public class TransferNotificationConsumer {
    
    private final ProcessedEventRepository processedRepo;
    private final NotificationService notificationService;
    private final MeterRegistry registry;
    
    @KafkaListener(
        topics = "banking.transfers",
        groupId = "notification-service",
        containerFactory = "transactionalKafkaListenerContainerFactory"
    )
    @Transactional
    public void consume(
        @Payload TransferEvent event,
        @Header(value = "X-Trace-Id", required = false) String traceId,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack
    ) {
        Timer.Sample sample = Timer.start(registry);
        MDC.put("traceId", traceId != null ? traceId : "no-trace");
        MDC.put("eventId", event.getId().toString());
        
        try {
            log.debug("Received transfer event: partition={}, offset={}", partition, offset);
            
            // Idempotency check
            if (processedRepo.existsByEventIdAndConsumerGroup(event.getId(), "notification-service")) {
                log.warn("Duplicate event skipped: {}", event.getId());
                registry.counter("consumer.duplicate.skipped", "topic", "banking.transfers").increment();
                ack.acknowledge();
                return;
            }
            
            // Process
            notificationService.sendTransferNotification(event);
            
            // Record
            processedRepo.save(new ProcessedEvent(
                event.getId(), 
                "notification-service", 
                Instant.now()
            ));
            
            ack.acknowledge();
            registry.counter("consumer.success", "topic", "banking.transfers").increment();
            
        } catch (RetryableException e) {
            log.warn("Retryable error: {}", e.getMessage());
            registry.counter("consumer.retry", "topic", "banking.transfers").increment();
            throw e;   // DefaultErrorHandler retry yapacak
        } catch (InvalidEventException e) {
            log.error("Invalid event, sending to DLT: {}", event.getId(), e);
            registry.counter("consumer.dlt", "topic", "banking.transfers").increment();
            throw e;   // Direkt DLT (NotRetryableException listesinde)
        } finally {
            sample.stop(registry.timer("consumer.processing.duration", "topic", "banking.transfers"));
            MDC.clear();
        }
    }
}
```

Production-grade: idempotency + tracing + metrics + retry/DLT pattern.

### 13. Header propagation — distributed tracing

Producer'da set edilen header'lar (X-Trace-Id, X-User-Id) consumer'da okunmalı.

```java
@KafkaListener(...)
public void consume(
    @Payload TransferEvent event,
    @Header("X-Trace-Id") String traceId,
    @Header(value = "X-Tenant-Id", required = false) String tenantId
) {
    MDC.put("traceId", traceId);
    try {
        // ...
    } finally {
        MDC.clear();
    }
}
```

Banking pratiği: TraceId her event'te zorunlu. Logs aggregation'da request correlation.

### 14. Consumer lifecycle — pause/resume

Bakım pencerelerinde consumer'ı durdur:

```java
@Autowired
private KafkaListenerEndpointRegistry registry;

public void pauseNotificationConsumer() {
    registry.getListenerContainer("transferListener").pause();
}

public void resumeNotificationConsumer() {
    registry.getListenerContainer("transferListener").resume();
}
```

Pause sırasında poll devam eder ama listener trigger olmaz. Backpressure mekanizması.

### 15. Banking anti-pattern'leri

**Anti-pattern 1: `enable-auto-commit: true` banking event'inde**

Commit timing belirsiz. Veri kaybı veya duplicate.

**Anti-pattern 2: Idempotency check yok**

Retry, rebalance → duplicate SMS, duplicate audit, duplicate notification.

**Anti-pattern 3: Listener method'da uzun iş**

```java
@KafkaListener(topics = "banking.transfers")
public void consume(TransferEvent event) {
    smsGateway.send(event);   // 10 sn external call
    callBack.notify(event);   // 10 sn external call
    auditLog.write(event);    // DB call
    // Total: 25 sn — max.poll.interval.ms aşılır
}
```

**Fix:** Listener'da event'i async executor'a at, commit'i schedule et. Veya `max.poll.interval.ms`'ı artır (riskli).

**Anti-pattern 4: DLT yok**

İşlenemeyen mesaj sonsuza retry. Consumer takılır.

**Anti-pattern 5: Cross-partition ordering varsayımı**

Partition'lar arası ordering YOK. "Account A'nın event'i Account B'nin event'inden önce" varsayma.

**Anti-pattern 6: SerializationException sonsuz retry**

Poison message — bozuk JSON. `addNotRetryableExceptions(SerializationException.class)` veya `ErrorHandlingDeserializer`.

```yaml
spring:
  kafka:
    consumer:
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
```

Deserialization fail → null event listener'a gelir → handler DLT'ye atar.

**Anti-pattern 7: `concurrency > partition count`**

Boşta consumer thread'leri. Waste.

---

## Önemli olabilecek araştırma kaynakları

- "Kafka: The Definitive Guide" — Chapter 4 (Consumers)
- Confluent consumer documentation
- Spring Kafka reference (DefaultErrorHandler, DLT, retryable topics)
- KIP-429 (incremental cooperative rebalancing)
- "Exactly-once Semantics in Apache Kafka" Confluent blog
- Idempotent consumer pattern (Microservices Patterns book)

---

## Mini task'ler

### Task 6.3.1 — Basic consumer + manual commit (30 dk)

```java
@KafkaListener(topics = "banking.transfers", groupId = "notification-service")
public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try {
        log.info("Received: {}", record.value());
        ack.acknowledge();
    } catch (Exception e) {
        log.error("Failed", e);
    }
}
```

Test:
- Auto commit kapalı
- Listener exception fırlat → restart sonrası mesajı yeniden al
- Listener başarılı → ack edildi, restart'ta yeniden alma

### Task 6.3.2 — Idempotent consumer pattern (45 dk)

`processed_events` migration + `ProcessedEventRepository` + idempotency check.

Test: 100 event gönder, listener'ı kasten 2 kez fail ettir. processed_events tablosunda 100 kayıt olmalı (duplicate yok).

### Task 6.3.3 — DefaultErrorHandler + DLT (45 dk)

3 retry + DLT pattern. `addNotRetryableExceptions(InvalidEventException.class)`.

Test:
- Transient error → 3 retry, sonra DLT
- Invalid event → direkt DLT (retry yok)
- DLT topic'i consume eden monitoring listener

### Task 6.3.4 — Exactly-once setup (60 dk)

Producer transactional + consumer read_committed + idempotent consumer.

Test:
- Producer transaction commit → consumer görür
- Producer transaction abort → consumer **görmez**
- Aynı mesaj 2 kez gelirse → tek kez işlenir

### Task 6.3.5 — Header propagation (30 dk)

Producer X-Trace-Id ekler. Consumer @Header ile alır, MDC'ye koyar. Log'larda traceId görünür.

### Task 6.3.6 — Concurrency + rebalancing (45 dk)

10 partition topic, concurrency=5, instance=1. Log'da hangi thread hangi partition'a atanmış gör.

İkinci instance başlat → cooperative rebalancing. Cluster log'da partition reassignment görünür.

### Task 6.3.7 — ErrorHandlingDeserializer (poison message) (30 dk)

Producer bozuk JSON gönder. Default deserializer SerializationException atar → consumer takılır.

`ErrorHandlingDeserializer` ile delegate. Bozuk mesaj → null → DLT.

---

## Test yazma rehberi

### Test 6.3.1 — Consumer integration with TestContainers

```java
@SpringBootTest
@Testcontainers
@DirtiesContext
class TransferNotificationConsumerIT {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void kafkaProps(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Autowired KafkaTemplate<String, TransferEvent> template;
    @MockBean NotificationService notificationService;
    @Autowired ProcessedEventRepository processedRepo;
    
    @Test
    void shouldProcessEventAndStoreIdempotencyRecord() throws Exception {
        TransferEvent event = new TransferEvent(...);
        template.send("banking.transfers", event.getId().toString(), event).get();
        
        await().atMost(10, TimeUnit.SECONDS).untilAsserted(() -> {
            assertThat(processedRepo.existsByEventIdAndConsumerGroup(
                event.getId(), "notification-service")).isTrue();
        });
        
        verify(notificationService, times(1)).sendTransferNotification(event);
    }
    
    @Test
    void shouldNotProcessDuplicateEvent() throws Exception {
        TransferEvent event = new TransferEvent(...);
        
        // First send
        template.send("banking.transfers", event.getId().toString(), event).get();
        await().atMost(10, SECONDS).untilAsserted(() -> 
            verify(notificationService).sendTransferNotification(event));
        
        // Second send — same event ID
        template.send("banking.transfers", event.getId().toString(), event).get();
        
        Thread.sleep(2000);
        
        // Still only 1 call (idempotency working)
        verify(notificationService, times(1)).sendTransferNotification(event);
    }
    
    @Test
    void shouldRetryAndSendToDlt() throws Exception {
        TransferEvent event = ...;
        doThrow(new TransientDataAccessException("DB timeout"))
            .when(notificationService).sendTransferNotification(any());
        
        template.send("banking.transfers", event.getId().toString(), event).get();
        
        // 3 retry
        await().atMost(15, SECONDS).untilAsserted(() ->
            verify(notificationService, times(3)).sendTransferNotification(any())
        );
        
        // DLT'de mesaj var mı?
        KafkaConsumer<String, String> dltConsumer = createTestConsumer("banking.transfers.DLT");
        ConsumerRecords<String, String> records = dltConsumer.poll(Duration.ofSeconds(5));
        assertThat(records).isNotEmpty();
    }
}
```

### Test 6.3.2 — Exactly-once integration

```java
@Test
void exactlyOnceShouldWork() throws Exception {
    UUID transferId = UUID.randomUUID();
    
    // Transactional producer
    transactionalPublisher.publishMultipleAtomic(transferId);
    
    // Consumer (read_committed) tüm event'leri görür
    await().atMost(10, SECONDS).untilAsserted(() -> {
        assertThat(processedRepo.existsByEventId(transferId)).isTrue();
    });
    
    verify(notificationService, times(1)).sendTransferNotification(any());
}

@Test
void abortedTransactionShouldNotBeSeen() throws Exception {
    UUID transferId = UUID.randomUUID();
    
    // Transactional producer with rollback
    try {
        transactionalPublisher.publishMultipleWithFailure(transferId);
    } catch (Exception e) {
        // expected
    }
    
    Thread.sleep(3000);
    
    // Consumer NOT görmemeli
    assertThat(processedRepo.existsByEventId(transferId)).isFalse();
    verify(notificationService, never()).sendTransferNotification(any());
}
```

### Test 6.3.3 — Header propagation

```java
@Test
void shouldPropagateTraceId() throws Exception {
    UUID transferId = UUID.randomUUID();
    String traceId = "trace-abc-123";
    
    ProducerRecord<String, TransferEvent> record = new ProducerRecord<>(
        "banking.transfers", transferId.toString(), createEvent(transferId));
    record.headers().add("X-Trace-Id", traceId.getBytes(StandardCharsets.UTF_8));
    
    template.send(record).get();
    
    await().atMost(5, SECONDS).until(() -> consumerInvoked.get());
    
    // Verify MDC was set during processing
    assertThat(capturedTraceIds).contains(traceId);
}
```

---

## Claude-verify prompt

```
Kafka consumer kodumu banking-grade kriterlere göre değerlendir:

1. Offset management:
   - enable-auto-commit=false mu?
   - ack-mode MANUAL_IMMEDIATE mi (per-record commit)?
   - Manuel commit hatasında commit YAPILMIYOR mu (rethrow)?

2. Idempotency:
   - processed_events tablosu var mı?
   - Consumer'da check before process pattern uygulanmış mı?
   - @Transactional ile process + record same TX mi?
   - Cleanup scheduled job var mı?

3. Error handling:
   - DefaultErrorHandler + DeadLetterPublishingRecoverer setup'lı mı?
   - 3-5 retry exponential backoff?
   - addNotRetryableExceptions specific class'larla?
   - DLT consumer ile alert?
   - ErrorHandlingDeserializer poison message için?

4. Exactly-once:
   - Producer transactional + consumer read_committed + idempotent consumer üçü birden var mı?
   - isolation-level: read_committed config'de mi?

5. Rebalancing:
   - CooperativeStickyAssignor kullanılmış mı (eager DEĞİL)?
   - max.poll.interval.ms processing süresinden büyük mü?
   - session.timeout.ms / heartbeat.interval.ms doğru orantıda mı?

6. Concurrency:
   - concurrency partition count'tan küçük veya eşit mi?
   - 10 partition + concurrency=5 yaratmış mı 5 thread?

7. Header propagation:
   - X-Trace-Id consumer'da okunup MDC'ye konuyor mu?
   - Banking distributed tracing entegre mi?

8. Banking patterns:
   - 3 ayrı service için 3 ayrı group-id (notification, audit, fraud)?
   - PII consumer log'larında görünüyor mu? (Olmamalı)
   - Metrics (success/failure/duplicate/dlt) sayılıyor mu?

9. Anti-pattern:
   - auto-commit=true banking event'inde?
   - Listener method'da uzun iş (max.poll.interval aşılır)?
   - DLT yok?
   - SerializationException sonsuz retry?
   - concurrency > partition count?

10. Test:
    - TestContainers KafkaContainer + PostgreSQLContainer?
    - Idempotency test (same event 2 kez → 1 process)?
    - Retry → DLT test?
    - Transactional rollback → consumer not see test?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `enable-auto-commit=false` + `MANUAL_IMMEDIATE` ack
- [ ] `processed_events` tablosu + idempotency check
- [ ] DefaultErrorHandler + DLT setup
- [ ] Exactly-once (producer transactional + consumer read_committed + idempotent)
- [ ] CooperativeStickyAssignor
- [ ] `concurrency` parameter ile multi-thread (partition count ≤)
- [ ] Header propagation (X-Trace-Id → MDC)
- [ ] ErrorHandlingDeserializer poison message için
- [ ] TestContainers ile 4+ integration test
- [ ] Idempotency canlı test edildi (duplicate event → 1 process)
- [ ] DLT consumer + alert
- [ ] processed_events cleanup scheduled job

---

## Defter notları (10 madde)

1. "Consumer group + partition assignment mekanizması (`__consumer_offsets` topic): ____."
2. "Auto-commit vs manual commit banking için karar: ____."
3. "At-least-once delivery + idempotent consumer → effective exactly-once: ____."
4. "Exactly-once için 3 koşul (producer tx + read_committed + idempotent): ____."
5. "Cooperative vs eager rebalancing STW impact: ____."
6. "max.poll.interval.ms processing süresine göre ayar: ____."
7. "DefaultErrorHandler retry + DLT pattern banking: ____."
8. "ErrorHandlingDeserializer poison message için: ____."
9. "processed_events tablosu + cleanup retention: ____."
10. "Header propagation (X-Trace-Id) distributed tracing rolü: ____."
