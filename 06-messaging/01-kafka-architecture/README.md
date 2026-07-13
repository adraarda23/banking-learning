# Topic 6.1 — Kafka Mimarisi: Broker, Topic, Partition, Replica

## Hedef

Kafka'nın **distributed commit log** olarak nasıl çalıştığını, bir mesajın disk'e nasıl yazıldığını, kümede nasıl replicate edildiğini, partition'ın neden Kafka'nın can damarı olduğunu derinlemesine öğrenmek. Banking projende `TransferCompleted` topic'ini doğru partition'lamak için karar verebilecek seviyeye gelmek.

## Süre

Okuma: 2 saat • Kurulum: 1 saat • Mini task'ler: 2 saat • Toplam: ~5 saat

## Önbilgi

- Docker compose temel (Faz 2'de PostgreSQL container'ı çalıştırdın)
- TCP/IP ve disk I/O hakkında genel fikir
- Java application'ın network'e nasıl bağlandığı

---

## Kavramlar

### 1. Kafka nedir, nasıl tanımlanmalı

LinkedIn, 2011'de Jay Kreps liderliğinde başlattı. Apache vakfına 2012'de devretti. **"Distributed event streaming platform"** olarak adlandırırlar.

Üç ana yetenek:
1. **Publish & subscribe** event stream'lerine
2. **Store** event'leri durable ve fault-tolerant şekilde
3. **Process** event'leri real-time olarak (Kafka Streams)

Junior'ın yanlış anladığı: Kafka **bir mesaj kuyruğu** zannediliyor. Aslında bir **distributed, append-only commit log**'dur. Aradaki fark:

| Klasik queue | Kafka topic |
|---|---|
| Consumer mesajı alınca silinir | Mesaj retention period kadar diskte kalır |
| FIFO sırası tek bir queue içinde | Sıra sadece partition içinde garanti |
| Bir mesajı sadece bir consumer alır | Aynı mesaj her consumer group'a teslim edilir |
| Broker mesaj sırasını yönetir | Consumer offset'i kendi yönetir (Kafka tutar ama yetki consumer'da) |
| Acknowledge → silinme | Acknowledge yok, offset commit var |

### 2. Topic — mantıksal kanal

Bir **topic**, ilgili event'lerin yayınlandığı isimlendirilmiş kanaldır.

Banking örneği:
- `transfer.completed` — başarılı transfer'ler
- `transfer.failed` — başarısız transfer'ler
- `account.opened` — yeni açılan hesaplar
- `account.frozen` — bloke edilen hesaplar
- `card.transaction.attempted` — kart işlem denemeleri (fraud'a beslenir)

**Naming convention** (TR bank'larda yaygın):
- `<domain>.<entity>.<verb-past-tense>` → `core.account.opened`, `payments.transfer.completed`
- Bazıları `<context>.<event>.v<version>` kullanır → `transfer.completed.v1` (schema evolution için)
- Underscore yerine **nokta** ile ayırma yaygındır (Kafka topic name'i underscore kabul eder ama nokta görsel olarak daha temiz).

**Topic seviyesinde yapılandırma:**
- Partition sayısı
- Replication factor
- Retention period (kaç gün diskte kalır)
- Cleanup policy (delete vs compact)
- Min in-sync replicas (`min.insync.replicas`)

### 3. Partition — Kafka'nın can damarı

Bir topic **bir veya birden fazla partition**'a bölünür. Partition, **append-only log file**'dır. Her mesaj sırayla disk'e yazılır.

```
Topic: transfer.completed (4 partition)

Partition 0:  [msg0][msg1][msg2][msg3][msg4]...
Partition 1:  [msg0][msg1][msg2][msg3][msg4]...
Partition 2:  [msg0][msg1][msg2][msg3][msg4]...
Partition 3:  [msg0][msg1][msg2][msg3][msg4]...
              ↑
              Offset 0 (her partition'da bağımsız)
```

**Önemli özellikler:**

1. **Sıra garantisi sadece partition içinde.** Partition 0'a yazılan `msg0` ve `msg1`, sırayla okunur. Ama `partition 0 msg2` ile `partition 1 msg2` arasında zaman ilişkisi **garanti değil**.

2. **Partition = parallelism unit.** 4 partition'lı topic = 4 consumer paralel okuyabilir. 1 partition'lı topic = 1 consumer (consumer group başına).

3. **Partition = scalability unit.** 1M msg/sec gerektiren bir topic için partition sayısını artırırsın. 10 yerine 100 partition.

4. **Partition'lar broker'lara dağılır.** 3 broker'lı cluster ve 6 partition'lı topic → her broker 2 partition'a (lider olarak) sahip.

### 4. Partition key — sıra ve dağıtım kontrolü

Producer mesaj gönderirken bir **key** verebilir. Kafka şu kuralı uygular:

```
partition = hash(key) % num_partitions
```

**Aynı key her zaman aynı partition'a gider.**

**Banking örneği — neden kritik:**

`TransferCompleted` event'ini partition'lamak için iki seçenek var:

**Seçenek A: Key yok (round-robin)**
```java
producer.send(new ProducerRecord<>("transfer.completed", null, event));
```
- Pro: Yük eşit dağılır
- Con: Aynı hesabın iki transfer event'i farklı partition'a düşebilir → consumer'larda sıra dışı işlenebilir
- Banking için **tehlikeli**: Önce transfer-2 işlenir, sonra transfer-1 → fraud servisi yanlış skor üretir

**Seçenek B: Key = accountId**
```java
producer.send(new ProducerRecord<>("transfer.completed", accountId.toString(), event));
```
- Pro: **Aynı hesabın tüm transfer'leri aynı partition'a → sırayla işlenir**
- Con: "Hot account" varsa (örn. holding banka hesabı) o partition aşırı yüklenir

**Banking'de yaygın çözüm:**
- `key = fromAccountId` (kaynak hesap perspektifinden sıra önemli)
- Hot account problemi varsa custom partitioner (ör. `key = fromAccountId + branchId`)

**Karar matrisi:**

| Kullanım | Partition key |
|---|---|
| Hesap event'leri (deposit, withdraw, transfer-out) | `accountId` |
| Kart işlemleri (fraud için kart bazlı sıralama) | `cardId` |
| Customer events (KYC update, blok) | `customerId` |
| Audit log (sıra önemsiz) | null (round-robin) |
| Cross-bank transfer | `correlationId` |

### 5. Offset — mesajın partition içindeki adresi

Her mesajın partition içinde **artan bir offset**'i var: 0, 1, 2, ...

```
Partition 0 of transfer.completed:
offset:  0    1    2    3    4    5    6
content: T0   T1   T2   T3   T4   T5   T6
                                       ↑
                                       LEO (Log End Offset)

Consumer A:           ↑ committed offset 3 (next read: 4)
Consumer B:                ↑ committed offset 4 (next read: 5)
```

- **Log End Offset (LEO):** Partition'a yazılmış son mesajın offset'i + 1.
- **Committed offset:** Consumer'ın "buraya kadar işledim" diye Kafka'ya bildirdiği nokta.
- **Lag = LEO - committed offset.** Consumer'ın ne kadar geri olduğunu gösterir. Banking ops için kritik metrik.

**Önemli kavrayış:** Kafka, mesajı consumer'a "vermez". Consumer Kafka'dan **fetch eder** (pull model), kendisi sırayla offset'i ilerletir. Bu yüzden:
- Consumer crash olsa, restart'tan sonra committed offset'ten devam eder.
- "Aynı mesajı tekrar oku" istersen offset'i geri al (offset reset).
- "Yeni event'lerden başla" istersen offset'i LEO'ya getir (`auto.offset.reset=latest`).

### 6. Broker — Kafka cluster'ının düğümü

Bir **broker**, Kafka cluster'ında bir JVM süreci. Tipik production: 3, 5, 7 broker.

Her broker:
- Partition'ları lokal diskinde tutar (log segment'ler şeklinde)
- Producer'lardan mesaj alır, partition'a append eder
- Consumer'ların fetch isteklerini cevaplar
- Diğer broker'larla replikasyon yapar

**Broker ID:** Her broker'ın unique integer ID'si var (`broker.id=1`). Cluster bunu kullanır.

### 7. Replication — fault tolerance

Her partition için **replication factor (RF)** belirlenir. RF=3 ise her partition 3 broker'da kopya tutulur:

```
Topic: transfer.completed, partitions: 4, RF: 3
Cluster: 3 broker

Partition 0:  Leader=Broker1, Followers=[Broker2, Broker3]
Partition 1:  Leader=Broker2, Followers=[Broker3, Broker1]
Partition 2:  Leader=Broker3, Followers=[Broker1, Broker2]
Partition 3:  Leader=Broker1, Followers=[Broker2, Broker3]
```

- **Leader:** Producer ve consumer **sadece leader ile** konuşur. Tüm read/write leader'dan.
- **Follower:** Leader'dan replicate eder. Read için kullanılmaz (Kafka 2.4'ten sonra follower fetching opsiyonel ama varsayılan değil).

**Avantaj:**
- Broker 1 düşerse, Partition 0'ın lider'i otomatik olarak Broker 2 veya 3 olur.
- Veri kaybı yok (eğer `acks=all` ve `min.insync.replicas` doğru ayarlıysa).

**Banking için RF kararı:**
- **Production: RF=3** (standart). Bir broker düşse 2 kopya hâlâ var.
- **RF=1 KESİNLİKLE YASAK** production'da — tek broker düşse data kaybı.
- **RF=2 önerilmez** — `min.insync.replicas=2` ile yazma için iki broker da gerekli olur, bir broker düştüğünde write durur.

### 8. ISR — In-Sync Replicas

Bir replica "in-sync" demek: leader'a göre **replica.lag.time.max.ms** (default 30 saniye) içinde kalmış demek.

Yavaş bir follower senkronlukta gecikirse ISR'den çıkarılır. ISR sayısı yazma garantisini belirler.

**`min.insync.replicas`** — yazma için kaç ISR olmalı:

```
RF=3, min.insync.replicas=2

İdeal durum: 3 ISR (1 lider + 2 follower). Write OK.
Bir follower düştü: 2 ISR. Write OK (min karşılandı).
İki follower düştü: 1 ISR. Write reddedilir (NotEnoughReplicasException).
```

**Banking örneği:**
- `transfer.completed` topic için `RF=3`, `min.insync.replicas=2`.
- İki broker hayatta olmadıkça transfer event'i **kabul edilmez**. Veri kaybı riskini minimize etmek için.
- Producer `acks=all` ile yazıyorsa, mesaj **tüm ISR'lere yazılana kadar** ack alınmaz.

### 9. Controller — cluster'ın "beyin" broker'ı

Cluster içinde **bir broker controller** rolündedir. Sorumlulukları:
- Partition leader election yapmak
- Yeni broker katılım/çıkışı yönetmek
- Topic'lerin metadata'sını yönetmek

Controller düşerse otomatik olarak başka bir broker controller olur (ZooKeeper veya KRaft üzerinden seçim).

### 10. ZooKeeper vs KRaft

**ZooKeeper (eski yöntem):**
- Kafka 0.x - 3.x boyunca cluster metadata'yı dış bir ZooKeeper cluster'ında tuttu.
- Operasyon karmaşası: iki cluster yönetmek (Kafka + ZK).
- Latency: leader election ZK'da, network round-trip ekstra.

**KRaft (Kafka Raft, modern yöntem):**
- Kafka 3.3+ production-ready (Kafka 4.0+ sadece KRaft).
- Cluster metadata Kafka'nın **kendi içinde** tutuluyor (internal topic).
- Tek cluster, daha basit ops.

**Banking'de durum:**
- Eski production cluster'lar **hâlâ ZooKeeper** kullanıyor (TR bankalarında yaygın).
- Yeni cluster'lar **KRaft**.
- Junior olarak ikisini de görebilirsin — temel kavramlar aynı, sadece metadata nerede tutuluyor farkı var.

Bu projede KRaft mode kullanacağız (yeni Docker imajları KRaft destekliyor).

### 11. Log segment ve dosya yapısı

Partition disk'te birden fazla **segment dosyası** halinde tutulur:

```
/var/kafka-logs/transfer.completed-0/
├── 00000000000000000000.log       (segment dosyası 1)
├── 00000000000000000000.index     (offset → byte position index)
├── 00000000000000000000.timeindex (timestamp → offset index)
├── 00000000000000150000.log       (segment dosyası 2, 150000 offset'inden başlar)
├── 00000000000000150000.index
└── 00000000000000150000.timeindex
```

**Segment**'ler:
- `log.segment.bytes` default 1GB → segment 1GB olunca yeni segment açılır.
- `log.roll.ms` default 7 gün → 7 gün sonra yeni segment açılır.
- Eski segment'ler retention policy'ye göre silinir veya compact edilir.

**Neden segment?**
- Sıralı disk I/O hızlıdır (Kafka'nın gücü).
- Eski mesajları silmek için **dosya silmek** yeterli (random delete yok).
- Index'ler sayesinde offset → byte position lookup O(1).

### 12. Retention policy

İki ana tür var:

**(a) `cleanup.policy=delete` (default)**

- `retention.ms`: mesajlar X süre sonra silinir (default 7 gün)
- `retention.bytes`: partition X byte aşınca eski mesajlar silinir

```
transfer.completed:
  retention.ms = 30 gün
  retention.bytes = -1 (sınırsız, sadece time-based)
```

**Banking:**
- `transfer.completed`: 30-90 gün (audit/replay için)
- `card.transaction`: 7-30 gün (fraud retroactive analysis için)
- Regulatory event'ler: 7-10 yıl (compliance) — büyük bütçe!

**(b) `cleanup.policy=compact` — log compaction**

Aynı key için **sadece en son değer** tutulur. Eski versiyonlar silinir.

```
Kafka log compaction example:
Önce:  (k1, v1) (k2, v2) (k1, v3) (k3, v4) (k2, v5)
Sonra: (k1, v3) (k3, v4) (k2, v5)
```

**Banking örneği — kullanım:**
- `account.balance.snapshot` — her hesap için en son balance. Müşterinin son durumu bilinmek isteniyor.
- `customer.kyc.status` — her müşteri için en son KYC durumu.
- Yeni event geldiğinde eski versiyon "silinir" (mark for deletion).

**Compact + delete** birlikte kullanılabilir (`cleanup.policy=compact,delete`):
- Compacted state + maximum retention.

**Tombstone:** `value=null` mesajı gönderirsen Kafka onu "sil" işareti olarak yorumlar. Compaction çalışınca o key tamamen silinir.

### 13. Banking için topic tasarımı — `TransferCompleted`

`core-banking` projende `TransferCompleted` event'i için tasarım:

**Topic config:**

```bash
kafka-topics --create \
  --topic transfer.completed.v1 \
  --bootstrap-server localhost:9092 \
  --partitions 12 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=2592000000  # 30 gün
  --config cleanup.policy=delete \
  --config compression.type=snappy
```

**Partition sayısı seçimi (12):**
- Beklenen throughput: 10K transfer/sec peak
- Bir partition ~10K msg/sec rahat işler → matematiksel olarak 1 partition yeterli.
- Ama **consumer paralelliği** için 12 partition. notification-service 6 instance, audit 6 instance, fraud 12 instance koşabilir.
- 12 = 2^2 * 3 → ileride 24, 36 partition'a kolay scale (multiple of mevcut sayı).

**Key seçimi:**
- `accountId` (transfer'in `fromAccountId`'si) — aynı hesabın transfer'leri sırayla.

**Schema (Avro veya JSON):**

```json
{
  "transferId": "uuid",
  "fromAccountId": "uuid",
  "toAccountId": "uuid",
  "amount": "10000.50",
  "currency": "TRY",
  "executedAt": "2025-05-12T14:30:00Z",
  "occurredAt": "2025-05-12T14:30:00.123Z",
  "version": 1,
  "correlationId": "uuid"
}
```

**Headers (Kafka header'lar):**

```
traceId: abc-123-def
spanId: xyz-456
sourceService: transfer-service
schemaVersion: 1
```

### 14. Banking anti-pattern'leri

**Anti-pattern 1: Tek partition topic'i**

```bash
kafka-topics --create --topic transfer.completed --partitions 1 --replication-factor 3
```

- Tüm yazma tek bir broker'a düşer (sadece 1 lider olduğu için)
- Tek bir consumer alabilir (bir consumer group'ta partition başına 1 consumer)
- Throughput sınırı 1 broker disk I/O'su

**Doğrusu:** Beklenen throughput'a göre `min(brokers * 2, throughput / 10K)` civarı partition.

**Anti-pattern 2: 1000 partition topic'i**

- ZooKeeper/KRaft metadata yükü artar
- Producer'da metadata cache büyür
- Rebalance süresi uzar
- Disk file descriptor sayısı patlar

**Doğrusu:** Toplam partition sayısını cluster çapında binlerle sınırla, topic başına 100'lerin altında tut.

**Anti-pattern 3: Key olmadan transfer event'i**

```java
producer.send(new ProducerRecord<>("transfer.completed", null, event));
```

- Mesajlar round-robin dağılır
- **Aynı hesabın iki transfer event'i farklı partition'a düşebilir**
- Consumer'da sıra dışı işlem → fraud false positive, balance inconsistency

**Doğrusu:** `key = fromAccountId.toString()`.

**Anti-pattern 4: PII (kişisel veri) ve hassas veri event payload'unda plain text**

```json
{
  "customerTcNo": "12345678901",
  "cardNumber": "4242 4242 4242 4242",
  "iban": "TR330006100519786457841326"
}
```

- KVKK/GDPR ihlali
- Kafka log'larında, replication target'larında, backup'larda hep var

**Doğrusu:**
- Mask: `cardNumber: "**** **** **** 4326"`
- Hash: `customerHash: "sha256:abc..."` ve PII'yi başka secure store'da tut
- Tokenize: PCI compliance için kart numarası tokenize edilmiş halde gönder

**Anti-pattern 5: Synchronous request/response Kafka üstünden**

Bazı geliştiriciler Kafka'yı "RPC" gibi kullanır: request topic + response topic + correlation ID. Çoğu kez yanlış.

- Kafka request/response için tasarlanmamış (high-latency, no built-in correlation)
- HTTP veya gRPC daha doğru
- Kafka **asynchronous event broadcast** için.

İstisna: Cross-system batch işlemler, "fire and forget + later poll".

**Anti-pattern 6: Single topic per use-case extreme**

Her küçük olay için ayrı topic açma:
- `transfer.completed.tl`
- `transfer.completed.usd`
- `transfer.completed.eur`
- `transfer.completed.amount.over.10k`

Tek `transfer.completed` topic + consumer'da filter yeterli. Topic sayısı patlamasın.

---

## Önemli olabilecek araştırma kaynakları

- "Kafka: The Definitive Guide" 2nd edition (O'Reilly, Neha Narkhede et al.)
- "Designing Event-Driven Systems" Ben Stopford (free O'Reilly e-book)
- Apache Kafka resmi docs — `kafka.apache.org/documentation/`
- "Apache Kafka Internals" blog yazıları (Confluent)
- "Kafka Streams in Action" Bill Bejeck (kitap, Phase 6 Topic 5 için)
- "Kafka improvement proposals (KIP)" — özellikle KIP-500 (KRaft), KIP-98 (exactly-once)
- Confluent developer portal — `developer.confluent.io`
- "Log: What every software engineer should know" Jay Kreps (uzun blog yazısı, manifesto niteliğinde)

---

## Mini task'ler

### Task 6.1.1 — Docker compose ile Kafka cluster kur (45 dk)

`~/projects/core-banking/docker-compose.yml`'a ekle (mevcut postgres'in yanına):

```yaml
services:
  postgres:
    # mevcut...
  
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: core-banking-kafka
    ports:
      - "9092:9092"
      - "9093:9093"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,EXTERNAL://0.0.0.0:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,EXTERNAL://localhost:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_MIN_INSYNC_REPLICAS: 1
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
    volumes:
      - kafka-data:/var/lib/kafka/data
  
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: core-banking-kafka-ui
    ports:
      - "8081:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    depends_on:
      - kafka

volumes:
  kafka-data:
```

> Not: Production'da 3 broker olur, RF=3 olur. Local için 1 broker + RF=1 (kaynak yetmiyor). Bunu defterine yaz.

```bash
docker compose up -d
docker ps | grep kafka
```

Kafka UI'ı aç: `http://localhost:8081`. Cluster görünüyor mu? Topic listesi şu an boş.

### Task 6.1.2 — `transfer.completed.v1` topic'i oluştur (15 dk)

CLI ile (container içinden):

```bash
docker exec -it core-banking-kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic transfer.completed.v1 \
  --partitions 4 \
  --replication-factor 1 \
  --config retention.ms=604800000 \
  --config cleanup.policy=delete
```

Topic'i listele ve detayını gör:

```bash
docker exec -it core-banking-kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic transfer.completed.v1
```

Çıktıda partition sayısı, leader, replicas, ISR'ı oku. **Defterine yaz**: "Partition 0'ın lideri broker ___, replicas ___, ISR ___."

Kafka UI'da da kontrol et (`http://localhost:8081/ui/clusters/local/all-topics`).

### Task 6.1.3 — Console producer/consumer ile manuel mesaj gönder (30 dk)

İki terminal aç.

Terminal 1 (producer):
```bash
docker exec -it core-banking-kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic transfer.completed.v1 \
  --property "parse.key=true" \
  --property "key.separator=:"
```

Terminal 2 (consumer):
```bash
docker exec -it core-banking-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic transfer.completed.v1 \
  --from-beginning \
  --property "print.key=true" \
  --property "key.separator=: "
```

Terminal 1'de yaz (key:value):
```
acc-001:{"transferId":"t1","amount":"100.00"}
acc-001:{"transferId":"t2","amount":"50.00"}
acc-002:{"transferId":"t3","amount":"75.00"}
acc-001:{"transferId":"t4","amount":"200.00"}
```

Consumer'da mesajları gör. **Aynı key'in mesajları ardışık mı?** Evet, çünkü hash(acc-001) hep aynı partition'a düşüyor.

### Task 6.1.4 — Partition assignment'i gör (15 dk)

```bash
docker exec -it core-banking-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic transfer.completed.v1 \
  --from-beginning \
  --property "print.partition=true" \
  --property "print.key=true" \
  --property "key.separator=: " \
  --property "print.value=true"
```

Hangi partition'a hangi key düştüğünü gözle. Birden fazla key gönder ve **aynı key her zaman aynı partition'a** mı kontrol et.

### Task 6.1.5 — Topic config değiştir (15 dk)

Topic retention'ını 1 saate düşür:

```bash
docker exec -it core-banking-kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name transfer.completed.v1 \
  --alter \
  --add-config retention.ms=3600000
```

Sonra describe et, değişikliği gör:

```bash
docker exec -it core-banking-kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name transfer.completed.v1 \
  --describe
```

> Banking ops gerçeği: Production topic config'i live değiştirmek mümkün ama dikkat — retention düşürürsen eski mesajlar silinir, geri alamazsın.

### Task 6.1.6 — Compacted topic dene (30 dk)

`account.balance.snapshot.v1` topic'i oluştur (compact):

```bash
docker exec -it core-banking-kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic account.balance.snapshot.v1 \
  --partitions 4 \
  --replication-factor 1 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.01 \
  --config segment.ms=1000
```

Producer terminalinde aynı hesap için 3 farklı balance gönder:

```
acc-001:{"balance":"1000.00"}
acc-001:{"balance":"850.00"}
acc-001:{"balance":"950.00"}
```

Birkaç saniye bekle (compaction tetiklenmesi için), sonra consumer'ı **baştan** başlat (`--from-beginning`). Sadece son değeri (`950.00`) görmen normal değil — compaction her zaman immediate değil, segment roll edip cleanup çalıştıktan sonra olur.

`segment.ms=1000` ile zorlamış olduk. Birkaç deneme yapman gerekebilir.

Tombstone gönder:
```
acc-001:
```
(value boş — null) → bu key'i silmek için. Compaction'dan sonra `acc-001` artık consumer'da gözükmeyecek.

### Task 6.1.7 — `core-banking-events` Maven modülü hazırla (30 dk)

Phase 6 boyunca event class'ları paylaşılacak. Domain'i temiz tutmak için **event tanımlarını ayrı bir modül**'de tutmak iyi pratik.

`~/projects/core-banking/` altına `events/` modülünü Maven multi-module olarak ekle (eğer henüz tek modül ise, bu fırsat).

`pom.xml` parent:
```xml
<modules>
    <module>app</module>      <!-- mevcut Spring Boot uygulaması -->
    <module>events</module>   <!-- yeni -->
</modules>
```

`events/pom.xml`:
```xml
<dependencies>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

`events/src/main/java/com/mavibank/banking/events/TransferCompletedEvent.java`:

```java
package com.mavibank.banking.events;

import com.fasterxml.jackson.annotation.JsonFormat;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

public record TransferCompletedEvent(
    UUID transferId,
    UUID fromAccountId,
    UUID toAccountId,
    BigDecimal amount,
    String currency,
    @JsonFormat(shape = JsonFormat.Shape.STRING)
    Instant executedAt,
    @JsonFormat(shape = JsonFormat.Shape.STRING)
    Instant occurredAt,
    int version,
    UUID correlationId
) {
    public static final String TOPIC = "transfer.completed.v1";
    public static final int CURRENT_VERSION = 1;
}
```

Şimdilik publish/consume yok — sadece **tanım**. Producer'ı Topic 6.2'de yazacağız.

---

## Test yazma rehberi

Bu topic'te kod testi azaltılmış (configuration odaklı). Yine de şunları yap:

### Test 6.1.1 — `TransferCompletedEventTest`

Event class'ının JSON serialization'ını doğrula:

```java
class TransferCompletedEventTest {
    
    private final ObjectMapper objectMapper = new ObjectMapper()
        .registerModule(new JavaTimeModule());
    
    @Test
    void shouldSerializeToJson() throws Exception {
        var event = new TransferCompletedEvent(
            UUID.fromString("11111111-1111-1111-1111-111111111111"),
            UUID.fromString("22222222-2222-2222-2222-222222222222"),
            UUID.fromString("33333333-3333-3333-3333-333333333333"),
            new BigDecimal("100.00"),
            "TRY",
            Instant.parse("2025-05-12T14:30:00Z"),
            Instant.parse("2025-05-12T14:30:00.123Z"),
            1,
            UUID.fromString("44444444-4444-4444-4444-444444444444")
        );
        
        String json = objectMapper.writeValueAsString(event);
        
        assertThat(json).contains("\"transferId\":\"11111111-1111-1111-1111-111111111111\"");
        assertThat(json).contains("\"amount\":100.00");
        assertThat(json).contains("\"currency\":\"TRY\"");
        assertThat(json).contains("\"executedAt\":\"2025-05-12T14:30:00Z\"");
        assertThat(json).contains("\"version\":1");
    }
    
    @Test
    void shouldDeserializeFromJson() throws Exception {
        String json = """
            {
              "transferId": "11111111-1111-1111-1111-111111111111",
              "fromAccountId": "22222222-2222-2222-2222-222222222222",
              "toAccountId": "33333333-3333-3333-3333-333333333333",
              "amount": "100.00",
              "currency": "TRY",
              "executedAt": "2025-05-12T14:30:00Z",
              "occurredAt": "2025-05-12T14:30:00.123Z",
              "version": 1,
              "correlationId": "44444444-4444-4444-4444-444444444444"
            }
            """;
        
        var event = objectMapper.readValue(json, TransferCompletedEvent.class);
        
        assertThat(event.transferId())
            .isEqualTo(UUID.fromString("11111111-1111-1111-1111-111111111111"));
        assertThat(event.amount()).isEqualByComparingTo("100.00");
        assertThat(event.version()).isEqualTo(1);
    }
    
    @Test
    void topicNameIsConstant() {
        assertThat(TransferCompletedEvent.TOPIC).isEqualTo("transfer.completed.v1");
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki Kafka kurulum ve topic tasarım kararlarımı banking konteks içinde 
değerlendir. Kod yazma, sadece eksik/yanlış olanları söyle:

1. docker-compose.yml:
   - Kafka tek broker (local) ile kuruldu mu? Production için NOT yazıldı mı?
   - KRaft mode (ZooKeeper'sız) kullanılıyor mu?
   - Kafka UI veya benzer monitoring tool ekli mi?
   - Volume mount edilmiş mi (restart'ta veri kaybolmasın)?

2. Topic tasarımı (transfer.completed.v1):
   - Naming convention domain-prefix + entity + verb-past + version mi?
   - Partition sayısı throughput'a uygun mu (1 değil)?
   - Replication factor production için 3 olarak NOT edilmiş mi?
   - min.insync.replicas konfigüre edilmiş mi?
   - Retention period banking için makul mu (7+ gün)?

3. Event schema (TransferCompletedEvent):
   - Record kullanılmış mı (immutable)?
   - transferId, fromAccountId, toAccountId UUID mi?
   - amount BigDecimal mi (double değil)?
   - currency String mi, ISO 4217 olduğu açıklanmış mı?
   - timestamp field'lar Instant mi, JSON'da ISO-8601 string olarak mı serialize ediliyor?
   - version field'ı var mı (schema evolution için)?
   - correlationId distributed tracing için var mı?
   - PII (TC no, kart no) event payload'unda VAR MI (olmamalı)?

4. Partition key:
   - Key olarak accountId kullanılması planlanmış mı?
   - "Hot account" probleminin farkında mıyım (notlarda var mı)?

5. Topic configs:
   - retention.ms açıkça set edilmiş mi?
   - compression.type (snappy/lz4/zstd) düşünülmüş mü?

6. Kavrayış kontrolü (defter notlarına bakarak):
   - Partition vs replica farkını kendi cümlemle anlatabiliyorum.
   - ISR ne demek söyleyebiliyorum.
   - Offset, LEO, lag arasındaki ilişkiyi biliyorum.
   - log compaction ne zaman kullanılır söyleyebiliyorum.

Her madde için PASS / FAIL / EKSIK işaretle. Sadece eksik olanları işaret et, 
düzeltme kodu yazma.
```

---

## Tamamlama kriterleri

- [ ] Docker compose ile Kafka cluster lokal çalışıyor
- [ ] Kafka UI'da cluster gözüküyor
- [ ] `transfer.completed.v1` topic'i oluşturuldu (4 partition)
- [ ] Console producer/consumer ile manuel mesaj gönderdim, aldım
- [ ] Aynı key'in aynı partition'a düştüğünü gözle gördüm
- [ ] Compacted topic ile deney yaptım, tombstone test ettim
- [ ] `events` modülü kuruldu, `TransferCompletedEvent` record yazıldı
- [ ] `TransferCompletedEventTest` JSON serialization test'leri geçti
- [ ] Partition vs replica farkını anlatabiliyorum
- [ ] ISR, leader, follower, controller terimlerini açıklayabiliyorum
- [ ] Banking için topic partition key seçim kriterlerini söyleyebiliyorum
- [ ] log compaction ne zaman kullanılır, ne zaman delete retention bilebiliyorum
- [ ] Production vs local broker konfig farklarını listeleyebiliyorum (RF, min.insync, vs)

---

## Defter notları

Aşağıdaki cümleleri kendi kelimelerinle doldur:

1. "Kafka mesaj kuyruğu değil, ____ olarak tanımlanmalı çünkü ____."
2. "Bir topic'in partition sayısını arttırmak ____ sağlar ama ____ riskini getirir."
3. "Partition key seçimi banking için kritik çünkü ____. Örnek: TransferCompleted için key = ____, sebep = ____."
4. "Replication factor 3 production'da standart çünkü ____. RF=1 yasak çünkü ____."
5. "ISR (In-Sync Replicas) şu kontrole hizmet eder: ____. min.insync.replicas=2 ne anlama gelir: ____."
6. "Offset, partition içinde mesajın ____. Consumer lag = ____."
7. "log compaction kullanım senaryosu: ____. Banking örneği: ____."
8. "ZooKeeper vs KRaft farkı: ____. TR bankalarında muhtemel durum: ____."
9. "TransferCompleted topic'i için seçtiğim partition sayısı: ____, sebep: ____."
10. "TransferCompleted event payload'unda PII tutmama sebebim: ____. Alternatif: ____."

→ Sıradaki topic: [02-producer/](../02-producer/index.md)
