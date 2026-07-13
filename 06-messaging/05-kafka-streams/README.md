# Topic 6.5 — Kafka Streams: Real-Time Stream Processing

## Hedef

Kafka Streams ile **continuous stream processing** öğrenmek. KStream/KTable/GlobalKTable kavramları, stateless ve stateful operations, windowing, join semantics, state store, exactly-once processing, interactive queries. Banking için **real-time fraud detection** ve **transaction enrichment** pipeline'ları kurabilmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 6.1-6.4 bitti (Kafka arch, producer, consumer, Spring Kafka)
- Idempotency + exactly-once kavramları biliniyor
- Banking domain: transfer event, fraud rule, account state

---

## Kavramlar

### 1. Stream processing ne, neden Kafka Streams

Banking'de **continuous data processing** ihtiyaçları:
- Real-time fraud detection (transaction geldiğinde anlık skoru)
- Transaction enrichment (transfer + FX rate + customer info join)
- Aggregation (per-account daily totals, rolling windows)
- Anomaly detection (Z-score, threshold breaches)

3 yöntem:

| Yöntem | Latency | Complexity | Banking Adoption |
|---|---|---|---|
| Batch (Spring Batch) | Saatler | Düşük | EOD jobs (Phase 5) |
| Reactive consumer + DB | Saniyeler | Orta | Yaygın |
| **Stream processing** | Milisaniye | Yüksek | Modern banking |

**Kafka Streams** library — broker değil, **client uygulama** içinde çalışır. Spring Boot içinde **JAR olarak deploy**.

vs Apache Flink, Spark Streaming:
- **Kafka Streams:** Lib, Spring Boot içinde, deployment simple
- **Flink/Spark:** Cluster, separate infra, daha güçlü ama operasyonel yük

**Banking pratiği:** Real-time fraud detection ve event enrichment için Kafka Streams. Heavy analytics için Flink.

### 2. KStream — event stream

```java
KStream<String, TransferEvent> transfers = builder.stream(
    "banking.transfers",
    Consumed.with(Serdes.String(), transferEventSerde())
);
```

Her record bir **event** (immutable, append-only). Her transfer bir KStream record.

**Banking örnek:**

```
Time: 10:00:01 → Transfer(A→B, 100 TL)
Time: 10:00:05 → Transfer(C→D, 500 TL)
Time: 10:00:08 → Transfer(A→E, 200 TL)
```

KStream'in her record'u **bağımsız event**.

### 3. KTable — changelog stream

```java
KTable<String, AccountBalance> balances = builder.table(
    "banking.account-balances",
    Consumed.with(Serdes.String(), accountBalanceSerde())
);
```

Her record bir **entity'nin son hali** (key-based, update semantics). Aynı key gelen yeni record → eski'yi **güncelle**.

**Banking örnek:**

```
Time: 10:00:01 → AccountBalance(A, 1000 TL)
Time: 10:00:05 → AccountBalance(A, 800 TL)    ← A'nın son hali
Time: 10:00:10 → AccountBalance(B, 500 TL)
```

KTable'da A için sadece **800 TL** kalır (sonuncusu). Eski 1000 TL tarihte.

**KStream vs KTable mental model:**
- KStream: "transferleri logla" — event log
- KTable: "balance'lar nereye geldi" — current state

### 4. GlobalKTable — full replicate

```java
GlobalKTable<String, CustomerInfo> customers = builder.globalTable(
    "banking.customer-info",
    Consumed.with(Serdes.String(), customerInfoSerde())
);
```

Tüm instance'larda **tam kopya**. Read-only lookup için. Stream-table join'lerde useful.

**Tehlike:** Memory'de tutulur. Büyük tablolarda **OOM riski**. Banking: küçük lookup tablo'ları (BIN list, currency list).

### 5. Stateless operations

#### filter

```java
KStream<String, TransferEvent> highValue = transfers
    .filter((key, event) -> event.getAmount().compareTo(new BigDecimal("10000")) > 0);

highValue.to("banking.transfers.high-value");
```

10000+ TL transfer'leri ayrı topic'e.

#### map / mapValues

```java
KStream<String, EnrichedTransfer> enriched = transfers
    .mapValues(event -> EnrichedTransfer.builder()
        .from(event.getFromAccount())
        .to(event.getToAccount())
        .amount(event.getAmount())
        .fxRate(fxService.getCurrentRate(event.getCurrency()))
        .build());
```

Banking: Transfer'i FX rate ile zenginleştir.

**Tuzak:** `mapValues` lambda'sında **external call** (DB, HTTP) yapma — every event'te roundtrip = throughput felaket. **Side state** kullan (KTable join, state store).

#### branch (split)

```java
Map<String, KStream<String, TransferEvent>> branches = transfers
    .split(Named.as("by-currency-"))
    .branch((k, v) -> "TRY".equals(v.getCurrency()), Branched.as("try"))
    .branch((k, v) -> "USD".equals(v.getCurrency()), Branched.as("usd"))
    .defaultBranch(Branched.as("other"));

branches.get("by-currency-try").to("banking.transfers.try");
branches.get("by-currency-usd").to("banking.transfers.usd");
branches.get("by-currency-other").to("banking.transfers.other");
```

Banking: Currency'e göre routing.

### 6. Stateful operations

#### groupBy + count

```java
KTable<String, Long> transferCountByAccount = transfers
    .groupBy((key, event) -> event.getFromAccount(), 
        Grouped.with(Serdes.String(), transferEventSerde()))
    .count(Materialized.as("transfer-count-store"));

transferCountByAccount.toStream().to("banking.account.transfer-counts");
```

Her account için cumulative count. State **RocksDB**'de tutulur (local disk).

#### aggregate (custom)

```java
KTable<String, AccountStatistics> stats = transfers
    .groupBy((k, v) -> v.getFromAccount(), Grouped.with(Serdes.String(), transferEventSerde()))
    .aggregate(
        AccountStatistics::empty,
        (key, event, agg) -> agg.addTransfer(event),
        Materialized.<String, AccountStatistics, KeyValueStore<Bytes, byte[]>>as("account-stats")
            .withValueSerde(accountStatsSerde())
    );

stats.toStream().to("banking.account.statistics");
```

Banking: Per-account statistics (total volume, count, avg, max).

### 7. Windowing — time-bounded aggregation

#### Tumbling window

```java
KTable<Windowed<String>, Long> hourlyCount = transfers
    .groupBy((k, v) -> v.getFromAccount(), Grouped.with(Serdes.String(), transferEventSerde()))
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
    .count();
```

Saatlik fixed window: `09:00-10:00`, `10:00-11:00`, ...

#### Hopping window

```java
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ZERO)
    .advanceBy(Duration.ofMinutes(1))
```

5 dakika pencere, her dakika yenisi başlar. Overlapping.

Banking: Sliding 5-minute fraud window, her dakika check.

#### Session window

```java
SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(5))
```

5 dakika aktivite yoksa yeni session başlar.

Banking örnek: Banking app session boundary — user 5 dk hareketsizse session bitti.

### 8. Banking örnek — Real-time fraud detection

**Kural:** Aynı karttan **1 dakika içinde 5+ transaction** → fraud alert.

```java
@Configuration
@EnableKafkaStreams
public class FraudDetectionTopology {
    
    @Bean
    public KStream<String, TransferEvent> fraudPipeline(StreamsBuilder builder) {
        
        // 1. Source stream
        KStream<String, TransferEvent> transfers = builder.stream(
            "banking.transfers",
            Consumed.with(Serdes.String(), transferEventSerde())
        );
        
        // 2. Filter card transactions only
        KStream<String, TransferEvent> cardTransactions = transfers
            .filter((k, v) -> v.getCardId() != null);
        
        // 3. Group by cardId, window 1 minute
        KTable<Windowed<String>, Long> txPerCardPerMinute = cardTransactions
            .groupBy((k, v) -> v.getCardId().toString(), 
                Grouped.with(Serdes.String(), transferEventSerde()))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .count(Materialized.as("tx-count-per-card-per-minute"));
        
        // 4. Filter > 5 + emit alert
        KStream<String, FraudAlert> fraudAlerts = txPerCardPerMinute
            .toStream()
            .filter((windowedKey, count) -> count >= 5)
            .map((windowedKey, count) -> {
                UUID cardId = UUID.fromString(windowedKey.key());
                Instant windowStart = windowedKey.window().startTime();
                Instant windowEnd = windowedKey.window().endTime();
                
                FraudAlert alert = FraudAlert.builder()
                    .alertId(UUID.randomUUID())
                    .cardId(cardId)
                    .ruleCode("HIGH_FREQUENCY_TRANSACTIONS")
                    .severity(count >= 10 ? "CRITICAL" : "HIGH")
                    .score(Math.min(count * 10, 100))
                    .windowStart(windowStart)
                    .windowEnd(windowEnd)
                    .transactionCount(count.intValue())
                    .detectedAt(Instant.now())
                    .build();
                
                return KeyValue.pair(cardId.toString(), alert);
            });
        
        // 5. Sink to alerts topic
        fraudAlerts.to("banking.fraud.alerts",
            Produced.with(Serdes.String(), fraudAlertSerde()));
        
        return transfers;
    }
    
    @Bean
    public KafkaStreamsConfiguration kStreamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "banking-fraud-detection");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
        props.put(StreamsConfig.STATE_DIR_CONFIG, "/var/lib/kafka-streams");
        props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 3);
        return new KafkaStreamsConfiguration(props);
    }
}
```

**Banking output:**

```
banking.fraud.alerts:
  Time 10:00:30 → FraudAlert(card-X, count=6, HIGH)
  Time 10:01:45 → FraudAlert(card-Y, count=12, CRITICAL)
```

Anlık alert, downstream consumer (alert-service) müşteriye SMS, kartı geçici bloke vs.

### 9. Stream-table join (enrichment)

```java
KStream<String, TransferEvent> transfers = builder.stream("banking.transfers");
GlobalKTable<String, AccountInfo> accounts = builder.globalTable("banking.account-info");

KStream<String, EnrichedTransfer> enriched = transfers.leftJoin(
    accounts,
    (transferKey, transferValue) -> transferValue.getFromAccount(),   // foreign key extractor
    (transferValue, accountInfo) -> EnrichedTransfer.builder()
        .transfer(transferValue)
        .fromAccountName(accountInfo != null ? accountInfo.getName() : "UNKNOWN")
        .fromAccountTier(accountInfo != null ? accountInfo.getTier() : "STANDARD")
        .build()
);

enriched.to("banking.transfers.enriched");
```

Banking: Transfer event + customer info (name, tier) → enriched event downstream.

### 10. Stream-stream join

```java
KStream<String, TransferEvent> transfers = builder.stream("banking.transfers");
KStream<String, FraudScore> scores = builder.stream("banking.fraud.scores");

KStream<String, ScoredTransfer> joined = transfers.join(
    scores,
    (transfer, score) -> new ScoredTransfer(transfer, score),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5))
);
```

5 dakika içinde aynı key'le iki stream'de olan event'leri eşle.

Banking: Transfer ve fraud score genelde paralel akar. Join'le birleştir.

### 11. State store — local persistent state

Stateful operations (`count`, `aggregate`, `join`) **local RocksDB**'de state tutar.

```
/var/lib/kafka-streams/
  banking-fraud-detection/
    0_0/        ← Stream task 0_0
      tx-count-per-card-per-minute/
        rocksdb/
          ...
```

Her stateful operation → state store. Changelog topic otomatik yaratılır (`<app-id>-<store>-changelog`). Replication için.

### 12. Exactly-once processing

```java
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
```

Kafka Streams **atomic read → process → write**:
- Input topic offset commit
- State store update
- Output topic publish

Hepsi **tek transaction**. Crash → rollback, restart → kaldığı yerden.

**Banking için kritik.** Fraud detection'da duplicate alert = customer rahatsızlığı, missed alert = kayıp.

### 13. Interactive queries

State store'a app içinden read access:

```java
@Component
public class FraudQueryService {
    
    @Autowired
    private StreamsBuilderFactoryBean factoryBean;
    
    public Long getCurrentMinuteCount(UUID cardId) {
        KafkaStreams streams = factoryBean.getKafkaStreams();
        
        ReadOnlyWindowStore<String, Long> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                "tx-count-per-card-per-minute",
                QueryableStoreTypes.windowStore())
        );
        
        Instant now = Instant.now();
        Instant oneMinuteAgo = now.minus(1, ChronoUnit.MINUTES);
        
        try (WindowStoreIterator<Long> iter = store.fetch(
            cardId.toString(), oneMinuteAgo, now)) {
            
            long total = 0;
            while (iter.hasNext()) {
                total += iter.next().value;
            }
            return total;
        }
    }
}
```

REST endpoint:

```java
@RestController
public class FraudController {
    
    @Autowired FraudQueryService queryService;
    
    @GetMapping("/cards/{id}/transaction-count")
    public Long getCount(@PathVariable UUID id) {
        return queryService.getCurrentMinuteCount(id);
    }
}
```

Banking pattern: Read model — DB sorgulamadan, state store'dan anlık veriler.

**Tehlike:** Kafka Streams instance restart sırasında state store rebuild — interactive query 5-10 dk unavailable.

### 14. Topology + processor API

Yüksek seviye DSL yetmezse, low-level Processor API:

```java
public class FraudProcessor implements Processor<String, TransferEvent, String, FraudAlert> {
    
    private ProcessorContext<String, FraudAlert> context;
    private KeyValueStore<String, AggregateState> store;
    
    @Override
    public void init(ProcessorContext<String, FraudAlert> context) {
        this.context = context;
        this.store = context.getStateStore("fraud-state");
        
        // Punctuator — periodic check (her dakika)
        context.schedule(Duration.ofMinutes(1), PunctuationType.WALL_CLOCK_TIME, ts -> {
            try (KeyValueIterator<String, AggregateState> iter = store.all()) {
                while (iter.hasNext()) {
                    KeyValue<String, AggregateState> entry = iter.next();
                    if (entry.value.shouldAlert(ts)) {
                        context.forward(new Record<>(entry.key, entry.value.toAlert(), ts));
                        store.put(entry.key, entry.value.reset());
                    }
                }
            }
        });
    }
    
    @Override
    public void process(Record<String, TransferEvent> record) {
        AggregateState state = store.get(record.key());
        if (state == null) state = AggregateState.empty();
        state = state.addEvent(record.value(), record.timestamp());
        store.put(record.key(), state);
    }
}
```

Banking: Custom windowing logic veya punctuator-based time triggers.

### 15. Spring Kafka Streams entegrasyonu

```yaml
spring:
  kafka:
    streams:
      application-id: banking-fraud-detection
      bootstrap-servers: kafka:9092
      properties:
        default.key.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
        default.value.serde: org.apache.kafka.common.serialization.Serdes$ByteArraySerde
        processing.guarantee: exactly_once_v2
        commit.interval.ms: 1000
        num.stream.threads: 3
        state.dir: /var/lib/kafka-streams
```

```java
@SpringBootApplication
@EnableKafkaStreams
public class FraudDetectionApplication {
    public static void main(String[] args) {
        SpringApplication.run(FraudDetectionApplication.class, args);
    }
}
```

`@EnableKafkaStreams` ile Spring otomatik StreamsBuilderFactoryBean yaratır. Topology bean'leri auto-discovered.

### 16. ksqlDB — alternative

SQL üzerinden Kafka Streams:

```sql
CREATE STREAM transfers (
    transfer_id VARCHAR KEY,
    card_id VARCHAR,
    amount DECIMAL(19, 4),
    currency VARCHAR
) WITH (KAFKA_TOPIC='banking.transfers', VALUE_FORMAT='AVRO');

CREATE TABLE fraud_alerts AS
SELECT 
    card_id,
    COUNT(*) AS tx_count,
    WINDOWSTART AS window_start
FROM transfers
WINDOW TUMBLING (SIZE 1 MINUTE)
GROUP BY card_id
HAVING COUNT(*) >= 5
EMIT CHANGES;
```

Banking adoption: SQL bilen analyst'lar prototyping için ksqlDB. Production deployment Kafka Streams Java tarafında.

### 17. Banking pattern — multi-stage fraud pipeline

```
Stage 1: Raw transfer stream
  ↓
Stage 2: Enrichment (account info via global table)
  ↓
Stage 3: Rule 1 — high frequency (1 min, 5+ tx)
Stage 4: Rule 2 — cumulative amount (24h, 100k+ TL)
Stage 5: Rule 3 — foreign destination (first time)
  ↓
Stage 6: Score aggregation (weighted sum)
  ↓
Stage 7: Threshold filter (score > 70)
  ↓
Output: banking.fraud.alerts
```

Her stage ayrı KStream operation. Topology görsel olarak:

```java
@Bean
public KStream<String, TransferEvent> fraudTopology(StreamsBuilder builder) {
    KStream<String, TransferEvent> raw = builder.stream("banking.transfers");
    GlobalKTable<String, AccountInfo> accounts = builder.globalTable("banking.accounts");
    
    KStream<String, EnrichedTransfer> enriched = raw.leftJoin(accounts, ...);
    
    KStream<String, FraudScore> highFreqScore = applyHighFrequencyRule(enriched);
    KStream<String, FraudScore> cumScore = applyCumulativeAmountRule(enriched);
    KStream<String, FraudScore> foreignScore = applyForeignDestRule(enriched);
    
    KStream<String, FraudScore> totalScore = highFreqScore.merge(cumScore).merge(foreignScore);
    
    KStream<String, FraudAlert> alerts = totalScore
        .filter((k, score) -> score.getTotal() > 70)
        .mapValues(score -> FraudAlert.from(score));
    
    alerts.to("banking.fraud.alerts");
    
    return raw;
}
```

### 18. Banking anti-pattern'leri

**Anti-pattern 1: `mapValues` lambda'da external call**

```java
.mapValues(event -> {
    AccountInfo info = accountRepository.findById(event.getAccountId());   // DB call
    return enrichWith(event, info);
})
```

Her event = DB roundtrip. **Throughput felaket.** GlobalKTable join veya state store kullan.

**Anti-pattern 2: GlobalKTable'a büyük data**

```java
GlobalKTable<String, FullCustomerProfile> customers = builder.globalTable("customers");
// 10M müşteri × 5KB = 50GB memory per instance
```

Selective fields, küçük lookup tablo'ları.

**Anti-pattern 3: `EXACTLY_ONCE_V2` yokken duplicate sorunu**

Banking için her zaman exactly-once.

**Anti-pattern 4: State store rebuild beklenmeden interactive query**

Restart sonrası 5-10 dk store unavailable. UI/API çağrıları timeout. **Health check** ile guard.

**Anti-pattern 5: Stream-stream join window çok geniş**

```java
JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofDays(1))
```

24 saat'lik join state = büyük RocksDB. **Realistic window** seç.

**Anti-pattern 6: Topology'i runtime'da değiştirmek**

Kafka Streams topology immutable after `start()`. Değişiklik = rebalance + state lost. **Schema/topology değişikliği release-driven**.

---

## Önemli olabilecek araştırma kaynakları

- "Kafka Streams in Action" (Bill Bejeck)
- Confluent Kafka Streams documentation
- Apache Kafka KIP-129 (exactly-once)
- ksqlDB documentation
- Spring Kafka Streams reference
- "Designing Event-Driven Systems" (Ben Stopford)

---

## Mini task'ler

### Task 6.5.1 — Basic stream filter + map (30 dk)

Spring Boot + Kafka Streams setup. `banking.transfers` topic'inden 10K+ TL transfer'leri `banking.transfers.high-value` topic'ine route.

```java
@Bean
public KStream<String, TransferEvent> highValueStream(StreamsBuilder builder) {
    KStream<String, TransferEvent> transfers = builder.stream("banking.transfers");
    
    transfers
        .filter((k, v) -> v.getAmount().compareTo(new BigDecimal("10000")) > 0)
        .to("banking.transfers.high-value");
    
    return transfers;
}
```

Producer test: 100 transfer gönder, 10K+ olan kaç tane → high-value topic'te aynı sayı görmeli.

### Task 6.5.2 — Tumbling window count (45 dk)

Her account için saatlik transfer count.

```java
KTable<Windowed<String>, Long> hourlyCount = transfers
    .groupBy((k, v) -> v.getFromAccount(), 
        Grouped.with(Serdes.String(), transferEventSerde()))
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
    .count();
```

Output topic'te `(windowed-key, count)`. Topic'i tüket ve gör.

### Task 6.5.3 — Fraud detection pipeline (60 dk)

`fraudPipeline` bean'i yukarıdaki kod. 1 dakika içinde 5+ transaction → alert.

Test:
- 4 transaction same card 50 sn'de → alert YOK
- 5 transaction same card 50 sn'de → alert VAR

### Task 6.5.4 — Stream-table join (enrichment) (45 dk)

`banking.account-info` GlobalKTable. Transfer event + customer info → enriched event.

```java
KStream<String, EnrichedTransfer> enriched = transfers.leftJoin(
    accounts,
    (k, v) -> v.getFromAccount(),
    (transferValue, accountInfo) -> ...
);
```

### Task 6.5.5 — State store interactive query (45 dk)

REST endpoint `/cards/{id}/transaction-count` → state store'dan oku.

Test: card-X için 3 transaction gönder → endpoint count=3 dönmeli.

### Task 6.5.6 — exactly_once_v2 + duplicate test (30 dk)

`processing.guarantee=exactly_once_v2`. Producer aynı event'i 2 kez send. Output topic'te **tek mesaj** olmalı.

### Task 6.5.7 — Multi-stage fraud pipeline (60 dk)

3 rule (high freq, cumulative, foreign) → score aggregation → threshold filter → alert.

---

## Test yazma rehberi

### Test 6.5.1 — TopologyTestDriver (unit test)

```java
class FraudDetectionTopologyTest {
    
    private TopologyTestDriver driver;
    private TestInputTopic<String, TransferEvent> inputTopic;
    private TestOutputTopic<String, FraudAlert> outputTopic;
    
    @BeforeEach
    void setUp() {
        StreamsBuilder builder = new StreamsBuilder();
        new FraudDetectionTopology().fraudPipeline(builder);
        Topology topology = builder.build();
        
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test-fraud");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:9092");
        
        driver = new TopologyTestDriver(topology, props);
        
        inputTopic = driver.createInputTopic("banking.transfers",
            Serdes.String().serializer(), transferEventSerde().serializer());
        outputTopic = driver.createOutputTopic("banking.fraud.alerts",
            Serdes.String().deserializer(), fraudAlertSerde().deserializer());
    }
    
    @AfterEach
    void tearDown() {
        driver.close();
    }
    
    @Test
    void shouldNotAlertWhenLessThan5TransactionsInWindow() {
        UUID cardId = UUID.randomUUID();
        Instant base = Instant.parse("2025-01-01T10:00:00Z");
        
        for (int i = 0; i < 4; i++) {
            inputTopic.pipeInput(cardId.toString(),
                TransferEvent.builder().cardId(cardId).amount(BigDecimal.TEN).build(),
                base.plusSeconds(i * 10));
        }
        
        List<KeyValue<String, FraudAlert>> alerts = outputTopic.readKeyValuesToList();
        assertThat(alerts).isEmpty();
    }
    
    @Test
    void shouldAlertWhenFiveTransactionsInOneMinute() {
        UUID cardId = UUID.randomUUID();
        Instant base = Instant.parse("2025-01-01T10:00:00Z");
        
        for (int i = 0; i < 5; i++) {
            inputTopic.pipeInput(cardId.toString(),
                TransferEvent.builder().cardId(cardId).amount(BigDecimal.TEN).build(),
                base.plusSeconds(i * 10));
        }
        
        // Window kapanması için time advance
        driver.advanceWallClockTime(Duration.ofMinutes(2));
        
        List<KeyValue<String, FraudAlert>> alerts = outputTopic.readKeyValuesToList();
        assertThat(alerts).hasSize(1);
        FraudAlert alert = alerts.get(0).value;
        assertThat(alert.getCardId()).isEqualTo(cardId);
        assertThat(alert.getTransactionCount()).isEqualTo(5);
        assertThat(alert.getRuleCode()).isEqualTo("HIGH_FREQUENCY_TRANSACTIONS");
    }
    
    @Test
    void shouldEscalateToCriticalAt10Transactions() {
        UUID cardId = UUID.randomUUID();
        Instant base = Instant.parse("2025-01-01T10:00:00Z");
        
        for (int i = 0; i < 10; i++) {
            inputTopic.pipeInput(cardId.toString(),
                TransferEvent.builder().cardId(cardId).amount(BigDecimal.TEN).build(),
                base.plusSeconds(i * 5));
        }
        
        driver.advanceWallClockTime(Duration.ofMinutes(2));
        
        List<KeyValue<String, FraudAlert>> alerts = outputTopic.readKeyValuesToList();
        assertThat(alerts).hasSize(1);
        assertThat(alerts.get(0).value.getSeverity()).isEqualTo("CRITICAL");
    }
}
```

### Test 6.5.2 — Integration with TestContainers

```java
@SpringBootTest
@Testcontainers
class FraudDetectionIT {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(...);
    
    @Test
    void endToEndFraudDetection() throws Exception {
        UUID cardId = UUID.randomUUID();
        
        for (int i = 0; i < 5; i++) {
            kafkaTemplate.send("banking.transfers", cardId.toString(),
                TransferEvent.builder().cardId(cardId).amount(BigDecimal.TEN).build());
        }
        
        // Consume fraud alerts
        KafkaConsumer<String, FraudAlert> consumer = createTestConsumer("banking.fraud.alerts");
        
        await().atMost(30, SECONDS).untilAsserted(() -> {
            ConsumerRecords<String, FraudAlert> records = consumer.poll(Duration.ofSeconds(2));
            assertThat(records).isNotEmpty();
            
            FraudAlert alert = records.iterator().next().value();
            assertThat(alert.getCardId()).isEqualTo(cardId);
        });
    }
}
```

---

## Claude-verify prompt

```
Kafka Streams kodumu banking-grade kriterlere göre değerlendir:

1. Stream / table tip seçimi:
   - Event stream için KStream?
   - State/changelog için KTable?
   - Read-only lookup için GlobalKTable (küçük data)?

2. Stateless operations:
   - filter, map, mapValues banking örneği uygun mu?
   - mapValues içinde external call (DB) YAPILMIYOR mu?

3. Stateful operations:
   - groupBy + count/aggregate pattern doğru mu?
   - Materialized.as ile state store named mi?

4. Windowing:
   - Tumbling vs Hopping vs Session uygun seçilmiş mi?
   - Window size banking kuralına uygun (1 dk fraud, 24h cumulative)?

5. Joins:
   - Stream-table (enrichment) için GlobalKTable veya KTable?
   - Stream-stream join window realistic mi?

6. Exactly-once:
   - processing.guarantee=EXACTLY_ONCE_V2 aktif mi?
   - Duplicate event test edildi mi?

7. State store + interactive queries:
   - Local RocksDB state store?
   - Interactive query REST endpoint?
   - Restart sırasında store unavailable handling?

8. Banking fraud pipeline:
   - Multi-rule pipeline (high freq + cumulative + foreign)?
   - Score aggregation + threshold filter?
   - Alert event yayını?

9. Anti-pattern:
   - mapValues içinde DB call?
   - Büyük GlobalKTable (OOM riski)?
   - exactly_once_v2 yok?
   - Çok geniş join window?

10. Test:
    - TopologyTestDriver unit test?
    - TestInputTopic / TestOutputTopic kullanımı?
    - advanceWallClockTime ile window kapanması?
    - TestContainers ile integration test?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Spring Kafka Streams setup
- [ ] Basic filter + map stream
- [ ] Tumbling window count
- [ ] Fraud detection pipeline (1 dk window, 5+ count)
- [ ] Stream-table join (enrichment)
- [ ] State store + interactive query
- [ ] exactly_once_v2 + duplicate test
- [ ] Multi-stage fraud pipeline
- [ ] TopologyTestDriver unit test'ler
- [ ] TestContainers integration test

---

## Defter notları (10 madde)

1. "KStream vs KTable vs GlobalKTable kullanım farkı banking örneği: ____."
2. "Stateless vs stateful operation — state store nerede tutulur: ____."
3. "Tumbling / Hopping / Session window banking örneği: ____."
4. "Stream-stream join window genişliği — realistic değerleri: ____."
5. "GlobalKTable OOM tuzağı, lookup table size limit: ____."
6. "exactly_once_v2 atomic read-process-write: ____."
7. "State store local RocksDB + changelog topic backup: ____."
8. "Interactive query restart sırasında unavailable handling: ____."
9. "Processor API ile DSL arasında karar (low-level vs high-level): ____."
10. "Banking real-time fraud detection 5+ tx/1dk pattern Java code: ____."
