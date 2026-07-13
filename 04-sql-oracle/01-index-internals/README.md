# Topic 4.1 — Index Internals

## Hedef

Veritabanı index'lerinin **içeride nasıl çalıştığını** derinlemesine anlamak. B-tree'nin yaprak/dal yapısı, bitmap index'in ne zaman uygun olduğu, function-based index'ler, composite index'te kolon sırasının önemi, selectivity ve cardinality, partial vs covering index. Banking domain'inde **hangi sorgu için hangi index** sorusunu kararlı cevaplayabilmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- SQL temel (SELECT, WHERE, JOIN)
- Faz 2'den JPA tarafında index'in faydasını gördün
- `CREATE INDEX` syntax'ını biliyorsun

---

## Kavramlar

### 1. Index nedir, neden gerekli — banking perspektifi

Bir banka düşün: 50 milyon hesap kaydı, 2 milyar transaction. Müşteri çağrı merkezini arıyor, "TC No 12345678901 hesaplarını listele" diyor. Bu sorgu **300 milisaniyede** dönmek zorunda (SLA).

Index olmasaydı:
- DB her satırı baştan sona okuyacak (**full table scan**)
- 2 milyar satır × ortalama 200 byte = ~400GB veri taranır
- Disk I/O sınırlı, bu sorgu **20-40 dakika** sürer
- Müşteri telefonu kapatır, çağrı merkezi memnuniyeti çöker

Index ile:
- DB önce küçük bir veri yapısına bakar (TC No → row pointer)
- Birkaç sayfa okur, doğru satıra direkt zıplar
- Sorgu **5-15 milisaniye**

**Index nedir?** Tabloya ait, **sıralı** veya **özel yapıda** bir veri kopyası. Sorguyu hızlandırmak için. Maliyeti var:
- Disk alanı
- Write performance (her INSERT/UPDATE/DELETE index'i de günceller)
- Maintenance (rebuild, statistics)

Banking'de denge kritik: **read-heavy tablolar için index bol**, **write-heavy/append-only tablolar için index seçici**.

---

### 2. B-tree Index — en yaygın yapı

B-tree (Balanced Tree). Oracle ve PostgreSQL'in **default** index türü. `CREATE INDEX foo ON t(c)` dediğinde B-tree yaratırsın.

#### Yapı

```
                    [Root Block]
                   /     |      \
                  /      |       \
          [Branch]   [Branch]   [Branch]
          /  |  \    /  |  \    /  |  \
        Leaf Leaf Leaf Leaf Leaf Leaf Leaf
         |    |    |    |    |    |    |
        Row  Row  Row  Row  Row  Row  Row
       Pointer pointers pointers...
```

**Anahtar kavramlar:**

- **Root block:** Ağacın tepesi, tek block
- **Branch block:** Ortadaki düğümler, "şu range için şu branch'e in" der
- **Leaf block:** En alttaki yaprak. Burada **gerçek key değerleri** ve **rowid pointer**'ları var
- **Height:** Root'tan leaf'e kadar olan derinlik. Oracle B-tree typically **2-4 seviye** (10M-1B satırda)

#### Neden "Balanced"?

Tüm leaf node'lar **aynı derinlikte**. Bir satır eklenince index split yapar (block bölünür), gerekirse yukarı doğru yayılır. Bu yüzden index lookup **logaritmik** zaman alır: O(log n).

#### Bir B-tree lookup adımları

`SELECT * FROM accounts WHERE account_no = '1234567890';`

1. Root block oku (1 disk I/O — ama cache'tedir genelde)
2. Root içinde "1234567890 hangi branch'e gider?" bul
3. O branch block'u oku (1 I/O)
4. Branch içinde "hangi leaf?" bul
5. Leaf block oku (1 I/O)
6. Leaf'te key'in tam karşılığını bul, rowid'i al
7. Tablo'dan rowid ile gerçek satırı oku (1 I/O — "table access by index rowid")

Toplam: ~4 I/O. **2 milyar satır için 4 I/O.** Index'in gücü bu.

#### Banking örnek

```sql
-- 50M hesap, müşteri TC'sine göre arama
CREATE TABLE accounts (
    account_id    VARCHAR2(20) PRIMARY KEY,
    customer_tc   VARCHAR2(11) NOT NULL,
    branch_code   VARCHAR2(4),
    balance       NUMBER(19,2),
    opened_at     DATE
);

-- B-tree index (PRIMARY KEY otomatik B-tree yaratır)
-- Customer TC için ayrı index
CREATE INDEX idx_accounts_customer_tc ON accounts(customer_tc);

-- Bu sorgu artık index kullanır
SELECT * FROM accounts WHERE customer_tc = '12345678901';
```

#### Leaf node detayı (Oracle özel)

Oracle B-tree leaf node'ları **bidirectional linked list** ile birbirine bağlı. Bu sayede **range scan** hızlı:

```sql
SELECT * FROM accounts WHERE opened_at BETWEEN DATE '2024-01-01' AND DATE '2024-01-31';
```

İlk leaf'i bulduktan sonra, leaf'ten leaf'e zıplayarak Ocak ayının tüm kayıtları gezilir. Hash index'te bu mümkün değil.

#### PostgreSQL farkı

PostgreSQL B-tree de aynı mantıkta, küçük internal farkları var (page boyutu farklı, sıkıştırma farklı). API'sı aynı: `CREATE INDEX`.

---

### 3. Bitmap Index — read-heavy OLAP için

**Tanım:** Her distinct değer için bir **bit dizisi** (bitmap) tutar. Bir bit = bir satır. 1 ise o satır o değere sahip, 0 ise değil.

```
status değerleri: 'ACTIVE', 'FROZEN', 'CLOSED'

Satır:    1  2  3  4  5  6  7  8
ACTIVE:   1  0  1  1  0  1  0  1
FROZEN:   0  1  0  0  0  0  1  0
CLOSED:   0  0  0  0  1  0  0  0
```

**Avantajları:**
- Çok düşük cardinality kolonlar için **çok küçük** (gender, status, country code)
- **AND/OR operasyonları çok hızlı** (bitwise) — birden fazla filter'ı combine ederken iyi
- Read-heavy data warehouse / OLAP için ideal

**Dezavantajları:**
- **Write performance felaket** — bir UPDATE → bitmap'in birden fazla satırı kilitlenir. OLTP için **kullanma**
- Yüksek cardinality için patlar (1M distinct değer → 1M bitmap)
- PostgreSQL'de **gerçek bitmap index yok** — bitmap heap scan var ama farklı şey

#### Banking örnek

```sql
-- Data warehouse'ta günlük transaction analizi
CREATE TABLE transactions_dwh (
    transaction_id  NUMBER PRIMARY KEY,
    branch_code     VARCHAR2(4),       -- ~500 distinct branch
    channel         VARCHAR2(20),      -- 'ATM','ONLINE','BRANCH','MOBILE' (4-5 distinct)
    transaction_type VARCHAR2(20),     -- 'DEPOSIT','WITHDRAW','TRANSFER',... (~10)
    amount          NUMBER(19,2),
    transaction_date DATE
);

-- OLTP transactions tablosu için bitmap KULLANMA
-- Bu tablo data warehouse, sadece nightly batch'le yüklenir → bitmap uygun
CREATE BITMAP INDEX bmi_dwh_channel ON transactions_dwh(channel);
CREATE BITMAP INDEX bmi_dwh_type ON transactions_dwh(transaction_type);

-- Bu sorgu bitmap AND ile çok hızlı
SELECT COUNT(*), SUM(amount)
FROM transactions_dwh
WHERE channel = 'MOBILE' AND transaction_type = 'TRANSFER';
```

**Pratik kural:** OLTP'de B-tree, OLAP/data warehouse'ta bitmap.

---

### 4. Function-Based Index

**Sorun:** WHERE clause'ta kolon üzerinde **fonksiyon** kullanılınca normal B-tree index kullanılmaz.

```sql
-- Index VAR ama kullanılmaz
CREATE INDEX idx_customer_tc ON accounts(customer_tc);

-- Bu sorgu full table scan yapar
SELECT * FROM accounts WHERE UPPER(customer_tc) = '12345678901';
-- ^^ UPPER fonksiyonu index'i invalidate ediyor
```

**Çözüm: Function-based index.**

```sql
CREATE INDEX idx_accounts_customer_tc_upper ON accounts(UPPER(customer_tc));

-- Şimdi sorgu index kullanır
SELECT * FROM accounts WHERE UPPER(customer_tc) = '12345678901';
```

#### Banking örnekler

```sql
-- 1. Case-insensitive arama (müşteri ad-soyad)
CREATE INDEX idx_customer_name_upper ON customers(UPPER(first_name), UPPER(last_name));

SELECT * FROM customers 
WHERE UPPER(first_name) = 'AHMET' AND UPPER(last_name) = 'YILMAZ';

-- 2. Tarihten yıl çıkarma (raporlamada yaygın)
CREATE INDEX idx_tx_year ON transactions(EXTRACT(YEAR FROM transaction_date));

SELECT * FROM transactions WHERE EXTRACT(YEAR FROM transaction_date) = 2024;

-- 3. IBAN'dan branch code çıkarma
-- TR IBAN format: TR + 24 char, position 5-9 banka kodu
CREATE INDEX idx_iban_bank ON accounts(SUBSTR(iban, 5, 4));

SELECT * FROM accounts WHERE SUBSTR(iban, 5, 4) = '0046'; -- Akbank
```

#### PostgreSQL: aynı şey "expression index"

```sql
-- PostgreSQL syntax (parantezler önemli)
CREATE INDEX idx_customer_name_lower ON customers((LOWER(first_name)));

SELECT * FROM customers WHERE LOWER(first_name) = 'ahmet';
```

**Dikkat:** Function-based index'in **deterministic** olması lazım. `SYSDATE` kullanan bir fonksiyon (sonucu her sefer değişir) index'lenemez.

---

### 5. Composite Index & Leftmost Prefix Rule

Birden fazla kolon üzerinde tek index:

```sql
CREATE INDEX idx_accounts_branch_currency ON accounts(branch_code, currency);
```

#### Leftmost Prefix Rule (en kritik kural)

Bir composite index `(A, B, C)` üzerine yaratıldıysa, şu sorgular index kullanır:

- `WHERE A = ?`                       (sadece A — kullanır, kısmi)
- `WHERE A = ? AND B = ?`             (A + B — kullanır)
- `WHERE A = ? AND B = ? AND C = ?`   (hepsi — kullanır)
- `WHERE A = ? AND C = ?`             (A var, B atlanmış — A kısmen kullanır, C için tarama)

Şu sorgular **index kullanmaz** (veya verimsiz kullanır):

- `WHERE B = ?`                       (A atlanmış → leftmost prefix yok)
- `WHERE C = ?`                       (sadece sondaki)
- `WHERE B = ? AND C = ?`             (A atlanmış)

**Sebep:** Index `A → B → C` sırasında sıralı. A bilinmeden B'yi bulamazsın (telefon rehberinde soyadına göre sıralı, ad bilinmeden ortalardan başlayamazsın).

#### Banking örnek — kolon sırası kararı

```sql
-- Sıklıkla bu üç sorgu:
-- 1) WHERE branch_code = ? AND currency = ?
-- 2) WHERE branch_code = ?
-- 3) WHERE currency = ?

-- En seçici ve en sık kullanılan kolon BAŞA gelmeli
-- branch_code 500 distinct, currency 5-10 distinct
-- branch_code daha selektif → başa
CREATE INDEX idx_accounts_branch_currency ON accounts(branch_code, currency);

-- Sorgu 1 ve 2 index kullanır.
-- Sorgu 3 (`WHERE currency = ?`) → index leftmost prefix yok → full table scan
-- Ya ayrı bir index ekle (currency tek başına), ya bu sorguyu kabul et
```

#### Selectivity-First Rule

**En seçici kolonu başa koy.** Selectivity = distinct değer sayısı / toplam satır sayısı. 1'e yakın olan selektiftir.

```
customer_tc:    50M satırda 30M distinct → selectivity 0.6 → çok selektif
branch_code:    50M satırda 500 distinct → selectivity 0.00001 → düşük
status:         50M satırda 3 distinct   → selectivity 0.0000001 → çok düşük
```

Eğer hem `customer_tc` hem `status` ile filter atıyorsan, `customer_tc`'yi başa koy. Çünkü `customer_tc`'ye göre filter çok az satıra düşürür, sonra `status`'a göre filter zaten az satır içinde.

#### Skip Scan (Oracle özel)

Oracle 9i+ — eğer leftmost kolon **çok az distinct** ise (örn. 3 değer), Oracle "skip scan" yapabilir. Her distinct değer için ayrı ayrı index range scan dener. Yavaş ama full table scan'den iyi.

```sql
-- (gender, salary) index'i var, sadece salary filter
SELECT * FROM employees WHERE salary > 100000;
-- Oracle: önce gender='M' index walk, sonra gender='F' walk → birleştirir
```

Garanti değil — CBO karar verir. Skip scan'e **güvenme**, doğru kolon sırasını seç.

---

### 6. Selectivity ve Cardinality

İki terim çok karıştırılır:

- **Cardinality:** Bir kolondaki **distinct değer sayısı** veya bir sorgunun döndürdüğü **tahmini satır sayısı**
- **Selectivity:** Bir filter'ın **döndürdüğü satır oranı** (0-1 arası, 1'e yakınsa çok satır gelir, 0'a yakınsa az)

```
status kolonu: 3 distinct (ACTIVE, FROZEN, CLOSED)
→ Cardinality 3

WHERE status = 'CLOSED' (toplam 50M satırın 100K'sı)
→ Selectivity = 100K / 50M = 0.002 (çok az)
```

#### Index neden bazen kullanılmaz?

Eğer selectivity **yüksekse** (örn. 0.5 — yarı satır geliyor), index kullanmak **YAVAŞLATIR**:

1. Index leaf'ten 25M rowid oku
2. Her rowid için tabloya atla, satırı oku (random I/O)
3. = 25M random I/O

**Daha hızlı:** Full table scan — sıralı disk okuma, 25M satırı baştan sona.

CBO (Cost-Based Optimizer) bu kararı **statistics'e bakarak** verir. Statistics yanlışsa, plan yanlış.

#### Banking sezgisi

```
Selectivity < 0.01  → Index yararlı (B-tree)
Selectivity 0.01-0.05 → Marjinal, statistics'e bağlı
Selectivity > 0.05  → Çoğunlukla full table scan daha iyi
```

`SELECT * FROM accounts WHERE status = 'CLOSED'` — 50M'in 100K'sı (0.002) → index yarar.
`SELECT * FROM accounts WHERE status = 'ACTIVE'` — 50M'in 49M'si (0.98) → full scan daha iyi.

Aynı index, aynı sorgu, **farklı parametre** → farklı plan. Bu "bind variable peeking" konusu (Topic 4.2'de).

---

### 7. Partial Index (PostgreSQL) ve Function-Based Index Trick (Oracle)

#### PostgreSQL partial index

```sql
-- Sadece aktif olmayan hesapları index'le (closed/frozen'lar küçük subset)
CREATE INDEX idx_accounts_inactive ON accounts(customer_tc)
WHERE status != 'ACTIVE';
```

Avantajlar:
- Index **çok daha küçük** (50M değil, 500K satır)
- Write maintenance az (active hesaplara INSERT/UPDATE index'i etkilemez)
- Memory'de daha az yer

#### Oracle'da partial index yok ama trick var

Oracle 12c+ "partial indexes for partitioned tables" var ama generic partial yok. Trick: **function-based index ile NULL kullan.**

```sql
-- "Aktif olmayan" hesaplar için index simulate et
CREATE INDEX idx_accounts_inactive_oracle ON accounts(
    CASE WHEN status != 'ACTIVE' THEN customer_tc ELSE NULL END
);

-- Oracle null değerleri B-tree'de **tutmaz** (kompozit hariç)
-- Sonuç: sadece status != 'ACTIVE' satırlar index'te
```

Sorgu da aynı CASE'i kullanmak zorunda — kullanışsız. Genelde Oracle'da **tüm tablo index'lenir**, partitioning ile dengelenir.

---

### 8. Reverse-Key Index (Oracle özel)

**Problem:** Sequential primary key'lerde (`customer_id` 1, 2, 3, ...) ardışık INSERT'ler hep **son leaf**'e gider. Bu hot spot olur (özellikle RAC'de).

```
Normal B-tree:
[100, 101, 102, 103, ...] ← yeni kayıt hep buraya, son leaf split olur sürekli
```

**Reverse-key index:** Key'in byte'larını **ters çevirir**.

```sql
CREATE INDEX idx_customer_id_rev ON customers(customer_id) REVERSE;

-- 100 → '001'
-- 101 → '101'
-- 102 → '201'
-- ...
-- Insert'ler farklı leaf'lere dağılır
```

**Trade-off:** Range scan **çalışmaz** (sıra bozuldu). `WHERE id BETWEEN 100 AND 200` artık index kullanamaz.

**Banking kullanımı:** Yüksek-throughput insert tablo (audit log, transaction log) ve **range scan yapılmayan** durumlar.

Genelde **sequence cache'i artırarak** çözmek daha pratik (Topic 4.5).

---

### 9. Covering Index — sorguyu index'ten çözmek

**Tanım:** Eğer sorgunun ihtiyaç duyduğu **tüm kolonlar** index'te varsa, DB tablo'ya gitmez, sadece index'i okur. "Index-only scan."

```sql
-- Tablo: accounts(account_id, customer_tc, branch_code, balance, status, opened_at)
CREATE INDEX idx_accounts_tc_balance ON accounts(customer_tc, balance);

-- Bu sorgu sadece customer_tc ve balance istiyor
-- Index'te ikisi de var → tabloya gitmez
SELECT customer_tc, balance FROM accounts WHERE customer_tc = '12345678901';
```

`EXPLAIN PLAN`'da görürsün: "INDEX RANGE SCAN" var, "TABLE ACCESS BY INDEX ROWID" yok.

#### PostgreSQL `INCLUDE` clause

PostgreSQL 11+:

```sql
-- customer_tc'ye göre filter, balance'ı sadece taşı (sırala değil)
CREATE INDEX idx_accounts_tc_inc_balance ON accounts(customer_tc) INCLUDE(balance);
```

`INCLUDE` kolonları index'te tutar ama **sıralama'ya katmaz**. Index daha küçük, covering avantajı korunur.

#### Oracle'da INCLUDE alternatifi

Oracle 19c+ — `CREATE INDEX foo ON t(a) INCLUDE(b)` desteği geldi. Önceki versiyonlarda kompozit index kullan: `(a, b)`. Tek fark: kompozit'te `b`'ye göre sıralama var (gereksiz overhead).

#### Banking örnek — covering index ile raporlama

```sql
-- Sıkça çekilen rapor: müşteri başına toplam bakiye
-- SELECT customer_tc, SUM(balance) FROM accounts GROUP BY customer_tc;

CREATE INDEX idx_accounts_tc_balance_cover ON accounts(customer_tc, balance);

-- Bu sorgu artık index'ten direkt cevaplanır
-- INDEX FAST FULL SCAN (sıralı leaf taraması, tabloya hiç gitmez)
```

**Dikkat:** Çok geniş covering index = çok büyük index → write performance düşer. Trade-off yap.

---

### 10. Index ne zaman zarar verir?

#### Write-heavy tablolarda overhead

Her INSERT için:
- Tabloya 1 yazma
- Her index için 1 yazma (block bul, key ekle, gerekirse split)

5 index olan tablo → 1 INSERT = 6 yazma.

Banking'de `transactions` tablosu (append-only, çok yüksek throughput):
- Birincil: `transaction_id` (PRIMARY KEY)
- Sorgulanan: `account_id`, `transaction_date`
- 2-3 index yeter. 10 index = TPS yarı yarıya düşer.

#### Cardinality düşük kolonlar

```sql
-- KÖTÜ
CREATE INDEX idx_accounts_status ON accounts(status);
-- status sadece 3 değer alır → çok az selectivity
-- WHERE status = 'ACTIVE' → 49M satır → index kullanılmaz (full scan daha hızlı)
-- WHERE status = 'CLOSED' → 100K satır → index kullanılabilir ama marjinal
```

**Çözüm:** Composite olarak ekle (status + bir başka selektif kolon), veya partial index (PostgreSQL), veya hiç ekleme.

#### Foreign key kolonu index'siz

**Banking'de en sık görülen hata.** FK kolonuna index koymamak.

```sql
-- accounts(customer_id REFERENCES customers(id))
-- accounts.customer_id üzerinde index YOK

-- Şu olur:
-- 1) DELETE FROM customers WHERE id = 123;
-- 2) Oracle "bu customer'ın accounts'u var mı?" diye accounts'u tarar
-- 3) Full table scan → DELETE çok yavaş
-- 4) Worst: TM lock alır (table-level), concurrent INSERT/UPDATE'leri bloklar

-- KESIN ÇÖZÜM: Her FK kolonuna index
CREATE INDEX idx_accounts_customer_id ON accounts(customer_id);
```

Bu **mandatory**. Banking'de FK index'siz tablo = production'da deadlock kaynağı.

#### Update'lerde overhead

```sql
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'X';
-- balance üzerinde index varsa → eski entry sil, yeni entry ekle
-- 1 satır UPDATE ama index için 2 operasyon
```

Sık güncellenen kolonlara (balance, version, updated_at) **dikkatli index** ekle.

---

### 11. Bitmap join index, function-based on multiple columns

Daha advanced türler (Oracle özel, fazla kullanılmaz ama bilelim):

- **Bitmap Join Index:** İki tablonun join'ini önceden hesaplayıp bitmap olarak tutar. DWH için.
- **Index-Organized Table (IOT):** Tablo zaten index — primary key sırasında saklanır. Heap yok. Range scan çok hızlı, point-lookup hızlı. Update'leri pahalı.
- **Reverse Function-Based Index:** Function-based + reverse, ikisinin combination'ı.

Banking junior'ın bunları bilmesine gerek yok, ama "ne için var" cevabını verebilmek lazım. Mid-level'da öğrenirsin.

---

### 12. Index Anti-Pattern'leri (banking)

**Anti-pattern 1:** "Her şeye index"
```sql
-- ❌ KÖTÜ
CREATE INDEX idx1 ON accounts(branch_code);
CREATE INDEX idx2 ON accounts(currency);
CREATE INDEX idx3 ON accounts(status);
CREATE INDEX idx4 ON accounts(opened_at);
CREATE INDEX idx5 ON accounts(closed_at);
CREATE INDEX idx6 ON accounts(balance);
-- Hiçbiri çok selektif değil, hepsi maintenance yükü
```

**Anti-pattern 2:** "Composite index'lerin kolon sırasını rastgele seçmek"
```sql
-- ❌ Kolon sırası yanlış
CREATE INDEX idx_bad ON transactions(currency, amount, account_id);
-- Çoğu sorgu account_id ile başlar → index leftmost yok
-- amount range query'leri için uygunsuz

-- ✅ Doğru
CREATE INDEX idx_good ON transactions(account_id, transaction_date, amount);
-- account_id en selektif, sonra date range, sonra amount sort
```

**Anti-pattern 3:** "WHERE clause'ta kolonu fonksiyona sokmak"
```sql
-- ❌ Index iptal
SELECT * FROM accounts WHERE TRUNC(opened_at) = DATE '2024-01-15';

-- ✅ Düzelt — fonksiyonu literal'e taşı
SELECT * FROM accounts 
WHERE opened_at >= DATE '2024-01-15' AND opened_at < DATE '2024-01-16';
```

**Anti-pattern 4:** "Implicit type conversion"
```sql
-- accounts.customer_tc VARCHAR2(11)
-- ❌ Java tarafından integer geliyor
SELECT * FROM accounts WHERE customer_tc = 12345678901;
-- Oracle implicit cast yapar: TO_NUMBER(customer_tc) = 12345678901
-- → Index'i bypass eder!

-- ✅ String olarak gönder
SELECT * FROM accounts WHERE customer_tc = '12345678901';
```

JPA tarafında parameter type'ı string olduğundan emin ol. Bu **prod'da gizli yavaşlama** kaynağı.

**Anti-pattern 5:** "Index'i rebuild ile düzeltmeye çalışmak"
```sql
ALTER INDEX idx_foo REBUILD;
```
Bu eski Oracle disiplini. Bugün Oracle 12c+ self-managing — gerek yok genelde. Statistics güncelle, gerçek soruna bak.

---

### 13. Banking'de pratik index önerileri

`accounts` tablosu için:
```sql
-- PK (otomatik)
ALTER TABLE accounts ADD CONSTRAINT pk_accounts PRIMARY KEY (account_id);

-- Müşteri lookup
CREATE INDEX idx_accounts_customer_tc ON accounts(customer_tc);

-- Branch raporlama
CREATE INDEX idx_accounts_branch_status ON accounts(branch_code, status);

-- IBAN unique lookup
CREATE UNIQUE INDEX idx_accounts_iban ON accounts(iban);
```

`transactions` tablosu için (en kritik):
```sql
-- PK
ALTER TABLE transactions ADD CONSTRAINT pk_tx PRIMARY KEY (transaction_id);

-- Hesap-tarih range (en yaygın sorgu)
CREATE INDEX idx_tx_account_date ON transactions(account_id, transaction_date DESC);
-- DESC çünkü genelde "son işlemler" çekilir

-- Idempotency key
CREATE UNIQUE INDEX idx_tx_idempotency_key ON transactions(idempotency_key);

-- FK index
CREATE INDEX idx_tx_account_id ON transactions(account_id);
-- ↑ Aslında idx_tx_account_date varsa bu redundant (leftmost prefix)
-- account_id tek başına da sorgulanabilir
```

`journal_lines` tablosu:
```sql
-- PK
ALTER TABLE journal_lines ADD CONSTRAINT pk_jl PRIMARY KEY (id);

-- FK'lar — banking'de mandatory
CREATE INDEX idx_jl_journal_entry ON journal_lines(journal_entry_id);
CREATE INDEX idx_jl_account ON journal_lines(account_id);

-- Hesap ekstresi sorgusu için
CREATE INDEX idx_jl_account_date ON journal_lines(account_id, created_at);
```

---

## Önemli olabilecek araştırma kaynakları

- Oracle Database SQL Tuning Guide (özellikle "Optimizer Statistics", "Indexes and Index-Organized Tables")
- Use The Index, Luke! (use-the-index-luke.com) — Markus Winand, çok yüksek kalite
- "SQL Performance Explained" — Markus Winand kitap
- PostgreSQL doc: "Indexes" chapter
- Oracle DBMS_INDEX_UTL package
- "Cost-Based Oracle Fundamentals" Jonathan Lewis (klasik referans)
- Tom Kyte (asktom.oracle.com) blog yazıları — index ile ilgili her şey
- "Expert Oracle Database Architecture" Tom Kyte
- Christian Antognini "Troubleshooting Oracle Performance"

---

## Mini task'ler

`~/projects/core-banking/` Oracle XE'yi henüz kurmadıysan mini-project'e başlamadan önce mini bir kurulum yap. Topic seviyesinde Docker'da Oracle çalıştır:

```bash
docker run -d --name oracle-xe \
  -p 1521:1521 -p 5500:5500 \
  -e ORACLE_PWD=banking_dev \
  gvenzl/oracle-xe:21-slim
```

Container'a gir:
```bash
docker exec -it oracle-xe sqlplus banking_dev/banking_dev@XEPDB1
```

### Task 4.1.1 — B-tree height'ı görme (20 dk)

```sql
-- Test tablosu oluştur
CREATE TABLE test_accounts (
    account_id    VARCHAR2(20) PRIMARY KEY,
    customer_tc   VARCHAR2(11) NOT NULL,
    balance       NUMBER(19,2),
    opened_at     DATE
);

-- 1M satır insert (5 dakika alır)
INSERT INTO test_accounts
SELECT 
    LPAD(ROWNUM, 20, '0'),
    LPAD(MOD(ROWNUM, 100000), 11, '0'),
    ROUND(DBMS_RANDOM.VALUE(0, 100000), 2),
    DATE '2020-01-01' + DBMS_RANDOM.VALUE(0, 1500)
FROM dual CONNECT BY LEVEL <= 1000000;
COMMIT;

-- Statistics güncel olsun
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'TEST_ACCOUNTS');

-- Index oluştur
CREATE INDEX idx_test_customer_tc ON test_accounts(customer_tc);

-- B-tree height'ı oku
SELECT index_name, blevel, leaf_blocks, num_rows
FROM user_indexes
WHERE table_name = 'TEST_ACCOUNTS';
```

`blevel` = ağacın derinliği (root hariç). 1M satır için typically 2-3. **Defterine yaz.**

### Task 4.1.2 — Index kullanıyor mu görmek (20 dk)

```sql
-- Plan görmek için
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE customer_tc = '00000012345';

SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
```

Output'ta "INDEX RANGE SCAN" görmelisin. **Eğer "FULL TABLE SCAN" görüyorsan**, neden? Defterine yaz.

Şimdi tam tersi:
```sql
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE balance > 50000;

SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
```

`balance` üzerinde index yok → full table scan. Bu normal.

### Task 4.1.3 — Implicit type conversion (20 dk)

```sql
-- customer_tc VARCHAR2(11), integer ile sorgu at
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE customer_tc = 12345;
-- ^^ string yerine number

SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
```

Output'ta "INDEX" görmeyebilirsin (TO_NUMBER cast index'i invalidate ediyor). Sonra düzelt:

```sql
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE customer_tc = '12345';

SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
```

İki plan'ın farkını **defterine yaz**. Bu hata production'da JPA parameter binding ile sık görülür.

### Task 4.1.4 — Composite index leftmost prefix (30 dk)

```sql
-- 3 kolonlu composite
DROP INDEX idx_test_customer_tc;
CREATE INDEX idx_test_composite ON test_accounts(customer_tc, opened_at, balance);

-- Test 1: tüm kolonlar → index kullanır
EXPLAIN PLAN FOR
SELECT * FROM test_accounts 
WHERE customer_tc = '00000012345' 
  AND opened_at = DATE '2024-01-15' 
  AND balance > 1000;

SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());

-- Test 2: sadece customer_tc → index kullanır (leftmost)
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE customer_tc = '00000012345';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());

-- Test 3: sadece opened_at → leftmost yok, ne olur?
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE opened_at = DATE '2024-01-15';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());

-- Test 4: customer_tc + balance (opened_at atlandı)
EXPLAIN PLAN FOR
SELECT * FROM test_accounts 
WHERE customer_tc = '00000012345' AND balance > 1000;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
-- Index kullanılır ama balance için kısmi (range scan + filter)
```

4 senaryoda planları **karşılaştır**, defterine yaz. Bu pratik bilgiyi cebine koy.

### Task 4.1.5 — Function-based index (30 dk)

```sql
-- Case-insensitive search lazım
-- Önce normal index ile dene
EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE UPPER(customer_tc) = '00000012345';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
-- FULL TABLE SCAN

-- Şimdi function-based
CREATE INDEX idx_test_customer_tc_upper ON test_accounts(UPPER(customer_tc));
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'TEST_ACCOUNTS');

EXPLAIN PLAN FOR
SELECT * FROM test_accounts WHERE UPPER(customer_tc) = '00000012345';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());
-- INDEX RANGE SCAN
```

### Task 4.1.6 — FK index yokluğunun zararı (45 dk)

```sql
-- Parent tablo
CREATE TABLE customers (
    customer_id   NUMBER PRIMARY KEY,
    customer_tc   VARCHAR2(11) NOT NULL UNIQUE,
    name          VARCHAR2(200)
);

-- Child tablo, FK
CREATE TABLE customer_accounts (
    account_id    NUMBER PRIMARY KEY,
    customer_id   NUMBER REFERENCES customers(customer_id),
    balance       NUMBER(19,2)
);

-- 100K parent, 1M child insert
INSERT INTO customers SELECT ROWNUM, LPAD(ROWNUM, 11, '0'), 'CUST_' || ROWNUM 
FROM dual CONNECT BY LEVEL <= 100000;

INSERT INTO customer_accounts 
SELECT ROWNUM, MOD(ROWNUM, 100000) + 1, DBMS_RANDOM.VALUE(0, 10000) 
FROM dual CONNECT BY LEVEL <= 1000000;
COMMIT;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMERS');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMER_ACCOUNTS');

-- DELETE'i zaman al
SET TIMING ON
DELETE FROM customers WHERE customer_id = 1;
-- ↑ customer_accounts'ta FK var ama index yok
-- Oracle full scan yapacak

ROLLBACK;

-- FK index ekle
CREATE INDEX idx_ca_customer_id ON customer_accounts(customer_id);

DELETE FROM customers WHERE customer_id = 1;
-- ↑ Çok daha hızlı

ROLLBACK;
```

Zaman farkını **defterine yaz**. Bu prod'da senin başına gelecek bir sorun.

### Task 4.1.7 — Covering index (30 dk)

```sql
-- Sadece customer_tc ve balance lazım
EXPLAIN PLAN FOR
SELECT customer_tc, balance FROM test_accounts WHERE customer_tc = '00000012345';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());

-- Plan'da "TABLE ACCESS BY INDEX ROWID" var mı? Bak.

-- Covering index ekle
DROP INDEX idx_test_composite;
CREATE INDEX idx_test_covering ON test_accounts(customer_tc, balance);

EXPLAIN PLAN FOR
SELECT customer_tc, balance FROM test_accounts WHERE customer_tc = '00000012345';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY());

-- Şimdi sadece "INDEX RANGE SCAN" var, tabloya gitmiyor
```

### Task 4.1.8 — Bitmap index OLTP'de neden zarar (PostgreSQL'de yok, Oracle'da test)

```sql
-- Önemli: Bitmap'i sadece deneyim için kuruyoruz, banking OLTP'de KULLANMA
CREATE BITMAP INDEX bmi_test_status ON test_accounts(status);
-- ↑ önce status kolonu eklemen gerek, ya da hayali yap

-- Concurrent UPDATE testi
-- Session 1
UPDATE test_accounts SET balance = balance + 1 WHERE account_id = '00000000000000000001';

-- Session 2 (başka pencere)
UPDATE test_accounts SET balance = balance + 1 WHERE account_id = '00000000000000000002';
-- Bekler mi? Bitmap index aynı block'taki tüm satırları locklar
```

Detay (status kolonu yok deyebilirsin) → görmek istiyorsan banking'in DWH tablosunda dene.

---

## Test yazma rehberi

Index tuning Java unit test'le değil, **SQL ile** test edilir. Strateji:

1. **Repeatable migration** ile index oluştur
2. **TestContainers + Oracle** ile integration test
3. Test'te `EXPLAIN PLAN` çalıştır, output'u parse et
4. Beklenen plan'ı assert et

```java
@Testcontainers
@SpringBootTest
class IndexPlanTest {

    @Container
    static OracleContainer oracle = new OracleContainer("gvenzl/oracle-xe:21-slim");

    @DynamicPropertySource
    static void registerProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", oracle::getJdbcUrl);
        registry.add("spring.datasource.username", oracle::getUsername);
        registry.add("spring.datasource.password", oracle::getPassword);
    }

    @Autowired JdbcTemplate jdbc;

    @Test
    void customerTcLookupShouldUseIndex() {
        // 10K satır yarat
        for (int i = 0; i < 10000; i++) {
            jdbc.update("INSERT INTO accounts (account_id, customer_tc, balance) VALUES (?,?,?)",
                "ACC" + i, String.format("%011d", i), 1000.0);
        }
        jdbc.execute("BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER, 'ACCOUNTS'); END;");

        // Plan al
        jdbc.execute("EXPLAIN PLAN FOR SELECT * FROM accounts WHERE customer_tc = '00000000123'");
        List<String> plan = jdbc.queryForList(
            "SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY())",
            String.class
        );
        String planText = String.join("\n", plan);

        // Assert
        assertThat(planText).contains("INDEX RANGE SCAN");
        assertThat(planText).doesNotContain("TABLE ACCESS FULL");
    }

    @Test
    void implicitTypeConversionShouldNotUseIndex() {
        // customer_tc VARCHAR2, integer literal verince index kullanmaz
        jdbc.execute("EXPLAIN PLAN FOR SELECT * FROM accounts WHERE customer_tc = 123");
        List<String> plan = jdbc.queryForList(
            "SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY())",
            String.class
        );
        String planText = String.join("\n", plan);
        assertThat(planText).contains("TABLE ACCESS FULL");
        // Bu test "regression" testi — kodda parameter binding string olduğundan emin olmak için
    }
}
```

### Test yazmanın kuralı

- Hardcoded plan output'un Oracle versiyonuna göre değişebilir. **Substring match** kullan, exact match değil.
- Statistics güncel olmadan plan farklı olabilir. Test'te her zaman `DBMS_STATS.GATHER_TABLE_STATS` çağır.
- TestContainer Oracle XE çok yavaş başlar (1-2 dk). Test sınıfı başına bir container, method bazında değil.

---

## Claude-verify prompt

```
Aşağıdaki SQL şemasını ve index kararlarımı banking-grade index design kriterlerine 
göre değerlendir. Kod yazma, sadece sorunları belirt:

1. B-tree index'ler:
   - Her tabloda primary key var mı (otomatik unique B-tree)?
   - Her foreign key kolonuna ayrı index eklenmiş mi?
   - Composite index'lerde kolon sırası selectivity-first prensibi ile mi?

2. Selectivity:
   - Düşük cardinality kolonlara (status, gender, country gibi) tek başına index 
     eklenmiş mi (genelde gereksiz)?
   - Çok yüksek cardinality kolonlar (transaction_id, IBAN) için UNIQUE constraint mı?

3. Composite index leftmost prefix:
   - Sorgu pattern'larına bakıldığında composite index'lerin başlangıç kolonları 
     sorgu WHERE'inde yer alıyor mu?
   - Aynı tabloda 3+ composite index varsa, redundancy var mı (örn. (A,B), (A,B,C), (A,C))?

4. Function-based / expression index:
   - WHERE clause'ta UPPER, TRUNC, EXTRACT gibi fonksiyonlar varsa, karşılığı 
     function-based index oluşturulmuş mu?
   - PostgreSQL ise expression index `((LOWER(col)))` formatında mı?

5. Covering index:
   - Sık çalışan raporlama sorguları için "select only columns in index" durumu 
     araştırılmış mı? (INDEX FAST FULL SCAN)
   - Çok geniş covering index (5+ kolon) write overhead düşünülerek dengelendi mi?

6. Bitmap index:
   - OLTP tablosunda bitmap index kullanılmış mı? (KULLANILMAMALI)
   - OLAP/DWH tablosunda düşük cardinality kolonlar için bitmap düşünülmüş mü?

7. Anti-pattern:
   - "Her kolona bir index" pattern'i var mı?
   - WHERE clause'ta kolon fonksiyona giriyor mu, karşılığı yok mu?
   - Implicit type conversion (string ↔ number) riski var mı (entity field tipleri)?
   - FK index'siz tablo var mı?
   - Reverse-key index gerçekten gerekli mi yoksa sequence cache yeterli mi?

8. Banking-specific:
   - accounts(customer_tc) için index var mı (en sık lookup)?
   - transactions(account_id, transaction_date DESC) composite var mı?
   - journal_lines(journal_entry_id) ve journal_lines(account_id) FK index'leri var mı?
   - IBAN unique index var mı?
   - idempotency_key unique index var mı?

9. Maintenance:
   - DBMS_STATS.GATHER_TABLE_STATS çalıştırılıyor mu (job veya manuel)?
   - Index rebuild gerekli mi yoksa Oracle 12c+ self-managing yeterli mi?

Her madde için PASS / FAIL / EKSIK işaretle. Kod yazma.
```

---

## Tamamlama kriterleri

- [ ] B-tree height'ı bir tabloda görmüş ve `blevel` kolonunu user_indexes'tan okumayı biliyorum
- [ ] `EXPLAIN PLAN FOR ...` + `DBMS_XPLAN.DISPLAY()` ile plan okuyabilirim
- [ ] "INDEX RANGE SCAN", "INDEX UNIQUE SCAN", "INDEX FAST FULL SCAN", "TABLE ACCESS BY INDEX ROWID", "TABLE ACCESS FULL" arasındaki farkı söyleyebilirim
- [ ] Composite index leftmost prefix kuralını anladım, örnek verebilirim
- [ ] Selectivity ve cardinality farkını ezbere biliyorum
- [ ] Function-based index ne zaman gerekli, sebepleri ile açıklayabilirim
- [ ] Covering index nedir, tablo erişimini nasıl önler söyleyebilirim
- [ ] PostgreSQL `INCLUDE` vs Oracle composite trade-off'unu biliyorum
- [ ] FK kolonuna neden index gerek, deneyimi yaşadım (DELETE yavaşlığı)
- [ ] Implicit type conversion sebebiyle index iptal olabileceğini gördüm
- [ ] Bitmap index'in OLTP'de neden zararlı olduğunu söyleyebilirim
- [ ] Banking için core index seti (`accounts`, `transactions`, `journal_lines`) tasarlayabilirim

→ Sonraki: [02-execution-plan-tuning/](../02-execution-plan-tuning/README.md)

---

## Defter notları

Aşağıdaki cümleleri **kendi kelimelerinle** doldur:

1. "B-tree index'in height'ı (`blevel`) ne ifade eder, neden 'balanced' denir: ____."
2. "Bitmap index'in OLTP'de neden kullanılmaması gerektiği: ____."
3. "Function-based index ne zaman gerekli, banking'deki örneği ____."
4. "Composite index `(A, B, C)`'de leftmost prefix kuralı: ____. WHERE B = ? sorgusu neden çalışmaz: ____."
5. "Selectivity ve cardinality farkı: ____. Banking'de örneklerle: ____."
6. "Foreign key kolonuna index neden mandatory: ____. Olmazsa ne olur: ____."
7. "Covering index avantajı: ____. PostgreSQL'de `INCLUDE`, Oracle'da alternative: ____."
8. "Implicit type conversion index'i nasıl bozar: ____. JPA tarafında nasıl önlersin: ____."
9. "Write-heavy tabloya çok index eklemenin maliyeti: ____."
10. "B-tree leaf node'ların 'bidirectional linked list' olması neden önemli: ____."
