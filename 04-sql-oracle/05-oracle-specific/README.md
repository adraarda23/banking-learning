# Topic 4.5 — Oracle-Specific Features

## Hedef

Oracle DB'nin **kendine özgü özelliklerini** banking-grade seviyede öğrenmek: sequences, partitioning (range/list/hash), materialized views, MVCC ve undo segments, ORA-01555 hatası, tablespaces, archivelog mode. Production banking Oracle ortamında DBA ile aynı dili konuşabilmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 45 dk • Toplam: ~4.5 saat

## Önbilgi

- Topic 4.1-4.4 bitti
- Oracle XE Docker'da çalışıyor
- PL/SQL temelleri biliniyor

---

## Kavramlar

### 1. Sequences — Oracle ID üretimi

```sql
CREATE SEQUENCE seq_account_id
    START WITH 1
    INCREMENT BY 1
    CACHE 50              -- 50'şer cache, sequence call'ları azalt
    NOORDER               -- RAC'te order garanti yok (perf için)
    NOCYCLE;              -- max'a ulaşınca dön mü?

-- Kullanım
INSERT INTO accounts(id, ...) VALUES (seq_account_id.NEXTVAL, ...);

-- Mevcut değeri okumadan
SELECT seq_account_id.NEXTVAL FROM dual;   -- artırır
SELECT seq_account_id.CURRVAL FROM dual;   -- current (aynı session'da NEXTVAL sonrası)
```

**`CACHE 50`:** Oracle 50 değeri memory'de tutar, her INSERT için disk roundtrip yok. **Production'da minimum 20-50.**

**`NOORDER`:** RAC (Real Application Clusters) ortamında node'lar arası sequence sıralama garantisi gerekmiyorsa. Default `ORDER` daha yavaş.

**`NOCYCLE`:** Banking için zorunlu. Cycle yaparsan ID collision olur.

#### Oracle 12c+: IDENTITY column

```sql
CREATE TABLE accounts(
    id NUMBER GENERATED ALWAYS AS IDENTITY 
       (START WITH 1 INCREMENT BY 1 CACHE 50),
    ...
);
```

Sequence'i implicit yönetir. PostgreSQL `SERIAL` benzeri.

#### `gen_random_uuid()` alternatif

UUID daha iyi distributed scenarios için. Oracle 12c+:

```sql
SELECT SYS_GUID() FROM dual;   -- 16-byte RAW UUID
```

PostgreSQL ile karşılaştırma — PostgreSQL `gen_random_uuid()` extension.

### 2. Partitioning — büyük tablolar için

Partitioning **bir mantıksal tabloyu fiziksel olarak birden fazla parçaya** böler. Her partition kendi I/O, kendi index'i olabilir.

**Faydalar:**
- Partition pruning: Sorgu sadece ilgili partition'ları okur
- Backup/archive: Eski partition'ları drop edebilir (saniyeler)
- Maintenance: Partition bazlı rebuild, gather stats
- Parallel processing: Partition başına paralel worker

**Banking örneği:** `transactions` tablosu — 10 yıl boyunca her gün yeni veri, milyarlarca satır. Partition'lasız yönetilemez.

#### RANGE partitioning — tarih için

```sql
CREATE TABLE transactions(
    id RAW(16),
    account_id RAW(16),
    amount NUMBER(19, 4),
    occurred_at TIMESTAMP WITH TIME ZONE,
    direction VARCHAR2(6)
)
PARTITION BY RANGE (occurred_at) (
    PARTITION p_2024_01 VALUES LESS THAN (TIMESTAMP '2024-02-01 00:00:00 UTC'),
    PARTITION p_2024_02 VALUES LESS THAN (TIMESTAMP '2024-03-01 00:00:00 UTC'),
    PARTITION p_2024_03 VALUES LESS THAN (TIMESTAMP '2024-04-01 00:00:00 UTC'),
    -- ...
    PARTITION p_max VALUES LESS THAN (MAXVALUE)
);
```

Sorgu:

```sql
SELECT * FROM transactions WHERE occurred_at BETWEEN '2024-02-01' AND '2024-02-28';
-- Oracle SADECE p_2024_02'yi okur (partition pruning)
```

**Interval partitioning** (Oracle 11g+) — otomatik partition yarat:

```sql
CREATE TABLE transactions(...)
PARTITION BY RANGE (occurred_at)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))   -- her ay yeni partition
(PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00 UTC'));
```

Yeni transaction insert edilince Oracle otomatik partition yaratır. **Banking standardı.**

#### LIST partitioning — kategori için

```sql
CREATE TABLE accounts(...)
PARTITION BY LIST (currency) (
    PARTITION p_try VALUES ('TRY'),
    PARTITION p_usd VALUES ('USD'),
    PARTITION p_eur VALUES ('EUR'),
    PARTITION p_other VALUES (DEFAULT)
);
```

Currency'e göre fiziksel ayrım. Sadece TRY transaction'ları arıyorsan p_try'i okur.

#### HASH partitioning — dağıtım için

```sql
CREATE TABLE journal_lines(...)
PARTITION BY HASH (account_id) PARTITIONS 16;
```

`account_id`'in hash'ine göre 16 partition'a dağıt. Eşit dağılım — paralel I/O.

**Banking:** Hot spot (popüler account'lar) varsa hash partitioning dağıtır.

#### Composite partitioning — RANGE + HASH

```sql
CREATE TABLE transactions(...)
PARTITION BY RANGE (occurred_at)
SUBPARTITION BY HASH (account_id) SUBPARTITIONS 8
(PARTITION p_2024_01 VALUES LESS THAN (TIMESTAMP '2024-02-01 UTC'),
 PARTITION p_2024_02 VALUES LESS THAN (TIMESTAMP '2024-03-01 UTC'));
```

Her aylık partition içinde 8 hash subpartition. **Time-based query + account-based dağıtım = ideal banking.**

### 3. Partition operations

```sql
-- Eski partition'ı drop et (archive sonrası)
ALTER TABLE transactions DROP PARTITION p_2020_01;   -- saniyeler, milyonlarca satır gider

-- Partition'ı boşalt
ALTER TABLE transactions TRUNCATE PARTITION p_2020_01;

-- Yeni partition ekle (interval olmayan tablolar için)
ALTER TABLE transactions ADD PARTITION p_2024_04 VALUES LESS THAN (TIMESTAMP '2024-05-01 UTC');

-- Partition başına stats topla
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'TRANSACTIONS', PARTNAME => 'P_2024_02');

-- Sadece bir partition'ı sorgula
SELECT * FROM transactions PARTITION (p_2024_02);
```

**Banking pattern:** Aylık rolling archive — 24 aydan eski partition'ları S3'e export et, sonra drop.

### 4. Materialized Views — pre-computed aggregate

Bir view normal sorgu shortcut'ıdır — her çağrıda yeniden hesaplar. **Materialized view** sonucu **disk'te** tutar.

```sql
CREATE MATERIALIZED VIEW mv_daily_summary
BUILD IMMEDIATE
REFRESH FAST ON DEMAND
ENABLE QUERY REWRITE
AS
SELECT TRUNC(occurred_at) AS business_date,
       account_id,
       COUNT(*) AS tx_count,
       SUM(amount) AS total_amount
FROM transactions
GROUP BY TRUNC(occurred_at), account_id;
```

**Parameter'lar:**

- `BUILD IMMEDIATE`: Yarat hemen doldur. Alternatif: `DEFERRED` (yaratır, doldurmaz)
- `REFRESH FAST ON DEMAND`: Manuel refresh, sadece değişen satırlar (materialized view log gerekir)
- `REFRESH COMPLETE`: Tüm view'i yeniden hesapla
- `REFRESH ON COMMIT`: Underlying tabloya her commit'te güncellenir (yavaşlatır)
- `ENABLE QUERY REWRITE`: Oracle uygun query'leri otomatik mv'ye yönlendirsin

**Refresh:**

```sql
EXEC DBMS_MVIEW.REFRESH('mv_daily_summary', 'F');   -- F = fast
EXEC DBMS_MVIEW.REFRESH('mv_daily_summary', 'C');   -- C = complete
```

**Banking örneği:** Daily/monthly reporting. Bir banka ekran açıyor "müşterinin son 12 ayının özeti" — her seferinde 10M satır aggregate yapmak yerine materialized view'den oku.

### 5. Materialized view log — fast refresh için

```sql
CREATE MATERIALIZED VIEW LOG ON transactions
WITH ROWID, SEQUENCE (account_id, amount, occurred_at)
INCLUDING NEW VALUES;
```

Underlying tablonun her değişikliği log'a yazılır. MV refresh sadece **log'daki değişiklikleri** uygular. **Sıfırdan yeniden hesaplamaz.**

**Tehlike:** Çok sık yazılan tablolarda log büyük olur, disk yer. İzlemek gerek.

### 6. Oracle MVCC — undo segments

PostgreSQL'in MVCC'si row-versioning ile (her satırın geçmişi). **Oracle MVCC undo segment'lerini kullanır.**

**Nasıl çalışır:**

1. UPDATE yaparsın → Oracle eski değeri **undo segment**'e yazar
2. Başka transaction senin update'ini commit'ten önce sorguluyorsa → undo segment'ten eski değeri okur ("read consistency")
3. Commit → undo segment'in o kısmı serbest kalır

**Faydalar:**
- Reader'lar writer'ları **bloklamaz**, writer'lar reader'ları **bloklamaz**
- Time travel (`AS OF TIMESTAMP`)

```sql
SELECT * FROM accounts AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR);
-- 1 saat önceki snapshot
```

### 7. ORA-01555 — "snapshot too old"

**Klasik banking incident.** Uzun süren bir query başlar, eski satır versiyonu lazım, ama undo segment **overwrite** olmuş.

```
ORA-01555: snapshot too old: rollback segment number XX with name "..." too small
```

**Sebepler:**
1. Uzun süren query (saatlerce çalışan rapor)
2. Aktif yazma yükü (undo segment hızla doluyor)
3. UNDO_RETENTION yetmiyor

**Önlem:**

```sql
-- Production tipik:
ALTER SYSTEM SET UNDO_RETENTION = 21600;   -- 6 saat
```

`UNDO_RETENTION` retentionsız değildir — sadece **hint**. Yer yoksa Oracle ihlal eder.

**Banking pratiği:** Long-running query'leri (raporlama) **read-only replica**'da çalıştır. Production transactional DB'de uzun query'den kaçın.

### 8. Tablespaces

Tablespace = veri dosyalarının mantıksal grubu. Tablolar tablespace'lerde yaşar.

```sql
CREATE TABLESPACE banking_data 
    DATAFILE '/u01/oradata/banking.dbf' SIZE 10G
    AUTOEXTEND ON NEXT 1G MAXSIZE 100G;

CREATE TABLE accounts(...) TABLESPACE banking_data;
```

**Banking pratiği:**
- Hot data (transactions) → fast SSD tablespace
- Cold data (archive) → slower disk
- Index → ayrı tablespace (I/O paralelize)
- Temp → ayrı (sort/aggregate)

### 9. Redo log ve archivelog

**Redo log:** Her DML değişikliği önce **redo log**'a yazılır (sıralı, çok hızlı), sonra data file'a (random I/O).

```
COMMIT → log buffer flush → redo log write → ACK to caller
```

**Crash recovery:** Server çakılırsa, restart'ta redo log'dan committed transaction'lar replay edilir.

**Archivelog mode:** Eski redo log'ları silinmeden arşivlenir. **Point-in-time recovery** için gerekli.

```sql
SELECT log_mode FROM v$database;   -- ARCHIVELOG veya NOARCHIVELOG
```

**Banking pratiği:** Production'da **mutlaka ARCHIVELOG mode**. "Saat 10:23'teki yedek + 10:23-12:45 redo log'ları → 12:45'e geri dön."

### 10. ROWID & ROWNUM

**ROWID:** Bir satırın **fiziksel adresi** (datafile, block, row). Sabit, biricik (satır taşınana kadar).

```sql
SELECT ROWID, id FROM accounts WHERE owner_id = '...';
-- ROWID: AAAVAVAAEAAAB7CAAA
```

Banking kullanımı: Aynı satırı tekrar fetch etmek için en hızlı yol.

**ROWNUM:** Sorgu sonucundaki **sıra numarası** (1, 2, 3, ...).

```sql
SELECT * FROM accounts WHERE ROWNUM <= 10;   -- ilk 10
```

**Tuzak:** ROWNUM `WHERE` sonrasında, ORDER BY öncesinde değerlendirilir.

```sql
-- ❌ Yanlış: random 10 satır, sorted değil
SELECT * FROM accounts WHERE ROWNUM <= 10 ORDER BY balance_amount DESC;

-- ✓ Doğru: top 10
SELECT * FROM (
    SELECT * FROM accounts ORDER BY balance_amount DESC
) WHERE ROWNUM <= 10;

-- veya Oracle 12c+
SELECT * FROM accounts ORDER BY balance_amount DESC FETCH FIRST 10 ROWS ONLY;
```

### 11. Optimizer hints (kısa)

```sql
SELECT /*+ INDEX(a idx_acc_owner) */ * FROM accounts a WHERE owner_id = '...';

SELECT /*+ PARALLEL(t, 8) */ * FROM transactions t WHERE occurred_at > SYSDATE - 7;

SELECT /*+ NO_INDEX(a) */ * FROM accounts a WHERE ...;

SELECT /*+ FIRST_ROWS(10) */ ...;   -- ilk 10 satır için optimize
```

Topic 4.2'de detaylandırıldı. Production'da **son çare**.

### 12. Oracle text data types

| Type | Açıklama |
|---|---|
| CHAR(n) | Fixed-length, padded with spaces |
| VARCHAR2(n) | Variable-length (banking standard) |
| NVARCHAR2(n) | Unicode variable-length |
| CLOB | Character large object (>4000 bytes) |
| BLOB | Binary large object |
| RAW(n) | Binary (UUID için RAW(16)) |
| NUMBER(p, s) | Precision + scale (banking money: NUMBER(19, 4)) |
| DATE | Date + time, second precision |
| TIMESTAMP | Microsecond precision |
| TIMESTAMP WITH TIME ZONE | TZ aware |

**Banking pratiği:**
- Money: `NUMBER(19, 4)` (PostgreSQL `NUMERIC(19, 4)` ile aynı)
- Date'ler: `TIMESTAMP WITH TIME ZONE` (multi-region banking)
- ID: `RAW(16)` (UUID için) veya `NUMBER` (sequence ile)

### 13. AWR ve ASH report (DBA dünyası, junior'a tanıtım)

**AWR (Automatic Workload Repository):** Her saat snapshot alır. Performans analizi için altın madeni.

```sql
-- AWR snapshot al
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();

-- AWR rapor üret (DBA permissions gerekir)
@?/rdbms/admin/awrrpt.sql
```

**ASH (Active Session History):** Saniye-bazlı session aktivitesi.

```sql
SELECT * FROM v$active_session_history WHERE sample_time > SYSDATE - 1/24;
```

**Banking gerçeği:** AWR/ASH'i DBA çalıştırır. Sen ürettiği raporu okuyabilmen yeterli. Senior dev'lik için tanıman iyi.

### 14. Banking örnekleri — Production scenarios

#### Senaryo 1: Yıllar boyu büyüyen transactions

```sql
CREATE TABLE transactions(
    id RAW(16) DEFAULT SYS_GUID(),
    account_id RAW(16) NOT NULL,
    amount NUMBER(19, 4),
    direction VARCHAR2(6),
    occurred_at TIMESTAMP WITH TIME ZONE NOT NULL
)
PARTITION BY RANGE (occurred_at)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'));

-- Index local olarak (her partition kendi index'i)
CREATE INDEX idx_tx_account ON transactions(account_id) LOCAL;
```

Otomatik aylık partition. 24 aydan eski partition'lar archive job ile S3'e + drop.

#### Senaryo 2: Daily reporting MV

```sql
CREATE MATERIALIZED VIEW mv_daily_balance_summary
REFRESH FORCE ON DEMAND
AS
SELECT 
    TRUNC(je.occurred_at) AS business_date,
    a.currency,
    COUNT(*) AS tx_count,
    SUM(CASE WHEN jl.direction = 'CREDIT' THEN jl.amount ELSE 0 END) AS total_credit,
    SUM(CASE WHEN jl.direction = 'DEBIT' THEN jl.amount ELSE 0 END) AS total_debit
FROM journal_entries je
JOIN journal_lines jl ON jl.journal_entry_id = je.id
JOIN accounts a ON a.id = jl.account_id
GROUP BY TRUNC(je.occurred_at), a.currency;
```

EOD job sonrası refresh. Reporting endpoint'i mv'den okur.

### 15. Anti-pattern'ler

**Anti-pattern 1: Partitioning'siz büyük tablo**

Banking transaction'ları aylık partition'lasız, tek tabloda saklamak. Bir gün sonra `archive` veya `drop` imkânsız.

**Anti-pattern 2: ROWNUM ile pagination**

```sql
-- ❌ Yanlış pagination
SELECT * FROM accounts WHERE ROWNUM > 100 AND ROWNUM <= 200;
```

ROWNUM `WHERE` ile incremental atanır, `>` kontrolü hep false. **Çözüm:** subquery veya Oracle 12c+ `OFFSET ... FETCH`.

```sql
SELECT * FROM accounts ORDER BY id 
OFFSET 100 ROWS FETCH NEXT 100 ROWS ONLY;
```

**Anti-pattern 3: Materialized view refresh on commit ağır query**

Her commit'te refresh = production yavaşlar. **Çözüm:** Refresh on demand + scheduled job.

**Anti-pattern 4: NOARCHIVELOG production**

Sıfır recovery imkânı. **Mutlaka ARCHIVELOG.**

**Anti-pattern 5: Sequence CACHE 0 veya çok küçük**

```sql
CREATE SEQUENCE seq_x CACHE 1;   -- her INSERT'te disk
```

High-volume INSERT'te bottleneck. CACHE 20-100 banking standardı.

---

## Önemli olabilecek araştırma kaynakları

- Oracle Database Concepts (official docs)
- Oracle Partitioning Guide
- "Expert Oracle Database Architecture" (Tom Kyte)
- Oracle Materialized View tutorial
- Oracle Performance Tuning Guide
- Tom Kyte (asktom.oracle.com)
- "Oracle 19c New Features"
- AWR/ASH report interpretation guides

---

## Mini task'ler

### Task 4.5.1 — Sequence (15 dk)

`seq_account_id` yarat `CACHE 50`. Bir INSERT yaparken `NEXTVAL` kullan, `CURRVAL` ile son ID'i al.

### Task 4.5.2 — RANGE partitioning (30 dk)

`transactions` tablosunu aylık `INTERVAL` ile partition'la. 3 ay verisi insert et. Partition'ları sorgula:

```sql
SELECT partition_name, num_rows FROM user_tab_partitions WHERE table_name = 'TRANSACTIONS';
```

### Task 4.5.3 — Partition pruning gör (15 dk)

```sql
EXPLAIN PLAN FOR
SELECT * FROM transactions WHERE occurred_at BETWEEN '2024-02-01' AND '2024-02-28';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

Plan'da "PARTITION RANGE SINGLE" veya benzeri pruning evidence görmeli.

### Task 4.5.4 — Materialized view (45 dk)

`mv_daily_balance_summary`'i yarat. Initial data load. 10 yeni transaction insert et. `REFRESH FAST` ile güncelle (MV LOG gerekir).

### Task 4.5.5 — ORA-01555 reproduction (30 dk)

`UNDO_RETENTION` çok düşük tut (60 saniye). Bir cursor aç (`OPEN c FOR SELECT * FROM accounts`), 2 dakika bekle, fetch'le. ORA-01555 görmen lazım.

### Task 4.5.6 — Time travel query (15 dk)

```sql
UPDATE accounts SET balance_amount = 0 WHERE id = '...';
COMMIT;

-- 1 dakika sonra
SELECT balance_amount FROM accounts AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' MINUTE) WHERE id = '...';
```

Eski değeri okur.

### Task 4.5.7 — Drop old partition (15 dk)

```sql
ALTER TABLE transactions DROP PARTITION p_2020_01;
```

Saniyeler içinde milyonlarca satır gider. Banking'de aylık archive job mantığı.

---

## Test yazma rehberi

### Test 4.5.1 — Partition pruning verification

```java
@Test
@Sql("/sql/transactions-3-months.sql")
void monthRangeQueryShouldPruneToSinglePartition() {
    String plan = jdbc.queryForObject("""
        SELECT operation_type FROM (
            EXPLAIN PLAN FOR
            SELECT * FROM transactions 
            WHERE occurred_at BETWEEN ? AND ?
        )
        """, String.class, ...);
    
    // Plan'da partition pruning evidence
    assertThat(plan).contains("PARTITION RANGE");
}
```

### Test 4.5.2 — Materialized view refresh

```java
@Test
void mvShouldUpdateOnRefresh() {
    // Insert new transaction
    jdbc.update("INSERT INTO transactions ...");
    
    // MV stale, refresh et
    jdbc.execute("BEGIN DBMS_MVIEW.REFRESH('mv_daily_balance_summary', 'F'); END;");
    
    // MV'den oku
    BigDecimal todayTotal = jdbc.queryForObject(
        "SELECT total_credit FROM mv_daily_balance_summary WHERE business_date = TRUNC(SYSDATE)",
        BigDecimal.class
    );
    
    assertThat(todayTotal).isPositive();
}
```

---

## Claude-verify prompt

```
Oracle-specific çalışmamı banking-grade kriterlere göre değerlendir:

1. Sequence:
   - CACHE değeri 1'den büyük mü (20-100 banking standardı)?
   - NOCYCLE banking için zorunlu mu?
   - IDENTITY column kullanılmış mı (Oracle 12c+)?

2. Partitioning:
   - Büyük tablolarda partitioning var mı (transactions)?
   - INTERVAL partitioning ile otomatik partition oluşturma mı?
   - Local index'ler partition'lı tablolarda mı?
   - Eski partition drop akışı (archive) net mi?

3. Materialized view:
   - Reporting için MV kullanılmış mı?
   - Refresh strategy (ON DEMAND vs ON COMMIT) bilinçli mi?
   - Materialized view log fast refresh için yaratılmış mı?

4. MVCC ve undo:
   - UNDO_RETENTION production'da yeterli mi (6+ saat)?
   - Long-running query'ler read-only replica'ya yönlendirilmiş mi?
   - AS OF TIMESTAMP ile time travel deneyimi var mı?

5. Tablespaces:
   - Hot/cold data ayrımı yapılmış mı?
   - Index'ler ayrı tablespace'te mi?

6. Archive log:
   - Production'da ARCHIVELOG mode mu?
   - Point-in-time recovery strateji?

7. ROWNUM/ROWID:
   - ROWNUM ile yanlış pagination var mı? (Olmamalı)
   - Oracle 12c+ FETCH FIRST kullanılmış mı?

8. Anti-pattern:
   - Partitioning'siz büyük tablo?
   - NOARCHIVELOG production?
   - CACHE 0 sequence?
   - MV refresh ON COMMIT yüksek-volume tabloda?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Sequence yarattım `CACHE 50` ile
- [ ] RANGE INTERVAL partitioning ile transactions tablosu
- [ ] Partition pruning'i EXPLAIN PLAN ile gözlemledim
- [ ] Materialized view yarattım + manuel refresh denedim
- [ ] ORA-01555 hatasını **canlı** reproduction yaptım
- [ ] `AS OF TIMESTAMP` ile time travel sorgu denedim
- [ ] DROP PARTITION ile bir aylık veriyi sildim (saniyeler)
- [ ] ROWNUM yanlış pagination tuzağını biliyorum
- [ ] FETCH FIRST ... ROWS ONLY pattern'ini kullanıyorum
- [ ] ARCHIVELOG mode'un production zorunluluğunu anlıyorum

---

## Defter notları

1. "Sequence CACHE değerinin performans etkisi: ____."
2. "RANGE INTERVAL partitioning'in banking'de neden standart: ____."
3. "Partition pruning'i EXPLAIN PLAN'de nasıl tanırım: ____."
4. "Materialized view fast refresh ile complete refresh farkı: ____."
5. "ORA-01555 hatasının kök nedeni (undo + uzun query): ____."
6. "UNDO_RETENTION ne işe yarar, banking için ideal değer: ____."
7. "AS OF TIMESTAMP time travel kullanım senaryosu: ____."
8. "ARCHIVELOG mode olmadan recovery imkânı: ____."
9. "ROWNUM `WHERE ROWNUM > 100` neden çalışmaz: ____."
10. "Oracle 12c+ OFFSET ... FETCH alternatifi neden tercih: ____."
