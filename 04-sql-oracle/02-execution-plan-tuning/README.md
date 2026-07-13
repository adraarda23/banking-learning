# Topic 4.2 — Execution Plan & Query Tuning

## Hedef

Bir SQL query'nin DB tarafından **nasıl çalıştırıldığını** okuyabilmek. EXPLAIN PLAN'ı (Oracle), EXPLAIN ANALYZE'ı (PostgreSQL) anlamak. Join algorithm'larını (nested loop, hash join, sort-merge) tanımak. CBO (cost-based optimizer)'un kararlarını sorgulamak ve gerektiğinde **hint** ile yönlendirmek. Banking'de yavaş query'leri 5 dakikada teşhis edebilen developer olmak.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 45 dk • Toplam: ~4.5 saat

## Önbilgi

- Topic 4.1 (Index Internals) bitti
- Temel SQL (SELECT, JOIN, WHERE, GROUP BY) rahat
- PostgreSQL veya Oracle ile pratik var

---

## Kavramlar

### 1. Execution plan nedir, neden okurum

Bir SQL query DB'ye verdiğinde:
1. **Parser** syntax kontrolü yapar
2. **Optimizer** birden fazla execution path düşünür
3. **Cost estimation** her path için maliyet hesabı (I/O, CPU, memory)
4. En düşük maliyetli plan **seçilir**
5. **Executor** planı çalıştırır

**Execution plan = optimizer'ın kararı.** Plan'ı görmek = "DB neyi neden yapıyor anlamak".

Banking örneği — yavaş query:

```sql
SELECT * FROM accounts WHERE owner_id = '...' AND status = 'ACTIVE';
```

Plan A: `owner_id` index'ini kullan, sonra `status` filtrele → 100ms
Plan B: Full table scan + filter ikisi de → 5 saniye

Optimizer yanlış seçim yaparsa: query yavaş kalır. Plan'ı okumak senin işin.

### 2. EXPLAIN — PostgreSQL

```sql
EXPLAIN SELECT * FROM accounts WHERE owner_id = '...';
```

Output:

```
Index Scan using idx_accounts_owner_id on accounts  (cost=0.42..8.44 rows=1 width=72)
  Index Cond: (owner_id = '...'::uuid)
```

- `Index Scan` — index kullanılıyor (iyi)
- `cost=0.42..8.44` — startup cost ve total cost (DB-internal birim, küçük > büyük)
- `rows=1` — tahmin: 1 satır dönecek
- `width=72` — her satır 72 byte (yaklaşık)

### 3. EXPLAIN ANALYZE — gerçek çalıştır + ölçüm

```sql
EXPLAIN ANALYZE SELECT * FROM accounts WHERE owner_id = '...';
```

```
Index Scan using idx_accounts_owner_id on accounts  
  (cost=0.42..8.44 rows=1 width=72) 
  (actual time=0.025..0.026 rows=1 loops=1)
  Index Cond: (owner_id = '...'::uuid)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```

- `actual time=0.025..0.026` — gerçek startup ve total süre (ms)
- `rows=1 loops=1` — gerçek satır sayısı

**Önemli karşılaştırma:** Cost-estimated `rows=1` vs actual `rows=1` — uyuyor mu? Sapma varsa stats güncellenmemiş.

**Sorunlu örnek:**

```
Seq Scan on accounts  (cost=0.00..18.24 rows=100 width=72) 
  (actual time=0.012..1234.567 rows=1000000 loops=1)
```

Cost 100 satır tahmin etti, gerçekte 1M satır geldi. Stats çok yanlış → ANALYZE çalıştır.

### 4. EXPLAIN options

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON) SELECT ...
```

- `ANALYZE` — gerçek çalıştır
- `BUFFERS` — buffer hit/miss
- `VERBOSE` — kolonları detayla
- `FORMAT JSON|YAML|XML` — yapısal output

```
Buffers: shared hit=10 read=5
```

`hit` cache'ten geldi, `read` disk'ten okundu. Disk read pahalı.

### 5. Oracle EXPLAIN PLAN

```sql
EXPLAIN PLAN FOR
SELECT * FROM accounts WHERE owner_id = '...';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

Output:

```
--------------------------------------------------------------------------------
| Id  | Operation                   | Name              | Rows  | Cost  |
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                   |     1 |     2 |
|*  1 |  INDEX RANGE SCAN           | IDX_ACC_OWNER_ID  |     1 |     2 |
--------------------------------------------------------------------------------
```

`AUTOTRACE` ile bir adım daha:

```sql
SET AUTOTRACE ON
SELECT * FROM accounts WHERE owner_id = '...';
```

Hem sorgu çalışır hem plan + stats görünür.

### 6. Join algorithm'ları

#### a) Nested Loop Join

```
for each row R in outer table:
    for each row S in inner table where condition:
        output (R, S)
```

- **Outer küçük**, **inner index'li** olmalı
- O(outer × log(inner)) — outer küçükse hızlı
- Banking: account → owner JOIN, hesap sayısı az ise nested loop iyi

```
Nested Loop  (cost=0.42..16.46 rows=1 width=144)
  ->  Index Scan using idx_accounts_owner_id on accounts a
        Index Cond: (owner_id = '...'::uuid)
  ->  Index Scan using owners_pkey on owners o
        Index Cond: (id = a.owner_id)
```

#### b) Hash Join

```
1. Build phase: inner table'ın hash table'ını yap (memory)
2. Probe phase: outer table'ın her satırı için hash'e bak
```

- **Büyük tablolar için iyi**
- Memory yeterli olmalı (yoksa diske yazar, yavaşlar)
- Equi-join (`=` koşulu) gerekli
- Banking: 1M transaction × 100k account JOIN — hash join uygun

```
Hash Join  (cost=27.50..68.42 rows=1000 width=144)
  Hash Cond: (t.account_id = a.id)
  ->  Seq Scan on transactions t  (cost=...)
  ->  Hash  (cost=...)
        ->  Seq Scan on accounts a  (cost=...)
```

#### c) Sort-Merge Join

```
1. Sort both tables on join column
2. Merge sorted tables, output matches
```

- **Çok büyük tablolar**, indexlerine bağlı sorted
- Memory için iyi
- Range join veya zaten sorted input

```
Merge Join  (cost=...)
  Merge Cond: (a.id = t.account_id)
  ->  Index Scan using accounts_pkey on accounts a
  ->  Sort
        ->  Seq Scan on transactions t
```

#### Banking için seçim

| Senaryo | Optimal Join |
|---|---|
| Küçük outer + büyük inner index | Nested Loop |
| İki büyük table, eşitlik koşulu | Hash Join |
| İki büyük table, range koşulu veya sorted | Sort-Merge |
| Outer tablonun çok küçük olduğu durumlar | Nested Loop |

CBO genelde doğru seçim yapar **ama** stats güncel değilse veya complex query'lerde yanlış seçer.

### 7. Index access methods

Plan'da görebileceğin index erişim tipleri:

**`Index Scan`:** Index üzerinden git → matched leaf'lerden table row'a (heap). Çoğu zaman bunu görürsün.

**`Index Only Scan` (PostgreSQL) / `INDEX FAST FULL SCAN` (Oracle):** Sorgu sadece index'teki kolonları istiyor → heap'e gitmeden cevap (covering index).

```sql
CREATE INDEX idx_acc_owner_curr ON accounts(owner_id, currency);

SELECT currency FROM accounts WHERE owner_id = '...';
-- Plan: Index Only Scan — heap'e gitmedi, hızlı
```

**`Bitmap Index Scan`:** Birden fazla index'ten OR/AND birleştir.

```sql
SELECT * FROM transactions WHERE status = 'PENDING' OR source = 'BATCH';
```

**`Seq Scan` (PostgreSQL) / `TABLE ACCESS FULL` (Oracle):** Full table scan. Küçük tablolarda OK, büyüklerde fatal.

### 8. CBO ve statistics

Cost-Based Optimizer her plan için **maliyet hesaplar**. Hesap için stats kullanır:

- Tablo satır sayısı
- Her kolonun unique value sayısı (NDV)
- Histogram (kolon değer dağılımı)
- Index istatistiği

```sql
-- PostgreSQL
ANALYZE accounts;
ANALYZE accounts(owner_id, status);   -- spesifik kolon

-- Oracle
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'ACCOUNTS');
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'ACCOUNTS', cascade => TRUE);  -- + index stats
```

**Banking pratiği:** Production'da her gece (EOD sonrası) ANALYZE çalıştır. Stats güncel olmazsa CBO yanlış plan seçer.

### 9. Histograms

Eğer bir kolonun değer dağılımı **eşit değilse** (skewed), histogram olmadan optimizer yanlış varsayım yapar:

```
status = 'CLOSED' → %1
status = 'ACTIVE' → %95
status = 'FROZEN' → %4
```

CBO histogram olmadan "eşit dağılım" varsayar, "her status'ten %33" hesabı yapar → yanlış plan.

```sql
-- PostgreSQL — automatic histogram
ANALYZE accounts;   -- otomatik kolonların histogram'ı

-- Manual: HISTOGRAM_BOUNDS column statistics
SELECT * FROM pg_stats WHERE tablename = 'accounts' AND attname = 'status';
```

```sql
-- Oracle — histogram explicit
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'ACCOUNTS', 
     method_opt => 'FOR COLUMNS SIZE 254 status');
```

### 10. Bind variable peeking & cursor sharing (Oracle)

```sql
SELECT * FROM accounts WHERE owner_id = :owner_id;
```

Oracle ilk çağrıda `:owner_id`'nin **gerçek değerini peek** eder ve plana karar verir. Sonraki çağrılarda **aynı plan**.

**Tuzak:** İlk çağrıda value rare (1 satır beklenir) → nested loop. Sonraki çağrılarda value common (1M satır) → nested loop **felaket** yavaş.

**Çözüm:** Oracle 11g+ **Adaptive Cursor Sharing** — runtime'da farklı plan seçer. Modern Oracle'da otomatik.

PostgreSQL benzer durum: `prepared statement` ilk planı cache'ler. `plan_cache_mode = force_custom_plan` veya `force_generic_plan` ile davranış değiştirebilirsin.

### 11. Query hints

CBO'yu **zorla** yönlendirmek için (genelde son çare).

#### Oracle

```sql
SELECT /*+ INDEX(a idx_acc_owner_id) */ *
FROM accounts a
WHERE owner_id = '...';
```

```sql
SELECT /*+ USE_HASH(a t) */ ...
FROM accounts a JOIN transactions t ON a.id = t.account_id;
```

```sql
SELECT /*+ NO_INDEX(a) */ ...   -- index kullanma, full scan yap
```

#### PostgreSQL

PostgreSQL hint **yok** (resmi). Alternatifler:
- `enable_seqscan = off` (session level)
- `pg_hint_plan` extension (Oracle benzeri)

**Banking pratiği:** Hint'leri **son çare** olarak kullan. Önce stats güncelle, query rewrite et, gerekirse index ekle. Hint kalıcı bir bağımlılık yaratır — DB versiyonu değişince patlayabilir.

### 12. Window function execution

```sql
SELECT id, balance,
       SUM(balance) OVER (PARTITION BY owner_id ORDER BY opened_at) AS running_balance
FROM accounts;
```

Plan'da:

```
WindowAgg  (cost=...)
  Sort Key: owner_id, opened_at
  ->  Sort
        ->  Seq Scan on accounts
```

Window function **sort gerektirir** (zaten sorted index yoksa). Banking'de transaction history reporting'de yaygın. Topic 4.3'te detay.

### 13. Banking örnekleri — yavaş query analizi

#### Vaka 1: Transaction history yavaş

```sql
SELECT * FROM transactions 
WHERE account_id = '...' AND occurred_at > '2024-01-01'
ORDER BY occurred_at DESC LIMIT 50;
```

**Yavaş plan:**

```
Sort  (cost=10000)
  ->  Seq Scan on transactions
        Filter: account_id = '...' AND occurred_at > '2024-01-01'
```

Full table scan + sort. 10M satır için katastrof.

**Çözüm:** Composite index

```sql
CREATE INDEX idx_tx_acc_time ON transactions(account_id, occurred_at DESC);
```

Yeni plan:

```
Limit (cost=0.42..50.42 rows=50)
  ->  Index Scan using idx_tx_acc_time on transactions
        Index Cond: account_id = '...' AND occurred_at > '2024-01-01'
```

Index sorted DESC, LIMIT 50 hemen alır. **1000x hızlanma.**

#### Vaka 2: Owner-balance reporting yavaş

```sql
SELECT o.name, SUM(a.balance_amount) total
FROM owners o
JOIN accounts a ON a.owner_id = o.id
WHERE a.status = 'ACTIVE'
GROUP BY o.id, o.name;
```

**Yavaş plan:**

```
HashAggregate (cost=...)
  ->  Hash Join
        ->  Seq Scan on owners
        ->  Seq Scan on accounts
              Filter: status = 'ACTIVE'
```

Full scan accounts (10M row). Eğer 90%'ı ACTIVE ise filter yardımcı olmuyor.

**Çözümler:**
1. ANALYZE ve histogram (eğer status skewed ise CBO daha iyi karar verecek)
2. Composite index `(status, owner_id)` — sadece ACTIVE'leri okur
3. Materialized view ile pre-aggregate (Topic 4.5)

### 14. Plan stability — production tuzakları

Production'da plan **aniden** değişebilir:
- Statistics değişti (ANALYZE çalıştı)
- Data dağılımı değişti
- Index eklendi/silindi
- Oracle/Postgres versiyonu upgrade

Sabah çalışan query öğleden sonra yavaşlar. Sebep: plan değişimi.

**Önlem:**
- Critical query'lerin plan'ını **monitor et**
- Plan değişimini alert'le yakala
- Oracle: SQL Plan Baseline ile sabitle (advanced)
- PostgreSQL: `pg_stat_statements` extension

### 15. Banking anti-pattern'leri

**Anti-pattern 1: `SELECT *`**

```sql
SELECT * FROM accounts WHERE ...
```

Tüm kolonları çek → daha fazla I/O, network. Covering index avantajını kaybedebilirsin.

**Çözüm:** Sadece gerekli kolonları yaz.

**Anti-pattern 2: Function on indexed column**

```sql
SELECT * FROM accounts WHERE UPPER(currency) = 'TRY';
-- ❌ idx_accounts_currency kullanılmaz (function index'i yok)
```

Çözüm: ya function-based index, ya da query'i değiştir:

```sql
SELECT * FROM accounts WHERE currency = 'TRY';
-- ✓ doğrudan index
```

**Anti-pattern 3: `OR` ile farklı kolonlar**

```sql
SELECT * FROM accounts WHERE owner_id = '...' OR customer_reference = '...';
```

İki ayrı index var ama `OR` ikisinden de yararlanma yöntemi kısıtlı. UNION daha iyi:

```sql
SELECT * FROM accounts WHERE owner_id = '...'
UNION
SELECT * FROM accounts WHERE customer_reference = '...';
```

**Anti-pattern 4: Implicit type cast**

```sql
SELECT * FROM accounts WHERE id = 123;
-- id UUID kolonu, 123 INTEGER → implicit cast, index kullanılamayabilir
```

**Çözüm:** Doğru tip ile sorgula.

**Anti-pattern 5: `LIKE '%xyz'`**

```sql
SELECT * FROM owners WHERE name LIKE '%doe';
```

Wildcard başta → B-tree index işe yaramaz. Trigram index (`pg_trgm`) veya full-text search gerekir.

**Anti-pattern 6: Statistics güncellenmiyor**

EOD sonrası ANALYZE çalıştırmıyorsan optimizer yanlış kararlar verir. Cron job ile düzenli ANALYZE.

---

## Önemli olabilecek araştırma kaynakları

- "SQL Performance Explained" (Markus Winand) — kısa, paha biçilmez
- "Use The Index, Luke" (Markus Winand — ücretsiz online kitap)
- PostgreSQL official docs — EXPLAIN section
- Oracle Database Performance Tuning Guide
- "Troubleshooting Oracle Performance" (Christian Antognini)
- Tom Kyte (asktom.oracle.com) — Oracle tuning altın kaynak
- "PostgreSQL High Performance" (Gregory Smith)
- "Database Performance Tuning Guide" (PostgreSQL wiki)

---

## Mini task'ler

### Task 4.2.1 — Basit query EXPLAIN (30 dk)

`core-banking` DB'de:

```sql
EXPLAIN ANALYZE SELECT * FROM accounts WHERE owner_id = '...';
EXPLAIN ANALYZE SELECT * FROM accounts WHERE status = 'ACTIVE';
EXPLAIN ANALYZE SELECT * FROM accounts WHERE balance_amount > 1000;
```

Her sonucu defter'e yapıştır, plan'ı oku.

### Task 4.2.2 — Index karşılaştırması (45 dk)

10000 random account insert et. Üç senaryo:

1. Index'siz: `SELECT * FROM accounts WHERE owner_id = ?` — ne kadar sürüyor?
2. Index ekle: `CREATE INDEX idx_owner ON accounts(owner_id);` — tekrar dene
3. Composite index: `CREATE INDEX idx_owner_status ON accounts(owner_id, status);` — `SELECT WHERE owner_id = ? AND status = 'ACTIVE'`

Her birinde `EXPLAIN ANALYZE`. **Defterine** sürelerle tablo.

### Task 4.2.3 — JOIN algoritması inceleme (45 dk)

```sql
EXPLAIN ANALYZE 
SELECT a.id, t.amount 
FROM accounts a 
JOIN transactions t ON t.account_id = a.id 
WHERE a.owner_id = '...';
```

Plan'da hangi join algoritması seçildi? Neden? `accounts` küçük olsa nested loop, büyük olsa hash join.

### Task 4.2.4 — Stats güncelleme etkisi (30 dk)

1. 1M random transaction insert et
2. `EXPLAIN ANALYZE` çalıştır, plan'ı not al
3. `ANALYZE transactions;` çalıştır
4. Aynı query'i tekrar `EXPLAIN ANALYZE` — plan değişti mi?

### Task 4.2.5 — Yavaş query'i optimize et (60 dk)

`core-banking`'te kasten yavaş bir endpoint yarat:

```java
@GetMapping("/accounts/{ownerId}/active-balance")
BigDecimal activeBalance(@PathVariable UUID ownerId) {
    return jdbcTemplate.queryForObject("""
        SELECT SUM(balance_amount) 
        FROM accounts 
        WHERE owner_id = ? AND status = 'ACTIVE'
        """, BigDecimal.class, ownerId);
}
```

1. 100k account, 10k owner ile veri yükle
2. Endpoint'i çağır, sürede ölç
3. `EXPLAIN ANALYZE` ile incele
4. Index ekle, ölç
5. Composite index dene, ölç

Sürede iyileşmeyi **defterine** yaz.

### Task 4.2.6 — Function on indexed column tuzağı (15 dk)

```sql
CREATE INDEX idx_currency ON accounts(currency);

EXPLAIN ANALYZE SELECT * FROM accounts WHERE UPPER(currency) = 'TRY';
-- Index kullanıldı mı?

EXPLAIN ANALYZE SELECT * FROM accounts WHERE currency = 'TRY';
-- Şimdi?
```

İki plan'ı defter'e yaz.

---

## Test yazma rehberi

### Test 4.2.1 — Plan analysis test

```java
@Test
@Sql("/test-data/100k-accounts.sql")
void ownerIdQueryShouldUseIndex() {
    List<Map<String, Object>> plan = jdbcTemplate.queryForList(
        "EXPLAIN (FORMAT JSON) SELECT * FROM accounts WHERE owner_id = ?",
        UUID.randomUUID()
    );
    
    String planJson = plan.get(0).get("QUERY PLAN").toString();
    assertThat(planJson).contains("Index Scan");
    assertThat(planJson).doesNotContain("Seq Scan");
}

@Test
void unindexedQueryShouldFail() {
    // No index on customer_reference
    List<Map<String, Object>> plan = jdbcTemplate.queryForList(
        "EXPLAIN (FORMAT JSON) SELECT * FROM accounts WHERE customer_reference = ?",
        "TEST-001"
    );
    
    String planJson = plan.get(0).get("QUERY PLAN").toString();
    assertThat(planJson).contains("Seq Scan");
    // CI'da bunu fail yapmak için: assertThat(planJson).contains("Index Scan");
}
```

### Test 4.2.2 — Query timing assertion

```java
@Test
@Sql("/test-data/1m-transactions.sql")
void transactionHistoryQueryShouldRunUnder100ms() {
    UUID accountId = ...;
    long start = System.currentTimeMillis();
    
    List<Transaction> result = jdbcTemplate.query("""
        SELECT * FROM transactions
        WHERE account_id = ? AND occurred_at > ?
        ORDER BY occurred_at DESC LIMIT 50
        """, new Object[]{accountId, Instant.now().minus(30, ChronoUnit.DAYS)},
        ...);
    
    long elapsed = System.currentTimeMillis() - start;
    assertThat(elapsed).isLessThan(100L);
}
```

**Banking pratiği:** Performance regression testleri CI'da. Plan değişirse threshold'u patlatır.

---

## Claude-verify prompt

```
SQL execution plan ve query tuning çalışmamı banking-grade kriterlere göre 
değerlendir. Sadece eksikleri ve yanlışları işaretle:

1. EXPLAIN kullanımı:
   - EXPLAIN ANALYZE (gerçek timing) kullanılmış mı sadece EXPLAIN değil?
   - BUFFERS option ile cache hit/miss görülmüş mü?
   - VERBOSE ile detayli output incelenmiş mi?

2. Plan analizi:
   - Index Scan vs Seq Scan ayrımı yapılıyor mu?
   - Estimated rows vs actual rows karşılaştırılmış mı (stats güncelliği)?
   - Cost yorumlama bilinçli mi?

3. Join algorithm:
   - Nested Loop, Hash Join, Sort-Merge ayrımları biliniyor mu?
   - Banking senaryosunda doğru algoritma seçildiği plan'da doğrulanıyor mu?

4. Statistics:
   - ANALYZE düzenli çalıştırılıyor mu (EOD cron)?
   - Histogram'ın gerektiği skewed kolonlarda var mı?
   - pg_stats / DBA_TAB_COL_STATISTICS ile sorgulanıyor mu?

5. Hints:
   - Production'da gereksiz hint var mı? (Olmamalı — son çare)
   - Hint kullanılıyorsa neden açıklamalı bir comment var mı?

6. Banking yavaş query analizi:
   - `SELECT * FROM transactions ...` tipi yavaş query optimize edilmiş mi?
   - Composite index doğru kolon sırasıyla (selectivity yüksek önce)?
   - LIMIT + ORDER BY ile early termination yapılıyor mu?

7. Anti-pattern'ler:
   - `SELECT *` production code'da var mı?
   - Function on indexed column var mı?
   - OR ile farklı kolonlar (UNION'a uygunsa)?
   - `LIKE '%xxx'` wildcard başta?
   - Implicit type cast?

8. Plan stability:
   - Production'da plan değişimi monitoring var mı?
   - Critical query'lerin plan'ları snapshot olarak saklanıyor mu?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] EXPLAIN ve EXPLAIN ANALYZE arası farkı biliyorum
- [ ] Plan output'unu okuyup Index Scan vs Seq Scan ayırt edebiliyorum
- [ ] Cost vs actual time karşılaştırması yapabiliyorum (stats güncelliği)
- [ ] Nested loop / hash join / sort-merge algoritmalarını tanıyorum
- [ ] Banking yavaş query'i index ekleyerek 1000x hızlandırdım
- [ ] Function on indexed column tuzağını canlı gördüm
- [ ] ANALYZE ile plan değişimi gözlemledim
- [ ] Histogram'ın skewed kolonlarda neden gerektiğini biliyorum
- [ ] Plan stability/monitoring kavramına aşinayım

---

## Defter notları

1. "EXPLAIN ANALYZE'ın `actual rows` vs `estimated rows` farkı bana ne söyler: ____."
2. "Nested loop, hash join, sort-merge — banking için karar matrisim: ____."
3. "ANALYZE/DBMS_STATS.GATHER neden gerekli (CBO için yakıt): ____."
4. "Histogram skewed kolonda neden gerekli: ____."
5. "`SELECT * FROM x WHERE UPPER(col) = 'X'` neden index'i kullanmaz: ____."
6. "Composite index'te kolon sırası seçimi (selectivity): ____."
7. "Banking yavaş query analiz akışım (5 adım): ____."
8. "Production'da plan değişimi nasıl yakalanır: ____."
9. "Hint kullanımı için 'son çare' demek ne demek: ____."
10. "PostgreSQL EXPLAIN vs Oracle DBMS_XPLAN farkı: ____."
