# Topic 4.3 — Window Functions & Advanced SQL

## Hedef

SQL'in en güçlü ama az kullanılan özelliklerini banking örnekleriyle öğrenmek: **window functions** (ROW_NUMBER, RANK, LEAD, LAG, NTILE), **CTE** (Common Table Expressions), **recursive CTE**, **MERGE** (upsert), Oracle hierarchical query (`CONNECT BY`). Banking reporting'de aggregate ile yapılamayan ama window function ile şık çözülen senaryoları kavramak.

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 45 dk • Toplam: ~4.5 saat

## Önbilgi

- Topic 4.1-4.2 bitti
- Temel `GROUP BY`, aggregate function'lar (SUM, AVG, COUNT) rahat
- JOIN tipleri biliniyor

---

## Kavramlar

### 1. Window function nedir, neden GROUP BY yetmiyor

**GROUP BY:** Satırları gruplar, her grup için tek agregat satır döner. Detay kaybolur.

```sql
SELECT owner_id, SUM(balance_amount) FROM accounts GROUP BY owner_id;
-- owner_id başına 1 satır
```

**Window function:** Aggregate hesaplar ama **her satır kaybolmaz** — her satır kendi agregat değerini taşır.

```sql
SELECT 
    id, owner_id, balance_amount,
    SUM(balance_amount) OVER (PARTITION BY owner_id) AS owner_total
FROM accounts;
-- Her account satırı + owner total
```

Banking örneği: Her hesabın bakiyesi + sahibinin **toplam** bakiyesi — aynı sorguda.

### 2. OVER clause — pencere tanımı

```sql
function() OVER (
    PARTITION BY col1, col2     -- pencere grupları (opsiyonel)
    ORDER BY col3 DESC          -- pencere içi sıralama (opsiyonel)
    ROWS BETWEEN ... AND ...    -- frame (opsiyonel)
)
```

- **PARTITION BY:** Window function'ı her partition için bağımsız hesap. `GROUP BY` gibi ama satır kaybetmez.
- **ORDER BY:** Pencere içinde sıralama — LEAD/LAG, running sum için kritik.
- **Frame (ROWS/RANGE):** Pencere içinde **alt-pencere**. Default: ORDER BY varsa `UNBOUNDED PRECEDING TO CURRENT ROW`.

### 3. ROW_NUMBER — sıra numarası

```sql
SELECT 
    id, owner_id, balance_amount,
    ROW_NUMBER() OVER (PARTITION BY owner_id ORDER BY balance_amount DESC) AS rn
FROM accounts;
```

Her owner için hesapları bakiyeye göre sıralar (1, 2, 3, ...). En zengin hesap `rn=1`.

**Banking örneği — owner'ın top N hesabı:**

```sql
WITH ranked AS (
    SELECT a.*, 
           ROW_NUMBER() OVER (PARTITION BY owner_id ORDER BY balance_amount DESC) rn
    FROM accounts a
)
SELECT * FROM ranked WHERE rn <= 3;
-- Her owner'ın en yüksek 3 hesabı
```

`ROW_NUMBER` **eşitlikleri ayırır** (tie-breaking). Hep unique sıra.

### 4. RANK ve DENSE_RANK — sıralama

```sql
SELECT 
    name, score,
    RANK() OVER (ORDER BY score DESC) AS r,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dr,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS rn
FROM scoreboard;
```

| name | score | r | dr | rn |
|---|---|---|---|---|
| A | 100 | 1 | 1 | 1 |
| B | 100 | 1 | 1 | 2 |
| C | 90  | 3 | 2 | 3 |
| D | 80  | 4 | 3 | 4 |

- **RANK:** Eşitlik aynı sıra, sonraki atlanır (1, 1, 3, 4)
- **DENSE_RANK:** Eşitlik aynı sıra, sonraki atlanmaz (1, 1, 2, 3)
- **ROW_NUMBER:** Tüm sıralar unique (1, 2, 3, 4)

Banking: Aynı bakiye olan müşteriler için fair ranking → DENSE_RANK.

### 5. LEAD ve LAG — bir önceki / sonraki satır

```sql
SELECT 
    id, occurred_at, amount,
    LAG(amount) OVER (PARTITION BY account_id ORDER BY occurred_at) AS prev_amount,
    LEAD(amount) OVER (PARTITION BY account_id ORDER BY occurred_at) AS next_amount
FROM transactions;
```

**Banking örneği — running balance:**

```sql
SELECT 
    occurred_at, amount,
    SUM(amount) OVER (PARTITION BY account_id ORDER BY occurred_at) AS running_balance
FROM transactions;
```

| occurred_at | amount | running_balance |
|---|---|---|
| 2025-01-01 | +100 | 100 |
| 2025-01-02 | +50  | 150 |
| 2025-01-03 | -30  | 120 |
| 2025-01-04 | +200 | 320 |

Her transaction'ın o ana kadarki kümülatif bakiyesi. **Aggregate ile çok zor, window function ile şık.**

### 6. NTILE — gruplara böl

```sql
SELECT 
    id, balance_amount,
    NTILE(4) OVER (ORDER BY balance_amount) AS quartile
FROM accounts;
```

Tüm satırları **4 eşit gruba** böler (quartile). Her satırın hangi quartile'da olduğu.

Banking: Müşteri segmentation (gelir quartile'a göre).

### 7. FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
SELECT 
    id, occurred_at, amount,
    FIRST_VALUE(amount) OVER (
        PARTITION BY account_id ORDER BY occurred_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS first_transaction_amount,
    LAST_VALUE(amount) OVER (
        PARTITION BY account_id ORDER BY occurred_at
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) AS last_transaction_amount
FROM transactions;
```

**Frame önemli:** `LAST_VALUE` default frame'i `UNBOUNDED PRECEDING TO CURRENT ROW`, yani şu anki satıra kadar olan en son. Tam doğru için **`ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`** ekle.

### 8. Frame clause — ROWS vs RANGE

```sql
SUM(amount) OVER (
    PARTITION BY account_id ORDER BY occurred_at
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW   -- son 7 satır
)
```

vs

```sql
SUM(amount) OVER (
    PARTITION BY account_id ORDER BY occurred_at
    RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW   -- son 7 gün
)
```

- **ROWS:** Satır bazlı (fiziksel pencere)
- **RANGE:** Değer bazlı (mantıksal pencere)

Banking: "Son 7 gün toplam transaction" → RANGE.

### 9. CTE (Common Table Expression) — WITH clause

Sorguyu **adlandırılmış alt-sorgulara** böl.

```sql
WITH owner_totals AS (
    SELECT owner_id, SUM(balance_amount) AS total
    FROM accounts
    GROUP BY owner_id
),
top_owners AS (
    SELECT owner_id FROM owner_totals WHERE total > 1000000
)
SELECT a.* FROM accounts a 
WHERE a.owner_id IN (SELECT owner_id FROM top_owners);
```

**Faydaları:**
- Okunabilirlik (subquery yığını yerine adlı adımlar)
- Aynı CTE'ye birden fazla referans
- Recursive query'ler

**Tuzak (PostgreSQL <12):** CTE **materialize** edilir → optimizer plan'ı pessimize edebilir. PostgreSQL 12+ ile CTE inline edilir (default).

### 10. Recursive CTE — hiyerarşi

```sql
WITH RECURSIVE org_chart AS (
    -- Base case
    SELECT id, name, manager_id, 0 AS level
    FROM employees WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

**Banking örneği — şube hiyerarşisi:**

```sql
WITH RECURSIVE branch_tree AS (
    SELECT id, name, parent_branch_id, 0 AS depth FROM branches WHERE parent_branch_id IS NULL
    UNION ALL
    SELECT b.id, b.name, b.parent_branch_id, bt.depth + 1
    FROM branches b JOIN branch_tree bt ON b.parent_branch_id = bt.id
)
SELECT * FROM branch_tree;
```

Genel müdürlük → bölge → şube hiyerarşisi.

**Banking örneği — balance reconstruction:**

Bir hesabın geçmiş balance'ını her gün için yeniden hesapla (event sourcing pattern).

```sql
WITH RECURSIVE daily_balance AS (
    -- Initial state
    SELECT account_id, opened_at::date AS day, 
           initial_balance AS balance
    FROM accounts WHERE id = '...'
    
    UNION ALL
    
    -- Each day's change
    SELECT account_id, day + 1, 
           balance + COALESCE((
               SELECT SUM(amount) FROM transactions 
               WHERE account_id = '...' AND occurred_at::date = day + 1
           ), 0)
    FROM daily_balance
    WHERE day < CURRENT_DATE
)
SELECT * FROM daily_balance;
```

### 11. Oracle CONNECT BY — hierarchical query

Oracle'a özgü, recursive CTE'den önce vardı:

```sql
SELECT id, name, LEVEL
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR id = manager_id;
```

- `START WITH` — base case
- `CONNECT BY PRIOR` — recursive joining
- `LEVEL` — derinlik (built-in pseudo-column)

PostgreSQL'de **yok** — recursive CTE kullan.

Banking pratiği: Oracle code'unu PostgreSQL'e taşırken `CONNECT BY` → recursive CTE çevirisi gerekir.

### 12. MERGE statement — upsert

```sql
MERGE INTO accounts target
USING (
    SELECT '...' AS id, '...' AS owner_id, 'TRY' AS currency, 0 AS balance
) source
ON (target.id = source.id)
WHEN MATCHED THEN
    UPDATE SET 
        owner_id = source.owner_id,
        currency = source.currency
WHEN NOT MATCHED THEN
    INSERT (id, owner_id, currency, balance_amount)
    VALUES (source.id, source.owner_id, source.currency, source.balance);
```

**Atomik upsert** — varsa update, yoksa insert.

**PostgreSQL alternatifi (Postgres 9.5+):**

```sql
INSERT INTO accounts (id, owner_id, currency, balance_amount)
VALUES ('...', '...', 'TRY', 0)
ON CONFLICT (id) DO UPDATE SET
    owner_id = EXCLUDED.owner_id,
    currency = EXCLUDED.currency;
```

PostgreSQL `MERGE`'ü 15+ versiyondan beri destekler.

**Banking örneği — FX rate günlük update:**

```sql
MERGE INTO fx_rates t
USING daily_fx_feed s ON (t.from_currency = s.from_currency AND t.to_currency = s.to_currency)
WHEN MATCHED THEN UPDATE SET rate = s.rate, updated_at = s.feed_date
WHEN NOT MATCHED THEN INSERT VALUES (s.from_currency, s.to_currency, s.rate, s.feed_date);
```

Günde 1 kez TCMB feed'i çekilir, MERGE ile rates tablosu sync edilir.

### 13. PIVOT — satır → kolon

Bir kategoriye göre değerleri kolon yapma:

```sql
-- PostgreSQL (crosstab extension)
SELECT * FROM crosstab(
    'SELECT month, currency, SUM(amount) FROM transactions GROUP BY 1,2 ORDER BY 1,2',
    'SELECT DISTINCT currency FROM transactions ORDER BY 1'
) AS ct(month TEXT, USD NUMERIC, EUR NUMERIC, TRY NUMERIC);
```

Output:

| month | USD | EUR | TRY |
|---|---|---|---|
| 2025-01 | 1000 | 500 | 100000 |
| 2025-02 | 1200 | 600 | 110000 |

Banking: Currency bazlı aylık özet rapor.

### 14. JSON support — PostgreSQL gücü

```sql
SELECT 
    id,
    metadata->>'channel' AS channel,        -- text olarak
    (metadata->'fee'->>'amount')::numeric AS fee
FROM transactions;
```

Banking: ISO 20022 mesajları JSON olarak saklanabilir, gerektiğinde extract.

`@>` (contains), `?` (key exists), `jsonb_path_query` (path lookup) operatörleri.

### 15. CASE — conditional logic

```sql
SELECT 
    id,
    CASE 
        WHEN balance_amount < 0 THEN 'NEGATIVE'
        WHEN balance_amount = 0 THEN 'ZERO'
        WHEN balance_amount < 1000 THEN 'LOW'
        WHEN balance_amount < 10000 THEN 'MEDIUM'
        ELSE 'HIGH'
    END AS balance_tier
FROM accounts;
```

CASE expression aggregate ile birleşince güçlü:

```sql
SELECT 
    owner_id,
    SUM(CASE WHEN currency = 'TRY' THEN balance_amount ELSE 0 END) AS try_total,
    SUM(CASE WHEN currency = 'USD' THEN balance_amount ELSE 0 END) AS usd_total
FROM accounts
GROUP BY owner_id;
```

Manuel pivot — PIVOT operatörü olmayan DB'lerde standart.

### 16. Banking pattern'leri

#### Pattern 1: Top N per group

Her owner'ın en yüksek 3 hesabı:

```sql
WITH ranked AS (
    SELECT a.*, 
           ROW_NUMBER() OVER (PARTITION BY owner_id ORDER BY balance_amount DESC) rn
    FROM accounts a
)
SELECT * FROM ranked WHERE rn <= 3;
```

#### Pattern 2: Running balance / cumulative sum

```sql
SELECT 
    occurred_at, amount,
    SUM(amount) OVER (PARTITION BY account_id ORDER BY occurred_at) AS running_balance
FROM transactions;
```

#### Pattern 3: Day-over-day, month-over-month change

```sql
SELECT 
    day, daily_revenue,
    LAG(daily_revenue) OVER (ORDER BY day) AS prev_day,
    daily_revenue - LAG(daily_revenue) OVER (ORDER BY day) AS change,
    (daily_revenue - LAG(daily_revenue) OVER (ORDER BY day)) * 100.0 / 
        NULLIF(LAG(daily_revenue) OVER (ORDER BY day), 0) AS change_pct
FROM daily_revenue_summary;
```

#### Pattern 4: Gap detection

```sql
WITH numbered AS (
    SELECT 
        occurred_at,
        LAG(occurred_at) OVER (PARTITION BY account_id ORDER BY occurred_at) AS prev_time,
        occurred_at - LAG(occurred_at) OVER (PARTITION BY account_id ORDER BY occurred_at) AS gap
    FROM transactions
)
SELECT * FROM numbered WHERE gap > INTERVAL '1 day';
```

Bir hesapta 1 günden uzun **işlem yok dönemi** tespit.

#### Pattern 5: Median (window function ile)

```sql
SELECT 
    DISTINCT owner_id,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY balance_amount) 
        OVER (PARTITION BY owner_id) AS median_balance
FROM accounts;
```

`PERCENTILE_CONT(0.5)` median. `0.25` Q1, `0.75` Q3.

### 17. Anti-pattern'ler

**Anti-pattern 1: Self-JOIN ile running sum**

```sql
SELECT t1.id, t1.amount,
       (SELECT SUM(t2.amount) FROM transactions t2 
        WHERE t2.account_id = t1.account_id AND t2.occurred_at <= t1.occurred_at) AS running
FROM transactions t1;
```

Her satır için subquery → O(n²). Window function `SUM() OVER (...)` çok daha hızlı.

**Anti-pattern 2: Çok karmaşık CTE'lerle her şeyi parçalamak**

CTE okunur ama derinleştirince optimizer'ı zorlar. Bazen tek query daha hızlı.

**Anti-pattern 3: Recursive CTE termination unutmak**

```sql
WITH RECURSIVE infinite AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM infinite   -- ❌ stop condition yok
)
SELECT * FROM infinite;
```

Sonsuz → DB patlatır. **Her zaman `WHERE n < limit`** koy.

**Anti-pattern 4: `LAST_VALUE` default frame**

```sql
LAST_VALUE(amount) OVER (ORDER BY occurred_at)   -- ❌ default frame yanlış sonuç
```

Default `UNBOUNDED PRECEDING TO CURRENT ROW` — son değil, "şu ana kadar son". Explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` koy.

---

## Önemli olabilecek araştırma kaynakları

- "SQL Cookbook" (Anthony Molinaro) — gerçek SQL örnekleri
- PostgreSQL Window Functions tutorial
- "Modern SQL" by Markus Winand (modern-sql.com)
- Oracle SQL Reference — Analytic Functions
- "SQL Performance Explained" (Markus Winand) — CTE pessimization tartışması
- Use The Index, Luke — Pagination chapter (window function pagination)

---

## Mini task'ler

### Task 4.3.1 — Running balance (30 dk)

`core-banking`'de transaction'lar için running balance query yaz:

```sql
SELECT 
    occurred_at, amount, direction,
    SUM(CASE WHEN direction = 'CREDIT' THEN amount ELSE -amount END) 
        OVER (PARTITION BY account_id ORDER BY occurred_at) AS running_balance
FROM journal_lines
WHERE account_id = '...'
ORDER BY occurred_at;
```

10 transaction oluştur, running balance'ı doğrula.

### Task 4.3.2 — Top N per owner (30 dk)

Her owner'ın en yüksek 3 hesabını listele:

```sql
WITH ranked AS (
    SELECT a.*, 
           ROW_NUMBER() OVER (PARTITION BY owner_id ORDER BY balance_amount DESC) rn
    FROM accounts a
)
SELECT * FROM ranked WHERE rn <= 3;
```

100 owner × 5 hesap test data ile çalıştır.

### Task 4.3.3 — Recursive CTE: branch tree (45 dk)

Migration:

```sql
CREATE TABLE branches (
    id UUID PRIMARY KEY,
    name VARCHAR(100),
    parent_branch_id UUID REFERENCES branches(id)
);
```

3 seviye hiyerarşi insert et (HQ → bölge → şube). Recursive CTE ile tüm tree'yi çıkar, her node'un `depth`'i ile.

### Task 4.3.4 — Day-over-day change (30 dk)

Daily transaction summary tablosu:

```sql
CREATE TABLE daily_summary (
    summary_date DATE,
    total_amount NUMERIC,
    transaction_count INT
);
```

LAG ile day-over-day change ve percent change query yaz.

### Task 4.3.5 — MERGE / UPSERT (30 dk)

Daily FX feed simulate et:

```sql
CREATE TABLE fx_rates (
    from_currency CHAR(3),
    to_currency CHAR(3),
    rate NUMERIC(19, 8),
    updated_at TIMESTAMP,
    PRIMARY KEY (from_currency, to_currency)
);
```

İlk gün insert, ikinci gün **MERGE** ile update veya insert.

### Task 4.3.6 — Quartile segmentation (30 dk)

```sql
SELECT 
    owner_id,
    SUM(balance_amount) AS total_balance,
    NTILE(4) OVER (ORDER BY SUM(balance_amount) DESC) AS quartile
FROM accounts
GROUP BY owner_id;
```

Q1 (en zengin %25), Q4 (en az). Her quartile'ın ortalama bakiyesi.

---

## Test yazma rehberi

### Test 4.3.1 — Running balance correctness

```java
@Test
void runningBalanceShouldMatchManualCalculation() {
    UUID accountId = createAccountWithTransactions(
        new BigDecimal("100"),    // CREDIT
        new BigDecimal("50"),     // CREDIT  
        new BigDecimal("-30"),    // DEBIT
        new BigDecimal("200")     // CREDIT
    );
    
    List<BigDecimal> runningBalances = jdbcTemplate.queryForList("""
        SELECT SUM(CASE WHEN direction = 'CREDIT' THEN amount ELSE -amount END)
                   OVER (ORDER BY occurred_at) AS running
        FROM journal_lines WHERE account_id = ?
        ORDER BY occurred_at
        """, BigDecimal.class, accountId);
    
    assertThat(runningBalances).containsExactly(
        new BigDecimal("100"),
        new BigDecimal("150"),
        new BigDecimal("120"),
        new BigDecimal("320")
    );
}
```

### Test 4.3.2 — Top N per group

```java
@Test
void topThreeAccountsPerOwnerShouldBeReturned() {
    UUID owner = createOwner();
    createAccountWithBalance(owner, "1000");
    createAccountWithBalance(owner, "500");
    createAccountWithBalance(owner, "200");
    createAccountWithBalance(owner, "50");      // 4. bu kalmamalı
    
    List<BigDecimal> top3 = jdbcTemplate.queryForList("""
        WITH ranked AS (
            SELECT balance_amount,
                   ROW_NUMBER() OVER (PARTITION BY owner_id ORDER BY balance_amount DESC) rn
            FROM accounts WHERE owner_id = ?
        )
        SELECT balance_amount FROM ranked WHERE rn <= 3 ORDER BY balance_amount DESC
        """, BigDecimal.class, owner);
    
    assertThat(top3).hasSize(3);
    assertThat(top3.get(0)).isEqualByComparingTo("1000");
    assertThat(top3.get(2)).isEqualByComparingTo("200");
}
```

### Test 4.3.3 — Recursive CTE

```java
@Test
void branchTreeShouldShowHierarchy() {
    UUID hq = createBranch("HQ", null);
    UUID region = createBranch("Marmara", hq);
    UUID branch = createBranch("Beşiktaş", region);
    
    List<Map<String, Object>> tree = jdbcTemplate.queryForList("""
        WITH RECURSIVE branch_tree AS (
            SELECT id, name, 0 AS depth FROM branches WHERE parent_branch_id IS NULL
            UNION ALL
            SELECT b.id, b.name, bt.depth + 1
            FROM branches b JOIN branch_tree bt ON b.parent_branch_id = bt.id
        )
        SELECT name, depth FROM branch_tree ORDER BY depth
        """);
    
    assertThat(tree).hasSize(3);
    assertThat((Integer) tree.get(0).get("depth")).isEqualTo(0);
    assertThat((Integer) tree.get(2).get("depth")).isEqualTo(2);
}
```

---

## Claude-verify prompt

```
SQL window function ve advanced SQL çalışmamı banking-grade kriterlere göre 
değerlendir:

1. Window function kullanımı:
   - PARTITION BY, ORDER BY, frame clause doğru kullanılmış mı?
   - ROW_NUMBER, RANK, DENSE_RANK farkı bilinçli mi?
   - LEAD, LAG ile temporal analiz yapılmış mı?

2. Running balance:
   - SUM() OVER (PARTITION BY ... ORDER BY ...) ile self-join'sız implement mi?
   - DEBIT/CREDIT direction handling doğru mu?

3. CTE:
   - Karmaşık query CTE ile parçalanmış mı (okunabilirlik)?
   - Recursive CTE'de termination koşulu var mı?
   - PostgreSQL CTE inlining (12+) bilinçli mi?

4. MERGE / UPSERT:
   - Atomik upsert için MERGE veya ON CONFLICT DO UPDATE kullanılmış mı?
   - Race condition'a karşı dirençli mi?

5. Frame clause:
   - LAST_VALUE'da default frame tuzağı bilinçli mi?
   - ROWS vs RANGE arası karar doğru mu?

6. Banking-specific:
   - Top N per owner pattern kullanılmış mı?
   - Day-over-day change LAG ile yapılmış mı?
   - Quartile segmentation NTILE ile mi?
   - Hierarchy (branch tree) recursive CTE ile mi?

7. Anti-pattern:
   - Self-JOIN ile running sum yapılmış mı? (Olmamalı)
   - Recursive CTE termination eksik mi?
   - LAST_VALUE default frame yanlış kullanılmış mı?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] ROW_NUMBER, RANK, DENSE_RANK farkını biliyorum
- [ ] LEAD, LAG ile temporal değişim sorgusu yazdım
- [ ] Running balance window function ile çalışıyor
- [ ] Recursive CTE ile branch hierarchy implement ettim
- [ ] MERGE veya ON CONFLICT DO UPDATE ile atomik upsert yazdım
- [ ] Frame clause (ROWS vs RANGE) farkını biliyorum
- [ ] LAST_VALUE default frame tuzağına dikkat ediyorum
- [ ] NTILE ile segmentation yapabiliyorum
- [ ] CTE'leri karmaşık query'lerde okunabilirlik için kullanıyorum

---

## Defter notları

1. "Window function vs GROUP BY farkı: ____."
2. "ROW_NUMBER, RANK, DENSE_RANK üçü arasında ne zaman hangisi: ____."
3. "LEAD/LAG kullanarak running balance'tan farklı temporal analiz örnekleri: ____."
4. "ROWS vs RANGE frame farkı (banking örneği): ____."
5. "Recursive CTE termination'sız bırakırsam ne olur: ____."
6. "MERGE / ON CONFLICT DO UPDATE atomicity garantisi: ____."
7. "LAST_VALUE default frame tuzağı: ____."
8. "CTE inlining (PostgreSQL 12+) ile materialize farkı: ____."
9. "Oracle CONNECT BY → PostgreSQL recursive CTE çevirisi: ____."
10. "NTILE(4) ile customer segmentation banking kullanımı: ____."
