# Topic 4.4 — PL/SQL Programming

## Hedef

Oracle'ın procedural extension'ı PL/SQL'i banking-grade seviyede öğrenmek. Procedure, function, **package**, cursor, **BULK COLLECT + FORALL**, exception handling, autonomous transaction. TR bankalarında milyonlarca satırlık production PL/SQL kodu var — bunu okuyabilen ve yazabilen developer olmak.

## Süre

Okuma: 2.5 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6.5 saat

## Önbilgi

- Topic 4.1-4.3 bitti
- Oracle XE Docker'da çalışıyor
- SQL Developer veya `sqlplus` ile bağlanabiliyorsun

---

## Kavramlar

### 1. PL/SQL nedir, neden TR bankacılığında bu kadar yaygın

PL/SQL = **Procedural Language / SQL**. Oracle'ın SQL'e procedural extension'ı (1990'lar).

**Banking'de yaygın olma sebepleri:**

1. **Performance:** Data DB'de, logic DB'de — network round-trip yok
2. **Atomicity:** Tüm operasyon tek transaction'da
3. **Mature ekosistem:** TR bankalarının 90'larda yazdığı core banking PL/SQL hâlâ üretimde
4. **Stored procedure API:** Java tarafı sadece "interest_pkg.compute_daily(:date)" çağırır, complex logic DB'de
5. **Sıkı erişim kontrolü:** PL/SQL erişimi DBA tarafından kontrollü

**Anti-side:**
- Cross-DB taşınamaz (Oracle'a kilitler)
- Test edilmesi zor (test framework eski)
- Version control'da zor (DB'den schema export gerekir)
- Modern Java/Spring entegrasyonu daha çekici

**Banking gerçeği:** Eski core banking → PL/SQL ağırlıklı, yeni microservices → Java ağırlıklı. **Her iki dili de bilmek gerekiyor.**

### 2. PL/SQL blok yapısı

```sql
DECLARE
    -- variable declarations
    v_balance NUMBER;
    v_owner_id RAW(16);
BEGIN
    -- executable statements
    SELECT balance_amount INTO v_balance 
    FROM accounts WHERE id = :acc_id;
    
    IF v_balance < 100 THEN
        DBMS_OUTPUT.PUT_LINE('Low balance: ' || v_balance);
    END IF;
EXCEPTION
    -- exception handlers
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Account not found');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Unexpected error: ' || SQLERRM);
END;
/
```

Her PL/SQL block'un 3 bölümü: **DECLARE / BEGIN / EXCEPTION / END**.

**Anonymous block** — adsız, tek seferlik. `/` ile sonlandırılır (SQL*Plus convention).

### 3. Variable types

```sql
DECLARE
    v_count    NUMBER(10);
    v_name     VARCHAR2(100);
    v_today    DATE := SYSDATE;
    v_balance  accounts.balance_amount%TYPE;     -- table column type
    r_account  accounts%ROWTYPE;                  -- entire row type
    
    TYPE acc_rec_type IS RECORD (                 -- custom record
        id RAW(16),
        balance NUMBER
    );
    r_custom acc_rec_type;
BEGIN
    NULL;
END;
/
```

**`%TYPE`:** Table column'un tipinden türet. Tip değişirse otomatik adapt.
**`%ROWTYPE`:** Tüm row tipini al — `r_account.balance_amount`, `r_account.currency` gibi field'lar.

Banking pratiği: Her zaman `%TYPE` kullan, hard-coded tip yazma.

### 4. SQL içinde — implicit cursor

```sql
DECLARE
    v_balance NUMBER;
BEGIN
    SELECT balance_amount INTO v_balance 
    FROM accounts WHERE id = '...';
    
    -- v_balance kullan
END;
/
```

`SELECT ... INTO` **bir satır** döndürür. 0 satır → `NO_DATA_FOUND`. Birden fazla → `TOO_MANY_ROWS`.

### 5. Explicit cursor — birden fazla satır

```sql
DECLARE
    CURSOR c_active_accounts IS
        SELECT id, owner_id, balance_amount 
        FROM accounts 
        WHERE status = 'ACTIVE';
    
    r_acc c_active_accounts%ROWTYPE;
BEGIN
    OPEN c_active_accounts;
    LOOP
        FETCH c_active_accounts INTO r_acc;
        EXIT WHEN c_active_accounts%NOTFOUND;
        
        -- her satır için iş
        DBMS_OUTPUT.PUT_LINE(r_acc.id || ': ' || r_acc.balance_amount);
    END LOOP;
    CLOSE c_active_accounts;
END;
/
```

**FOR LOOP** ile daha kısa:

```sql
BEGIN
    FOR r_acc IN (SELECT id, balance_amount FROM accounts WHERE status = 'ACTIVE') LOOP
        DBMS_OUTPUT.PUT_LINE(r_acc.id || ': ' || r_acc.balance_amount);
    END LOOP;
END;
/
```

Cursor implicit açılır/kapanır, kolay ama her satır için round-trip. **Büyük dataset için yavaş.**

### 6. BULK COLLECT — toplu fetch

```sql
DECLARE
    TYPE acc_ids_type IS TABLE OF accounts.id%TYPE;
    v_ids acc_ids_type;
BEGIN
    SELECT id BULK COLLECT INTO v_ids
    FROM accounts WHERE status = 'ACTIVE';
    
    -- v_ids artık collection — array-like
    FOR i IN 1 .. v_ids.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_ids(i));
    END LOOP;
END;
/
```

**Tek round-trip, tüm satırlar memory'de.** 100k satır için cursor loop'tan **10-100x hızlı**.

**Tehlike:** Memory tüketimi. Çok büyük datasette `BULK COLLECT INTO v_ids LIMIT 1000` ile sayfala.

```sql
DECLARE
    TYPE acc_table IS TABLE OF accounts%ROWTYPE;
    v_accs acc_table;
    
    CURSOR c IS SELECT * FROM accounts;
BEGIN
    OPEN c;
    LOOP
        FETCH c BULK COLLECT INTO v_accs LIMIT 1000;
        EXIT WHEN v_accs.COUNT = 0;
        
        -- v_accs 1000 satır ile dolu
        FOR i IN 1 .. v_accs.COUNT LOOP
            -- her satır
        END LOOP;
    END LOOP;
    CLOSE c;
END;
/
```

### 7. FORALL — toplu DML

```sql
DECLARE
    TYPE id_list_type IS TABLE OF accounts.id%TYPE;
    v_ids id_list_type := id_list_type();
BEGIN
    v_ids.EXTEND(1000);
    -- v_ids doldur
    
    FORALL i IN 1 .. v_ids.COUNT
        UPDATE accounts SET balance_amount = balance_amount + 100 
        WHERE id = v_ids(i);
END;
/
```

**Tek SQL round-trip, 1000 UPDATE.** Loop ile UPDATE çok yavaş, FORALL çok hızlı.

Banking pattern: Günsonu bulk update'lerinde **mutlaka** FORALL.

### 8. Procedure ve Function

#### Procedure (return etmez)

```sql
CREATE OR REPLACE PROCEDURE close_inactive_accounts(
    p_inactive_days NUMBER,
    p_closed_count OUT NUMBER
) AS
BEGIN
    UPDATE accounts 
    SET status = 'CLOSED', closed_at = SYSTIMESTAMP
    WHERE status = 'ACTIVE' 
      AND opened_at < SYSTIMESTAMP - INTERVAL '1' DAY * p_inactive_days;
    
    p_closed_count := SQL%ROWCOUNT;
    COMMIT;
END;
/

-- Çağırım
DECLARE
    v_count NUMBER;
BEGIN
    close_inactive_accounts(365, v_count);
    DBMS_OUTPUT.PUT_LINE('Closed: ' || v_count);
END;
/
```

Parameter modes:
- **IN** (default): readonly, parameter olarak alır
- **OUT**: değer döndürür (caller'a)
- **IN OUT**: hem alır hem değiştirir

#### Function (return eder)

```sql
CREATE OR REPLACE FUNCTION calculate_interest(
    p_balance NUMBER,
    p_rate NUMBER,
    p_days NUMBER
) RETURN NUMBER AS
    v_interest NUMBER;
BEGIN
    v_interest := p_balance * p_rate * p_days / 365 / 100;
    RETURN ROUND(v_interest, 2);
END;
/

-- SQL içinde kullanım
SELECT id, balance_amount,
       calculate_interest(balance_amount, 8.5, 30) AS monthly_interest
FROM accounts;
```

Function'lar SQL içinde çağrılabilir (deterministic ise daha iyi).

### 9. Package — banking standardı

Package = ilgili procedure + function + variable'ların **container**'ı. Spring'de Service class'a benzer.

#### Specification (interface)

```sql
CREATE OR REPLACE PACKAGE interest_pkg AS
    
    -- public types
    TYPE interest_record IS RECORD (
        account_id RAW(16),
        amount NUMBER
    );
    
    -- public constants
    DEFAULT_RATE CONSTANT NUMBER := 8.5;
    
    -- public procedures
    PROCEDURE accrue_daily(p_business_date DATE);
    
    -- public functions
    FUNCTION calculate_interest(
        p_balance NUMBER, 
        p_rate NUMBER DEFAULT DEFAULT_RATE, 
        p_days NUMBER
    ) RETURN NUMBER;
    
END interest_pkg;
/
```

#### Body (implementation)

```sql
CREATE OR REPLACE PACKAGE BODY interest_pkg AS
    
    -- private helper
    FUNCTION compound_rate(p_rate NUMBER, p_periods NUMBER) RETURN NUMBER AS
    BEGIN
        RETURN POWER(1 + p_rate/100/365, p_periods) - 1;
    END;
    
    PROCEDURE accrue_daily(p_business_date DATE) IS
    BEGIN
        FOR r IN (SELECT id, balance_amount FROM accounts WHERE status = 'ACTIVE') LOOP
            DECLARE
                v_interest NUMBER;
            BEGIN
                v_interest := calculate_interest(r.balance_amount, DEFAULT_RATE, 1);
                
                INSERT INTO interest_postings(account_id, business_date, amount)
                VALUES (r.id, p_business_date, v_interest);
                
                UPDATE accounts 
                SET balance_amount = balance_amount + v_interest 
                WHERE id = r.id;
            END;
        END LOOP;
    END accrue_daily;
    
    FUNCTION calculate_interest(p_balance NUMBER, p_rate NUMBER, p_days NUMBER) RETURN NUMBER IS
    BEGIN
        RETURN ROUND(p_balance * p_rate * p_days / 365 / 100, 2);
    END;
    
END interest_pkg;
/
```

#### Çağırım

```sql
BEGIN
    interest_pkg.accrue_daily(SYSDATE);
END;
/

-- veya
SELECT interest_pkg.calculate_interest(10000, 8.5, 30) FROM dual;
```

#### Package state

Package'lar **session-level state** tutar. Bir session boyunca package variable'ları **kalıcı**.

```sql
CREATE OR REPLACE PACKAGE config_pkg AS
    g_active_currency VARCHAR2(3) := 'TRY';
END;
/

-- Session 1
BEGIN config_pkg.g_active_currency := 'USD'; END;
/
-- Aynı session'da bu değişiklik kalıcı

-- Session 2 — bağımsız state
```

Banking tuzağı: Session-bound state, aynı user'ın iki session'ı arasında **paylaşılmaz**. Connection pool'da farklı transaction farklı state görebilir.

### 10. Exception handling

```sql
DECLARE
    insufficient_funds EXCEPTION;
    PRAGMA EXCEPTION_INIT(insufficient_funds, -20001);   -- error code map
BEGIN
    -- ...
    IF v_balance < p_amount THEN
        RAISE_APPLICATION_ERROR(-20001, 'Insufficient funds');
    END IF;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20002, 'Account not found');
    WHEN insufficient_funds THEN
        ROLLBACK;
        RAISE;   -- üst seviyeye fırlat
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Unexpected: ' || SQLERRM || ' ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
        RAISE;
END;
/
```

**Built-in exception'lar:**
- `NO_DATA_FOUND` — SELECT INTO 0 satır
- `TOO_MANY_ROWS` — SELECT INTO >1 satır
- `DUP_VAL_ON_INDEX` — UNIQUE constraint violation
- `INVALID_NUMBER` — string → number conversion fail
- `ZERO_DIVIDE` — 0'a bölme

**Custom exception:** `EXCEPTION_INIT` ile error code map'le. `RAISE_APPLICATION_ERROR(code, message)` ile fırlat.

**Banking pratiği:** Error code'ları organize et. -20001..-20999 user-defined range. `error_codes_pkg` tutarsanız bakım kolay.

### 11. Autonomous transaction

```sql
CREATE OR REPLACE PROCEDURE audit_log(p_action VARCHAR2, p_user VARCHAR2) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO audit_log(action, "user", logged_at) 
    VALUES (p_action, p_user, SYSTIMESTAMP);
    COMMIT;   -- kendi transaction'ında commit
END;
/
```

**Kullanım:** Caller transaction rollback olsa bile audit log **kalmalı**.

```sql
BEGIN
    audit_log('TRANSFER_ATTEMPT', 'user-123');   -- kendi transaction
    
    transfer_pkg.execute(...);   -- caller transaction
    
    -- transfer rollback olursa audit kalır
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;   -- transfer rollback
        -- audit zaten commit oldu, kayıp yok
END;
/
```

**Banking standardı.** Java tarafında `@Transactional(propagation = REQUIRES_NEW)` ile aynı pattern.

### 12. Banking örneği — EOD Reconciliation Package

```sql
CREATE OR REPLACE PACKAGE eod_reconciliation_pkg AS
    
    PROCEDURE reconcile_balances(p_business_date DATE);
    
    FUNCTION get_mismatch_count(p_business_date DATE) RETURN NUMBER;
    
END eod_reconciliation_pkg;
/

CREATE OR REPLACE PACKAGE BODY eod_reconciliation_pkg AS
    
    PROCEDURE reconcile_balances(p_business_date DATE) IS
        TYPE mismatch_record IS RECORD (
            account_id RAW(16),
            stored_balance NUMBER,
            calculated_balance NUMBER,
            diff NUMBER
        );
        TYPE mismatch_table IS TABLE OF mismatch_record;
        v_mismatches mismatch_table;
        
    BEGIN
        -- Toplam debit/credit hesabı, journal_lines üzerinden
        SELECT account_id, stored_balance, calculated_balance,
               (stored_balance - calculated_balance) AS diff
        BULK COLLECT INTO v_mismatches
        FROM (
            SELECT 
                a.id AS account_id,
                a.balance_amount AS stored_balance,
                NVL(SUM(CASE 
                    WHEN jl.direction = 'CREDIT' THEN jl.amount 
                    WHEN jl.direction = 'DEBIT' THEN -jl.amount 
                END), 0) AS calculated_balance
            FROM accounts a
            LEFT JOIN journal_lines jl ON jl.account_id = a.id
            LEFT JOIN journal_entries je ON je.id = jl.journal_entry_id
                AND TRUNC(je.occurred_at) <= p_business_date
            GROUP BY a.id, a.balance_amount
        )
        WHERE stored_balance != calculated_balance;
        
        -- Mismatch'leri tabloya yaz
        FORALL i IN 1 .. v_mismatches.COUNT
            INSERT INTO reconciliation_mismatches(
                account_id, business_date, stored_balance, calculated_balance, diff_amount, created_at
            ) VALUES (
                v_mismatches(i).account_id, 
                p_business_date,
                v_mismatches(i).stored_balance, 
                v_mismatches(i).calculated_balance,
                v_mismatches(i).diff,
                SYSTIMESTAMP
            );
        
        -- Log
        audit_pkg.log('EOD_RECONCILE', 'Mismatches: ' || v_mismatches.COUNT);
        
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            audit_pkg.log('EOD_RECONCILE_ERROR', SQLERRM);
            RAISE;
    END reconcile_balances;
    
    FUNCTION get_mismatch_count(p_business_date DATE) RETURN NUMBER IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count
        FROM reconciliation_mismatches
        WHERE business_date = p_business_date;
        RETURN v_count;
    END;
    
END eod_reconciliation_pkg;
/
```

### 13. Java'dan PL/SQL çağırma

```java
@Repository
public class InterestRepository {
    
    private final JdbcTemplate jdbc;
    
    public BigDecimal calculateInterest(BigDecimal balance, BigDecimal rate, int days) {
        return jdbc.queryForObject(
            "SELECT interest_pkg.calculate_interest(?, ?, ?) FROM dual",
            BigDecimal.class, balance, rate, days
        );
    }
    
    public void runEodReconciliation(LocalDate businessDate) {
        SimpleJdbcCall call = new SimpleJdbcCall(jdbc)
            .withCatalogName("eod_reconciliation_pkg")
            .withProcedureName("reconcile_balances");
        
        Map<String, Object> params = Map.of("p_business_date", java.sql.Date.valueOf(businessDate));
        call.execute(params);
    }
}
```

### 14. Triggers — neden DİKKATLİ olmalısın

```sql
CREATE OR REPLACE TRIGGER trg_account_status_audit
AFTER UPDATE OF status ON accounts
FOR EACH ROW
BEGIN
    INSERT INTO account_status_audit(account_id, old_status, new_status, changed_at)
    VALUES (:OLD.id, :OLD.status, :NEW.status, SYSTIMESTAMP);
END;
/
```

**Trigger anti-pattern'leri:**
- Business logic'i trigger'a koymak (gizli, debug zor)
- Trigger içinde DML (cascade trigger fire)
- Cross-row mantık (mutating table error riski)

**Banking pratiği:** Trigger'ı **sadece audit** için kullan. Business logic application veya procedure'da.

### 15. Performance tips

1. **BULK COLLECT + FORALL** — loop yerine
2. **Cursor variable** (ref cursor) — generic procedures
3. **NOLOGGING** — direct insert için (audit'i kapatır)
4. **PARALLEL hint** — büyük tablolar
5. **PRAGMA UDF** — function SQL'den çağrılırken hızlı
6. **DETERMINISTIC** — same input → same output ise işaretle
7. **Result Cache** — `/*+ RESULT_CACHE */` veya `RESULT_CACHE` clause

### 16. Anti-pattern'ler

**Anti-pattern 1: WHEN OTHERS THEN NULL**

```sql
EXCEPTION
    WHEN OTHERS THEN NULL;   -- ❌ exception yutuluyor
END;
```

Tüm hataları görmezden gel → debug imkânsız.

**Çözüm:** En azından log + RAISE.

**Anti-pattern 2: Loop içinde tek satır DML**

```sql
FOR r IN c LOOP
    UPDATE accounts SET ... WHERE id = r.id;   -- ❌ 100k UPDATE round-trip
END LOOP;
```

**Çözüm:** Toplu collection oluştur, FORALL ile bulk update.

**Anti-pattern 3: %TYPE kullanmamak**

```sql
v_balance NUMBER(15, 4);   -- hardcoded
```

Tablo değişirse manuel sync. Hep `accounts.balance_amount%TYPE`.

**Anti-pattern 4: Commit/Rollback procedure içinde**

Çağıran kim olduğunu bilmiyorsun. `COMMIT` kullanırsan caller'ın transaction'ını **kaza ile sonlandırırsın**.

**Çözüm:** Procedure transaction yönetimi yapmasın — caller karar versin. Sadece **autonomous transaction**'larda commit/rollback procedure içinde OK.

**Anti-pattern 5: Trigger'da business logic**

Yukarıda anlatıldı.

---

## Önemli olabilecek araştırma kaynakları

- "Oracle PL/SQL Programming" (Steven Feuerstein) — referans kitap
- Oracle PL/SQL Language Reference (official docs)
- Tom Kyte (asktom.oracle.com) altın kaynak
- "Effective Oracle by Design" (Tom Kyte)
- Steven Feuerstein YouTube serisi
- "Oracle SQL Recipes" (kötü duruma çare)

---

## Mini task'ler

### Task 4.4.1 — Anonymous block: balance check (15 dk)

Bir account ID için bakiyeyi ekrana yazdıran anonymous PL/SQL block. `NO_DATA_FOUND` handle.

### Task 4.4.2 — Procedure: close inactive accounts (30 dk)

`close_inactive_accounts` procedure'unu yaz. OUT parameter ile kapanan count dön.

### Task 4.4.3 — Function: interest calculation (30 dk)

`calculate_interest(balance, rate, days)` function. SQL içinde çağırarak account'ların monthly interest'lerini görüntüle.

### Task 4.4.4 — Package: interest_pkg (60 dk)

Tam package yaz — spec + body. `accrue_daily` procedure'u BULK COLLECT + FORALL ile.

### Task 4.4.5 — Autonomous transaction: audit_log (30 dk)

`audit_log` procedure'u autonomous transaction ile. Caller'ı rollback yapsa bile audit kalmalı. Test et.

### Task 4.4.6 — EOD reconciliation package (60 dk)

Yukarıdaki `eod_reconciliation_pkg`'i yaz. 100 hesap, 1000 transaction test verisi ile çalıştır. Bir hesabın bakiyesini manuel bozarak mismatch oluştur, package yakaladığını doğrula.

### Task 4.4.7 — Java'dan çağırma (30 dk)

`core-banking`'in `JdbcTemplate` veya `SimpleJdbcCall` ile interest_pkg çağırma:

```java
@Test
void shouldCallInterestPackage() {
    BigDecimal result = repo.calculateInterest(
        new BigDecimal("10000"), new BigDecimal("8.5"), 30
    );
    assertThat(result).isEqualByComparingTo("69.86");
}
```

---

## Test yazma rehberi

### Test 4.4.1 — utPLSQL (PL/SQL testing framework)

```sql
-- utPLSQL paketi gerekir (Oracle community)
CREATE OR REPLACE PACKAGE test_interest_pkg AS
    --%suite(Interest Calculation Tests)
    
    --%test(Calculates simple interest)
    PROCEDURE test_simple_interest;
END;
/

CREATE OR REPLACE PACKAGE BODY test_interest_pkg AS
    PROCEDURE test_simple_interest IS
        v_result NUMBER;
    BEGIN
        v_result := interest_pkg.calculate_interest(10000, 8.5, 30);
        ut.expect(v_result).to_equal(69.86);
    END;
END;
/

-- Çalıştır
EXEC ut.run('test_interest_pkg');
```

### Test 4.4.2 — Integration test from Java

```java
@Test
@Sql("/sql/interest-test-data.sql")
void interestPackageShouldAccrueCorrectly() {
    LocalDate today = LocalDate.now();
    interestRepo.runDailyAccrual(today);
    
    BigDecimal posted = jdbc.queryForObject(
        "SELECT SUM(amount) FROM interest_postings WHERE business_date = ?",
        BigDecimal.class, java.sql.Date.valueOf(today)
    );
    
    assertThat(posted).isPositive();
}
```

---

## Claude-verify prompt

```
PL/SQL kodumu banking-grade kriterlere göre değerlendir:

1. Yapı:
   - Package spec + body ayrımı yapılmış mı?
   - Public/private metot ayrımı doğru mu (spec'te sadece public)?
   - %TYPE / %ROWTYPE kullanılmış mı (hardcoded tip yok)?

2. Performans:
   - Loop içinde tek-satır DML yerine BULK COLLECT + FORALL var mı?
   - Implicit cursor kullanımı verimli mi?
   - BULK COLLECT LIMIT ile memory tüketimi kontrol altında mı?

3. Exception handling:
   - WHEN OTHERS THEN NULL var mı? (Olmamalı)
   - Custom exception ile RAISE_APPLICATION_ERROR yapılmış mı?
   - DBMS_UTILITY.FORMAT_ERROR_BACKTRACE ile stack trace logging?

4. Transaction:
   - Procedure içinde COMMIT/ROLLBACK var mı? (Sadece autonomous'ta)
   - Autonomous transaction audit_log için kullanılmış mı?

5. Banking specific:
   - interest_pkg compound interest doğru hesaplıyor mu?
   - eod_reconciliation_pkg mismatch'leri bulup yazıyor mu?
   - Java tarafından SimpleJdbcCall ile çağrılabilir mi?

6. Anti-pattern:
   - Trigger'da business logic var mı?
   - Hardcoded literal'lar (rate, day) magic number mu?
   - Loop'larda performance felaket mi?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Anonymous block ile basit balance check yazabiliyorum
- [ ] Procedure ve function arası farkı biliyorum
- [ ] Package spec + body yazdım
- [ ] BULK COLLECT + FORALL ile bulk DML
- [ ] Autonomous transaction ile audit log
- [ ] Custom exception + RAISE_APPLICATION_ERROR
- [ ] interest_pkg + eod_reconciliation_pkg implementasyonu
- [ ] Java tarafından SimpleJdbcCall ile package çağırma
- [ ] WHEN OTHERS THEN NULL anti-pattern'inden kaçınıyorum
- [ ] Trigger'ı sadece audit için kullanmaya razıyım

---

## Defter notları

1. "PL/SQL TR bankacılığında neden bu kadar yaygın: ____."
2. "BULK COLLECT + FORALL ile loop+DML performans farkı: ____."
3. "Package state (session-level) tuzakları: ____."
4. "Autonomous transaction (PRAGMA) banking kullanımı: ____."
5. "WHEN OTHERS THEN NULL anti-pattern'i neden tehlikeli: ____."
6. "%TYPE / %ROWTYPE ile hardcoded tip karşılaştırması: ____."
7. "Procedure içinde COMMIT — ne zaman, ne zaman değil: ____."
8. "Trigger banking'de sadece ne için kullanılır: ____."
9. "Java tarafından PL/SQL package çağırma (SimpleJdbcCall): ____."
10. "RAISE_APPLICATION_ERROR error code range (-20001..-20999): ____."
