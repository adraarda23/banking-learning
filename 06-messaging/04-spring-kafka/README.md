# Topic 6.4 — Spring Kafka Integration Deep

## Hedef

Spring Kafka abstraction'ını derinlemesine öğrenmek. `@KafkaListener`, `ConcurrentKafkaListenerContainerFactory`, error handler, `@RetryableTopic`, batch listener, transactional listener, lifecycle control. Banking event-driven architecture'da production-grade Spring Kafka setup'ı.

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~4.5 saat

## Önbilgi

- Topic 6.1-6.3 bitti (Kafka arch, producer, consumer)
- Spring Boot + KafkaTemplate aşinasın
- @KafkaListener basic seen

---

## Kavramlar

### 1. Spring Kafka katmanları

```
Application code
    ↓
@KafkaListener / KafkaTemplate (Spring abstraction)
    ↓
ConcurrentKafkaListenerContainer / ProducerFactory
    ↓
Apache Kafka client (KafkaConsumer/Producer)
    ↓
Kafka brokers
```

Spring Kafka:
- Lifecycle management (start/stop containers)
- Spring transaction integration
- Error handling + retry policies
- Listener annotation-based programming
- Container concurrency
- DLT publishing

### 2. Configuration deep

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory(KafkaProperties properties) {
        Map<String, Object> props = properties.buildProducerProperties();
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> pf) {
        return new KafkaTemplate<>(pf);
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory(KafkaProperties properties) {
        Map<String, Object> props = properties.buildConsumerProperties();
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> cf,
            DefaultErrorHandler errorHandler) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(cf);
        factory.setConcurrency(5);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        factory.setCommonErrorHandler(errorHandler);
        factory.getContainerProperties().setMicrometerEnabled(true);
        return factory;
    }
}
```

### 3. `@KafkaListener` annotation

```java
@Component
public class TransferConsumer {
    
    @KafkaListener(
        id = "transferListener",                                 // bean name for runtime control
        topics = "banking.transfers",
        groupId = "notification-service",
        containerFactory = "kafkaListenerContainerFactory",
        concurrency = "5",                                       // 5 thread per @KafkaListener
        autoStartup = "${kafka.consumer.autostartup:true}"       // env-based start control
    )
    public void consume(
        @Payload TransferEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long ts,
        @Header(value = "X-Trace-Id", required = false) String traceId,
        Acknowledgment ack
    ) {
        // process
        ack.acknowledge();
    }
}
```

### 4. Multi-topic listener

```java
@KafkaListener(
    topics = {"banking.transfers", "banking.payments", "banking.refunds"},
    groupId = "audit-service"
)
public void auditMultiple(@Payload Object event, @Header("X-Source") String source) {
    auditService.log(source, event);
}
```

Veya pattern:

```java
@KafkaListener(topicPattern = "banking\\..*", groupId = "global-audit")
public void wildcardConsumer(@Payload Object event) {
    // tüm banking topic'leri tüket
}
```

**Tehlike:** Pattern eklenen yeni topic'leri otomatik consume eder. Production'da explicit topic list tercih.

### 5. Partition-specific listener

```java
@KafkaListener(
    groupId = "vip-customer-service",
    topicPartitions = @TopicPartition(
        topic = "banking.transfers",
        partitions = {"0", "1", "2"}   // sadece bu partition'lar
    )
)
public void vipTransfers(@Payload TransferEvent event) {
    // sadece partition 0-2'deki event'ler (VIP müşteriler)
}
```

Banking pattern: Custom partitioner ile VIP customers → partition 0-2, normal → 3-9. Specialized consumer.

### 6. Batch listener

```java
@KafkaListener(
    topics = "banking.audit",
    groupId = "audit-service",
    containerFactory = "batchListenerContainerFactory"
)
public void batchConsume(
    List<AuditEvent> events,
    @Header(KafkaHeaders.RECEIVED_PARTITION) List<Integer> partitions,
    @Header(KafkaHeaders.OFFSET) List<Long> offsets,
    Acknowledgment ack
) {
    // Tek bir batch'te 100 event işle
    auditRepository.saveAll(events);
    ack.acknowledge();
}
```

Container factory:

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> batchListenerContainerFactory(
        ConsumerFactory<String, Object> cf) {
    ConcurrentKafkaListenerContainerFactory<String, Object> factory = ...;
    factory.setBatchListener(true);
    factory.setConcurrency(3);
    return factory;
}
```

Banking örnek: High-volume audit event'leri (100 event/poll) batch process. Throughput katlanır, DB roundtrip azalır.

### 7. Reply (request/response over Kafka)

```java
@KafkaListener(topics = "fraud.check.request", groupId = "fraud-service")
@SendTo("fraud.check.response")
public FraudResult check(@Payload FraudCheckRequest req) {
    return fraudService.calculate(req);
}
```

Consumer process → return value otomatik response topic'e gönder.

Client side:

```java
@Autowired ReplyingKafkaTemplate<String, FraudCheckRequest, FraudResult> template;

public FraudResult checkFraud(FraudCheckRequest req) throws Exception {
    ProducerRecord<String, FraudCheckRequest> record = 
        new ProducerRecord<>("fraud.check.request", req);
    record.headers().add(KafkaHeaders.REPLY_TOPIC, "fraud.check.response".getBytes());
    
    RequestReplyFuture<String, FraudCheckRequest, FraudResult> future = 
        template.sendAndReceive(record);
    
    return future.get(5, TimeUnit.SECONDS).value();
}
```

**Banking anti-pattern uyarısı:** Sync request/response Kafka'da overhead. **HTTP/gRPC daha doğru.** Use sparingly.

### 8. DefaultErrorHandler — deep

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> {
            // Custom DLT routing
            if (ex.getCause() instanceof JsonProcessingException) {
                return new TopicPartition("banking.poison", -1);   // poison mesajlar ayrı
            }
            return new TopicPartition(record.topic() + ".DLT", record.partition());
        });
    
    // Exponential backoff: 1s, 2s, 4s, 8s, max 5 retry
    ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(5);
    backOff.setInitialInterval(1000L);
    backOff.setMultiplier(2.0);
    backOff.setMaxInterval(30000L);
    
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
    
    // Retry yapılmayacak exception'lar
    handler.addNotRetryableExceptions(
        IllegalArgumentException.class,
        JsonProcessingException.class,
        InvalidEventException.class,
        DeserializationException.class
    );
    
    // Retry yapılacak exception'lar (explicit)
    handler.addRetryableExceptions(
        TransientDataAccessException.class,
        ResourceAccessException.class,
        TimeoutException.class
    );
    
    // Retry öncesi callback
    handler.setRetryListeners((record, ex, deliveryAttempt) -> {
        log.warn("Retry attempt {} for record at offset {}", deliveryAttempt, record.offset());
        meterRegistry.counter("kafka.retry", "topic", record.topic()).increment();
    });
    
    return handler;
}
```

### 9. `@RetryableTopic` — non-blocking retry

Default retry **blocking** — listener thread tied up retry duration boyunca. Yüksek latency = consumer lag.

`@RetryableTopic` non-blocking pattern:

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 30000),
    autoCreateTopics = "true",
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
    dltStrategy = DltStrategy.FAIL_ON_ERROR,
    exclude = {IllegalArgumentException.class, JsonProcessingException.class}
)
@KafkaListener(topics = "banking.transfers", groupId = "notification-service")
public void consume(@Payload TransferEvent event) {
    notificationService.sendSms(event);
}

@DltHandler
public void handleDlt(
    @Payload TransferEvent event, 
    @Header(KafkaHeaders.ORIGINAL_TOPIC) String topic,
    @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exMsg
) {
    log.error("DLT: topic={}, event={}, error={}", topic, event, exMsg);
    dlqRepo.save(new DeadLetterRecord(event, topic, exMsg, Instant.now()));
}
```

Spring otomatik retry topic'leri yaratır:

```
banking.transfers              (original)
banking.transfers-retry-0      (1s sonra retry)
banking.transfers-retry-1      (2s sonra retry)
banking.transfers-retry-2      (4s sonra retry)
banking.transfers-retry-3      (8s sonra retry)
banking.transfers-dlt          (final failure)
```

**Avantaj:** Main consumer beklemez. Retry message ayrı topic'te, ayrı consumer, ayrı thread.

**Banking örnek:** External SMS gateway 30 sn timeout. Sync retry main consumer'ı bloklar. Non-blocking retry topic'leriyle main partition consume devam eder.

### 10. Transactional listener

```java
@KafkaListener(topics = "banking.transfers", groupId = "transfer-orchestrator")
@Transactional("kafkaTransactionManager")   // Kafka tx
public void consume(@Payload TransferEvent event) {
    // İşle
    Transfer t = processTransfer(event);
    
    // Yeni event'ler publish — aynı transaction
    kafkaTemplate.send("banking.transfers.completed", t);
    kafkaTemplate.send("banking.audit", t.toAudit());
    
    // Read-process-write transaction
    // Eğer exception fırlarsa: input offset commit OLMAZ + output publish ROLLBACK
}
```

**Transactional consumer-producer** = read-process-write atomicity. Exactly-once stream processing.

Kafka Streams (Topic 6.5) bu pattern üzerine kuruludur.

### 11. Header propagation

```java
@KafkaListener(topics = "banking.transfers")
public void consume(
    @Payload TransferEvent event,
    @Header(value = "X-Trace-Id", required = false) String traceId,
    @Header(value = "X-Tenant-Id", required = false) String tenant,
    @Header(value = "X-User-Id", required = false) String userId
) {
    MDC.put("traceId", traceId != null ? traceId : "no-trace");
    MDC.put("tenant", tenant);
    MDC.put("userId", userId);
    
    try {
        process(event);
    } finally {
        MDC.clear();
    }
}
```

Banking distributed tracing — Phase 1 traceId + Phase 9 OpenTelemetry entegrasyonu.

### 12. Listener lifecycle control

```java
@Autowired KafkaListenerEndpointRegistry registry;

// Pause consumer (poll continues, no listener invocation)
public void pauseTransferListener() {
    registry.getListenerContainer("transferListener").pause();
}

public void resumeTransferListener() {
    registry.getListenerContainer("transferListener").resume();
}

// Stop completely
public void stopTransferListener() {
    registry.getListenerContainer("transferListener").stop();
}
```

**Banking örnek — bakım penceresi:**

```java
@Component
public class MaintenanceWindowManager {
    
    @Autowired KafkaListenerEndpointRegistry registry;
    
    @Scheduled(cron = "0 0 3 * * *")   // 03:00
    public void enterMaintenance() {
        log.info("Entering maintenance window — pausing consumers");
        registry.getAllListenerContainers().forEach(c -> c.pause());
    }
    
    @Scheduled(cron = "0 0 4 * * *")   // 04:00
    public void exitMaintenance() {
        log.info("Exiting maintenance window — resuming consumers");
        registry.getAllListenerContainers().forEach(c -> c.resume());
    }
}
```

### 13. ContainerCustomizer

Container behavior'ını ince ayarla:

```java
@Bean
public ContainerCustomizer<String, Object, ConcurrentMessageListenerContainer<String, Object>> 
        containerCustomizer() {
    return container -> {
        ContainerProperties props = container.getContainerProperties();
        props.setIdleEventInterval(30000L);   // idle event her 30 sn
        props.setMicrometerEnabled(true);
        props.setObservationEnabled(true);    // OpenTelemetry tracing
    };
}
```

### 14. Banking pattern — full setup

```java
@Configuration
@EnableKafka
public class BankingKafkaConfig {
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> pf) {
        KafkaTemplate<String, Object> template = new KafkaTemplate<>(pf);
        template.setObservationEnabled(true);   // Micrometer Tracing
        return template;
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> cf,
            DefaultErrorHandler errorHandler) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(cf);
        factory.setConcurrency(5);
        factory.setCommonErrorHandler(errorHandler);
        
        ContainerProperties cp = factory.getContainerProperties();
        cp.setAckMode(AckMode.MANUAL_IMMEDIATE);
        cp.setObservationEnabled(true);
        cp.setMicrometerEnabled(true);
        cp.setLogContainerConfig(true);
        
        return factory;
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> batchListenerContainerFactory(
            ConsumerFactory<String, Object> cf) {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(cf);
        factory.setBatchListener(true);
        factory.setConcurrency(3);
        return factory;
    }
}

@Component
@Slf4j
public class TransferNotificationConsumer {
    
    private final NotificationService notificationService;
    private final ProcessedEventRepository processedRepo;
    
    @KafkaListener(
        id = "transferListener",
        topics = "banking.transfers",
        groupId = "notification-service"
    )
    @Transactional
    public void consume(
        @Payload TransferEvent event,
        @Header("X-Trace-Id") String traceId,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack
    ) {
        MDC.put("traceId", traceId);
        try {
            if (processedRepo.existsByEventIdAndConsumerGroup(event.getId(), "notification-service")) {
                ack.acknowledge();
                return;
            }
            
            notificationService.sendSms(event);
            processedRepo.save(new ProcessedEvent(event.getId(), "notification-service"));
            
            ack.acknowledge();
        } finally {
            MDC.clear();
        }
    }
}
```

### 15. Banking anti-pattern'leri

**Anti-pattern 1: ContainerFactory default config'le production'da**

Default ack mode (BATCH), auto-commit true → veri kaybı/duplicate. Banking için **explicit config**.

**Anti-pattern 2: @KafkaListener method exception swallow**

```java
@KafkaListener(...)
public void consume(TransferEvent event) {
    try {
        process(event);
    } catch (Exception e) {
        log.error("error", e);   // swallow → DefaultErrorHandler kapsamı dışında, DLT YOK
    }
}
```

Exception rethrow et — error handler retry/DLT yapsın.

**Anti-pattern 3: `@RetryableTopic` ile blocking retry karıştırmak**

İkisi birden olur ama overlapping retry kaotik. Bir yöntemi seç.

**Anti-pattern 4: Listener içinde uzun-running async başlatmak**

```java
@KafkaListener
public void consume(TransferEvent event) {
    CompletableFuture.runAsync(() -> longTask(event));
    ack.acknowledge();   // ❌ commit before completion
}
```

Future fail olursa kaybolur. Process bittikten sonra commit.

**Anti-pattern 5: `ReplyingKafkaTemplate` (sync) HTTP yerine**

Sync request/response Kafka'da overhead + ordering issue. HTTP/gRPC kullan.

**Anti-pattern 6: Tüm topic'lere tek listener**

```java
@KafkaListener(topicPattern = ".*")   // ❌ her şey
```

Concurrency yönetimi karışır. Topic group'ları için ayrı listener.

---

## Önemli olabilecek araştırma kaynakları

- Spring Kafka reference (current version 3.x)
- "Apache Kafka with Spring Boot 3" tutorial serisi
- Spring Kafka @RetryableTopic deep dive blog
- ContainerProperties JavaDoc
- DefaultErrorHandler customization examples
- Micrometer Tracing + Spring Kafka observation

---

## Mini task'ler

### Task 6.4.1 — Full container factory setup (30 dk)

`KafkaConfig` class'ı yaz:
- ProducerFactory (idempotence, acks=all)
- ConsumerFactory (manual commit, read_committed)
- ContainerFactory (concurrency 5, MANUAL_IMMEDIATE, error handler)
- BatchContainerFactory (concurrency 3, batch=true)
- MicrometerEnabled + observation

### Task 6.4.2 — Batch listener (45 dk)

Audit event'lerini batch consume + bulk DB insert:

```java
@KafkaListener(
    topics = "banking.audit",
    groupId = "audit-service",
    containerFactory = "batchListenerContainerFactory"
)
public void batchConsume(List<AuditEvent> events, Acknowledgment ack) {
    auditRepo.saveAll(events);   // bulk insert
    ack.acknowledge();
}
```

Test: 1000 event hızlıca gönder, batch listener 10x'lik chunk'larla işliyor mu?

### Task 6.4.3 — @RetryableTopic non-blocking retry (45 dk)

`@RetryableTopic` ile 4 attempt + exponential backoff + DLT handler.

Test:
- Transient error → retry topic'lerinde mesaj görünür
- 4 fail → DLT handler tetiklenir
- Main consumer **bloklanmamış** (kontrol et)

### Task 6.4.4 — Transactional listener (45 dk)

Read-process-write pattern:

```java
@KafkaListener(...)
@Transactional("kafkaTransactionManager")
public void consume(TransferEvent event) {
    process(event);
    kafkaTemplate.send("output.topic", processedEvent);
    // input offset + output publish atomic
}
```

Test:
- Mid-process exception → output topic'te mesaj YOK, input offset commit YOK
- Successful run → output görünür, offset commit

### Task 6.4.5 — Lifecycle control (30 dk)

`MaintenanceWindowManager` ile scheduled pause/resume. Endpoint ekle: `POST /admin/consumers/pause`, `/resume`.

### Task 6.4.6 — Header propagation (30 dk)

X-Trace-Id, X-Tenant-Id, X-User-Id headers propagation. MDC entegrasyonu.

### Task 6.4.7 — Custom DLT routing (30 dk)

DefaultErrorHandler'da exception tipine göre farklı DLT:
- JsonProcessingException → `banking.poison`
- TransientException → `banking.transfers.DLT`

---

## Test yazma rehberi

### Test 6.4.1 — @KafkaListener integration

```java
@SpringBootTest
@Testcontainers
class KafkaListenerIT {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(...);
    
    @Autowired KafkaTemplate<String, TransferEvent> template;
    @MockBean NotificationService notificationService;
    
    @Test
    void shouldProcessAndAck() throws Exception {
        TransferEvent event = ...;
        template.send("banking.transfers", event.getId().toString(), event).get();
        
        await().atMost(10, SECONDS).untilAsserted(() ->
            verify(notificationService).sendSms(event)
        );
    }
    
    @Test
    void shouldRetryThenDlt() throws Exception {
        doThrow(new TransientDataAccessException("temp"))
            .when(notificationService).sendSms(any());
        
        TransferEvent event = ...;
        template.send("banking.transfers", event.getId().toString(), event).get();
        
        // 5 retry
        await().atMost(60, SECONDS).untilAsserted(() ->
            verify(notificationService, times(5)).sendSms(any())
        );
        
        // DLT'de mesaj
        KafkaConsumer<String, TransferEvent> dltConsumer = createTestConsumer("banking.transfers.DLT");
        ConsumerRecords<String, TransferEvent> records = dltConsumer.poll(Duration.ofSeconds(10));
        assertThat(records).isNotEmpty();
    }
}
```

### Test 6.4.2 — Batch listener

```java
@Test
void batchListenerShouldProcessInChunks() throws Exception {
    List<AuditEvent> events = createEvents(100);
    
    for (AuditEvent e : events) {
        template.send("banking.audit", e.getId(), e);
    }
    
    await().atMost(20, SECONDS).untilAsserted(() -> {
        assertThat(auditRepo.count()).isEqualTo(100);
    });
    
    // saveAll called fewer times (batching)
    verify(auditRepo, atMost(20)).saveAll(any());
}
```

### Test 6.4.3 — @RetryableTopic

```java
@Test
void retryableTopicShouldRouteThroughRetryTopics() throws Exception {
    AtomicInteger calls = new AtomicInteger();
    doAnswer(inv -> {
        if (calls.incrementAndGet() < 4) throw new TransientDataAccessException("retry");
        return null;
    }).when(notificationService).sendSms(any());
    
    TransferEvent event = ...;
    template.send("banking.transfers", event.getId().toString(), event).get();
    
    await().atMost(30, SECONDS).untilAsserted(() -> {
        assertThat(calls.get()).isEqualTo(4);
    });
    
    // Retry topic'leri yaratıldı mı?
    Set<String> topics = kafkaAdmin.listTopics().listings().get().stream()
        .map(TopicListing::name)
        .collect(Collectors.toSet());
    assertThat(topics).contains(
        "banking.transfers", 
        "banking.transfers-retry-0",
        "banking.transfers-retry-1",
        "banking.transfers-retry-2"
    );
}
```

---

## Claude-verify prompt

```
Spring Kafka integration kodumu banking-grade kriterlere göre değerlendir:

1. Container factory config:
   - ConsumerFactory: enable-auto-commit=false, isolation_level=read_committed?
   - ListenerContainerFactory: MANUAL_IMMEDIATE, concurrency, error handler?
   - Observability enabled (micrometer, tracing)?

2. @KafkaListener:
   - id parameter (lifecycle control için)?
   - groupId explicit (defaults yerine)?
   - containerFactory belirtilmiş?
   - Method exception swallow YAPMIYOR (rethrow)?

3. Error handling:
   - DefaultErrorHandler + DeadLetterPublishingRecoverer?
   - Exponential backoff (1s, 2s, 4s, 8s)?
   - addNotRetryableExceptions specific class'larla?
   - DLT consumer + alert?
   - Custom DLT routing (poison vs transient)?

4. @RetryableTopic:
   - Non-blocking retry pattern banking için kullanılmış mı?
   - @DltHandler ile final failure?

5. Batch listener:
   - High-volume topic'ler için batch=true?
   - saveAll bulk insert?

6. Transactional listener:
   - Read-process-write transaction (consume + publish atomic)?

7. Header propagation:
   - X-Trace-Id, X-Tenant-Id MDC'ye konuluyor mu?

8. Lifecycle control:
   - KafkaListenerEndpointRegistry ile pause/resume?
   - Maintenance window pattern?

9. Banking patterns:
   - Idempotent consumer (processed_events) entegre mi?
   - Metrics counter (success/failure/duplicate)?

10. Anti-pattern:
    - Container factory default config?
    - Listener exception swallow?
    - ReplyingKafkaTemplate sync HTTP yerine?
    - Topic pattern wildcard?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] ContainerFactory full config (manual ack, concurrency, error handler)
- [ ] Batch listener pattern denedim
- [ ] @RetryableTopic non-blocking retry
- [ ] Transactional listener (read-process-write)
- [ ] Lifecycle control (pause/resume)
- [ ] Header propagation (X-Trace-Id)
- [ ] Custom DLT routing
- [ ] Idempotent consumer entegrasyon
- [ ] TestContainers ile 4+ integration test
- [ ] Maintenance window scheduled task

---

## Defter notları (10 madde)

1. "Spring Kafka layered architecture: ____."
2. "ContainerFactory ve ProducerFactory ayrımı: ____."
3. "@KafkaListener `id` parameter lifecycle control için: ____."
4. "Batch listener vs per-record — banking ne zaman hangisi: ____."
5. "@RetryableTopic non-blocking retry avantajı (main consumer): ____."
6. "Transactional listener read-process-write atomicity: ____."
7. "DefaultErrorHandler + custom DLT routing: ____."
8. "Header propagation X-Trace-Id MDC entegrasyonu: ____."
9. "Lifecycle pause/resume maintenance window pattern: ____."
10. "ReplyingKafkaTemplate vs HTTP/gRPC anti-pattern: ____."
