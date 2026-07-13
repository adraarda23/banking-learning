# Phase 4 Mini-Project — `core-banking` Oracle Migration & PL/SQL

## Hedef

`core-banking` projesini PostgreSQL'den **Oracle**'a migrate et. 3 production-grade PL/SQL package yaz (interest, EOD reconciliation, fraud check). 1M-satır raporlama query'sini partition + materialized view + index ile **50x+ hızlandır**. SKIP LOCKED ile distributed job worker pattern uygula. Deadlock + ORA-01555 hatasını **canlı reproduction** + fix.

Sonunda elinde: Oracle XE Docker'da çalışan, PL/SQL ile business logic'in bir kısmı DB'de yaşayan, partitioned + materialized view + optimized index'li tam Oracle banking servisi. JaCoCo coverage Phase 1'deki seviyenin altına düşmemiş.

## Süre

7-10 gün (günde 2-3 saat ciddi çalışma).

## Önbilgi

Phase 4'ün 6 topic'i bitti. Index, EXPLAIN, window functions, PL/SQL, Oracle-specific, locking biliyorsun. Phase 1-2-3'teki `core-banking` projen Postgres'te çalışıyor.

---

## Görev 1 — Oracle XE Docker setup (yarım gün)

### 1.1 Docker compose

`core-banking/docker-compose.oracle.yml`:

```yaml
services:
  oracle:
    image: container-registry.oracle.com/database/express:21.3.0-xe
    container_name: banking-oracle
    environment:
      ORACLE_PWD: BankingDev2024!
      ORACLE_CHARACTERSET: AL32UTF8
    ports:
      - "1521:1521"
    volumes:
      - oracle_data:/opt/oracle/oradata
    healthcheck:
      test: ["CMD-SHELL", "echo 'select 1 from dual;' | sqlplus -s sys/${ORACLE_PWD}@//localhost:1521/XE as sysdba"]
      interval: 30s
      timeout: 20s
      retries: 10

volumes:
  oracle_data:
```

```bash
docker compose -f docker-compose.oracle.yml up -d

# Bağlantı testi
docker exec -it banking-oracle bash -c "sqlplus sys/BankingDev2024!@//localhost:1521/XE as sysdba"
```

### 1.2 Banking schema yaratma

İlk girişte:

```sql
-- as sys
ALTER SESSION SET CONTAINER = XEPDB1;

CREATE USER banking_dev IDENTIFIED BY "BankingDev2024!";
GRANT CREATE SESSION, CREATE TABLE, CREATE SEQUENCE, CREATE PROCEDURE, 
      CREATE TRIGGER, CREATE VIEW, CREATE MATERIALIZED VIEW,
      UNLIMITED TABLESPACE TO banking_dev;
GRANT EXECUTE ON DBMS_LOCK TO banking_dev;
GRANT EXECUTE ON DBMS_MVIEW TO banking_dev;
```

### 1.3 Spring Boot config

`pom.xml`'a Oracle JDBC driver:

```xml
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc11</artifactId>
    <version>23.3.0.23.09</version>
</dependency>
```

`application-oracle.yml`:

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/XEPDB1
    username: banking_dev
    password: ${DB_PASSWORD:BankingDev2024!}
    driver-class-name: oracle.jdbc.OracleDriver
    hikari:
      maximum-pool-size: 20
      connection-timeout: 30000
      leak-detection-threshold: 30000
  jpa:
    database-platform: org.hibernate.dialect.OracleDialect
    properties:
      hibernate:
        default_schema: BANKING_DEV
        jdbc:
          batch_size: 50
        order_inserts: true
  flyway:
    locations: classpath:db/migration/oracle
    default-schema: BANKING_DEV
```

### 1.4 Deliverables

- [ ] `docker-compose.oracle.yml` çalışıyor
- [ ] `banking_dev` schema oluşturuldu
- [ ] Spring Boot `oracle` profile ile bağlanıyor
- [ ] Defterine: Oracle XE versiyonu, ek konfigürasyon notları, ilk girişte yaptığın PROBLEM ÇÖZÜMLERİ

---

## Görev 2 — Flyway Oracle migrations (1 gün)

### 2.1 V1: accounts table

`src/main/resources/db/migration/oracle/V1__create_accounts_table.sql`:

```sql
-- Account ID için sequence (CACHE 50 banking standardı)
CREATE SEQUENCE seq_account_id 
    START WITH 1 INCREMENT BY 1 
    CACHE 50 NOORDER NOCYCLE;

CREATE TABLE accounts (
    id              RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    owner_id        RAW(16) NOT NULL,
    currency        CHAR(3) NOT NULL,
    balance_amount  NUMBER(19, 4) DEFAULT 0 NOT NULL,
    status          VARCHAR2(20) DEFAULT 'ACTIVE' NOT NULL,
    opened_at       TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    closed_at       TIMESTAMP WITH TIME ZONE,
    version         NUMBER(19) DEFAULT 0 NOT NULL,
    created_by      VARCHAR2(100) DEFAULT 'system' NOT NULL,
    updated_by      VARCHAR2(100) DEFAULT 'system' NOT NULL,
    
    CONSTRAINT chk_acc_status CHECK (status IN ('ACTIVE', 'FROZEN', 'CLOSED')),
    CONSTRAINT chk_acc_currency CHECK (REGEXP_LIKE(currency, '^[A-Z]{3}$')),
    CONSTRAINT chk_acc_balance CHECK (balance_amount >= 0 OR status = 'CLOSED')
);

CREATE INDEX idx_acc_owner ON accounts(owner_id);
CREATE INDEX idx_acc_status_owner ON accounts(status, owner_id) WHERE status != 'CLOSED';

COMMENT ON TABLE accounts IS 'Customer bank accounts with double-entry ledger';
COMMENT ON COLUMN accounts.id IS 'UUID stored as RAW(16)';
COMMENT ON COLUMN accounts.balance_amount IS 'Maintained balance, authoritative from journal_lines';
COMMENT ON COLUMN accounts.version IS 'Optimistic locking via @Version';
```

### 2.2 V2: journal tables (partitioned)

`V2__create_journal_tables.sql`:

```sql
-- Journal entries — INTERVAL partitioned by month
CREATE TABLE journal_entries (
    id              RAW(16) DEFAULT SYS_GUID() NOT NULL,
    transaction_id  RAW(16) NOT NULL,
    description     VARCHAR2(500),
    occurred_at     TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    created_by      VARCHAR2(100) NOT NULL,
    
    CONSTRAINT pk_journal_entries PRIMARY KEY (id, occurred_at),
    CONSTRAINT uk_journal_tx UNIQUE (transaction_id, occurred_at)
)
PARTITION BY RANGE (occurred_at)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00 UTC'));

-- Journal lines — partitioned same as parent
CREATE TABLE journal_lines (
    id                  RAW(16) DEFAULT SYS_GUID() NOT NULL,
    journal_entry_id    RAW(16) NOT NULL,
    journal_occurred_at TIMESTAMP WITH TIME ZONE NOT NULL,  -- partition key
    account_id          RAW(16) NOT NULL,
    direction           VARCHAR2(6) NOT NULL,
    amount              NUMBER(19, 4) NOT NULL,
    currency            CHAR(3) NOT NULL,
    
    CONSTRAINT pk_journal_lines PRIMARY KEY (id, journal_occurred_at),
    CONSTRAINT chk_jl_direction CHECK (direction IN ('DEBIT', 'CREDIT')),
    CONSTRAINT chk_jl_amount CHECK (amount > 0)
)
PARTITION BY RANGE (journal_occurred_at)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00 UTC'));

-- Local indexes (her partition kendi index'i)
CREATE INDEX idx_jl_account ON journal_lines(account_id) LOCAL;
CREATE INDEX idx_jl_entry ON journal_lines(journal_entry_id) LOCAL;

-- Materialized view log (fast refresh için)
CREATE MATERIALIZED VIEW LOG ON journal_lines 
    WITH ROWID, SEQUENCE (account_id, direction, amount, currency, journal_occurred_at)
    INCLUDING NEW VALUES;
```

### 2.3 V3: idempotency keys

```sql
CREATE TABLE idempotency_keys (
    key             RAW(16) PRIMARY KEY,
    transfer_id     RAW(16) NOT NULL,
    request_hash    VARCHAR2(64) NOT NULL,
    response_status NUMBER(3) NOT NULL,
    response_body   CLOB NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    expires_at      TIMESTAMP WITH TIME ZONE DEFAULT (SYSTIMESTAMP + INTERVAL '24' HOUR) NOT NULL
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
```

### 2.4 V4: reconciliation tables

```sql
CREATE TABLE reconciliation_mismatches (
    id                  RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    business_date       DATE NOT NULL,
    account_id          RAW(16) NOT NULL,
    stored_balance      NUMBER(19, 4) NOT NULL,
    calculated_balance  NUMBER(19, 4) NOT NULL,
    diff_amount         NUMBER(19, 4) NOT NULL,
    detected_at         TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    resolved_at         TIMESTAMP WITH TIME ZONE,
    resolution_note     VARCHAR2(500),
    
    CONSTRAINT uk_recon_unique UNIQUE (business_date, account_id)
);

CREATE INDEX idx_recon_unresolved ON reconciliation_mismatches(business_date) 
    WHERE resolved_at IS NULL;

CREATE TABLE interest_postings (
    id              RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    account_id      RAW(16) NOT NULL,
    business_date   DATE NOT NULL,
    amount          NUMBER(19, 4) NOT NULL,
    rate            NUMBER(7, 4) NOT NULL,
    days            NUMBER(5) NOT NULL,
    posted_at       TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    
    CONSTRAINT uk_interest_unique UNIQUE (account_id, business_date)
);

CREATE TABLE audit_log (
    id          NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_type  VARCHAR2(50) NOT NULL,
    event_data  VARCHAR2(4000),
    logged_at   TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    logged_by   VARCHAR2(100) DEFAULT USER NOT NULL
);

CREATE INDEX idx_audit_logged_at ON audit_log(logged_at);
```

### 2.5 V5: fraud detection tables

```sql
CREATE TABLE fraud_alerts (
    id              RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    transaction_id  RAW(16) NOT NULL,
    account_id      RAW(16) NOT NULL,
    rule_code       VARCHAR2(50) NOT NULL,
    severity        VARCHAR2(10) NOT NULL,
    score           NUMBER(5, 2) NOT NULL,
    evidence        CLOB,
    detected_at     TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    
    CONSTRAINT chk_fraud_severity CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL'))
);

CREATE INDEX idx_fraud_account ON fraud_alerts(account_id, detected_at);
```

### 2.6 Validation: Flyway başarılı çalıştı mı?

```bash
mvn spring-boot:run -Dspring.profiles.active=oracle

# Check
docker exec -it banking-oracle sqlplus banking_dev/BankingDev2024!@//localhost:1521/XEPDB1
SQL> SELECT version, description, success FROM flyway_schema_history ORDER BY installed_rank;
```

5 migration başarılı olmalı.

### 2.7 Deliverables

- [ ] V1-V5 migration dosyaları yazıldı
- [ ] Tüm migration'lar Flyway tarafından başarıyla uygulandı
- [ ] `flyway_schema_history` tablosunda 5 success kaydı
- [ ] Defterine: PostgreSQL → Oracle değişen 8+ şey (UUID handling, currency CHECK, INTERVAL partitioning, sequence vs IDENTITY vs SYS_GUID, vb.)

---

## Görev 3 — `interest_pkg` PL/SQL package (1 gün)

### 3.1 Package specification

`src/main/resources/db/plsql/interest_pkg_spec.sql`:

```sql
CREATE OR REPLACE PACKAGE banking_dev.interest_pkg AS
    -- Constants
    DEFAULT_RATE CONSTANT NUMBER := 8.5;  -- Default annual rate
    MIN_BALANCE_FOR_ACCRUAL CONSTANT NUMBER := 100;
    
    -- Custom exceptions
    insufficient_balance EXCEPTION;
    PRAGMA EXCEPTION_INIT(insufficient_balance, -20100);
    
    invalid_business_date EXCEPTION;
    PRAGMA EXCEPTION_INIT(invalid_business_date, -20101);
    
    -- Public types
    TYPE accrual_summary IS RECORD (
        accounts_processed NUMBER,
        total_interest_posted NUMBER,
        skipped_accounts NUMBER,
        elapsed_seconds NUMBER
    );
    
    -- Public procedures
    PROCEDURE accrue_daily(p_business_date IN DATE);
    
    -- Public functions
    FUNCTION calculate_compound_interest(
        p_balance NUMBER,
        p_rate NUMBER DEFAULT DEFAULT_RATE,
        p_days NUMBER DEFAULT 1
    ) RETURN NUMBER;
    
    FUNCTION calculate_simple_interest(
        p_balance NUMBER,
        p_rate NUMBER DEFAULT DEFAULT_RATE,
        p_days NUMBER DEFAULT 1
    ) RETURN NUMBER;
    
    FUNCTION get_accrual_summary(p_business_date IN DATE) RETURN accrual_summary;
    
END interest_pkg;
/
```

### 3.2 Package body

`interest_pkg_body.sql`:

```sql
CREATE OR REPLACE PACKAGE BODY banking_dev.interest_pkg AS
    
    -- Private helper
    FUNCTION daily_rate(p_annual_rate NUMBER) RETURN NUMBER IS
    BEGIN
        RETURN p_annual_rate / 100 / 365;
    END;
    
    -- Simple interest: P * R * T (no compounding within period)
    FUNCTION calculate_simple_interest(
        p_balance NUMBER, 
        p_rate NUMBER, 
        p_days NUMBER
    ) RETURN NUMBER IS
    BEGIN
        IF p_balance IS NULL OR p_balance <= 0 THEN
            RETURN 0;
        END IF;
        IF p_rate IS NULL OR p_rate <= 0 THEN
            RETURN 0;
        END IF;
        IF p_days IS NULL OR p_days <= 0 THEN
            RETURN 0;
        END IF;
        
        RETURN ROUND(p_balance * (p_rate / 100) * (p_days / 365), 4);
    END;
    
    -- Compound interest: P * ((1 + r)^n - 1)
    FUNCTION calculate_compound_interest(
        p_balance NUMBER, 
        p_rate NUMBER, 
        p_days NUMBER
    ) RETURN NUMBER IS
        v_daily_rate NUMBER;
        v_compound_factor NUMBER;
    BEGIN
        IF p_balance IS NULL OR p_balance <= 0 THEN RETURN 0; END IF;
        IF p_rate IS NULL OR p_rate <= 0 THEN RETURN 0; END IF;
        IF p_days IS NULL OR p_days <= 0 THEN RETURN 0; END IF;
        
        v_daily_rate := daily_rate(p_rate);
        v_compound_factor := POWER(1 + v_daily_rate, p_days);
        
        RETURN ROUND(p_balance * (v_compound_factor - 1), 4);
    END;
    
    -- Main accrual procedure
    PROCEDURE accrue_daily(p_business_date IN DATE) IS
        TYPE acc_rec_type IS RECORD (
            id RAW(16),
            balance NUMBER,
            currency CHAR(3)
        );
        TYPE acc_table IS TABLE OF acc_rec_type;
        v_accounts acc_table;
        
        v_interest NUMBER;
        v_count_processed NUMBER := 0;
        v_total_posted NUMBER := 0;
        v_count_skipped NUMBER := 0;
        v_start_time TIMESTAMP := SYSTIMESTAMP;
        
        CURSOR c IS 
            SELECT id, balance_amount, currency 
            FROM accounts 
            WHERE status = 'ACTIVE'
              AND balance_amount >= MIN_BALANCE_FOR_ACCRUAL
              AND TRUNC(opened_at) < p_business_date
            ORDER BY id;
    BEGIN
        -- Pre-check: business date validation
        IF p_business_date > SYSDATE THEN
            RAISE_APPLICATION_ERROR(-20101, 'Cannot accrue for future date: ' || p_business_date);
        END IF;
        
        -- Idempotency: bu tarih için zaten posting var mı?
        DECLARE
            v_existing NUMBER;
        BEGIN
            SELECT COUNT(*) INTO v_existing 
            FROM interest_postings 
            WHERE business_date = p_business_date;
            
            IF v_existing > 0 THEN
                RAISE_APPLICATION_ERROR(-20102, 
                    'Accrual for date ' || p_business_date || ' already processed: ' || v_existing || ' postings');
            END IF;
        END;
        
        -- BULK COLLECT + FORALL pattern (Topic 4.4)
        OPEN c;
        LOOP
            FETCH c BULK COLLECT INTO v_accounts LIMIT 1000;
            EXIT WHEN v_accounts.COUNT = 0;
            
            -- Calculate interest per account
            FOR i IN 1 .. v_accounts.COUNT LOOP
                v_interest := calculate_compound_interest(
                    v_accounts(i).balance, 
                    DEFAULT_RATE, 
                    1
                );
                
                IF v_interest > 0 THEN
                    v_total_posted := v_total_posted + v_interest;
                    v_count_processed := v_count_processed + 1;
                ELSE
                    v_count_skipped := v_count_skipped + 1;
                END IF;
            END LOOP;
            
            -- Bulk insert interest postings
            FORALL i IN 1 .. v_accounts.COUNT
                INSERT INTO interest_postings(account_id, business_date, amount, rate, days)
                VALUES (
                    v_accounts(i).id,
                    p_business_date,
                    calculate_compound_interest(v_accounts(i).balance, DEFAULT_RATE, 1),
                    DEFAULT_RATE,
                    1
                );
            
            -- Bulk update account balances
            FORALL i IN 1 .. v_accounts.COUNT
                UPDATE accounts 
                SET balance_amount = balance_amount + 
                    calculate_compound_interest(v_accounts(i).balance, DEFAULT_RATE, 1),
                    updated_by = 'INTEREST_PKG'
                WHERE id = v_accounts(i).id;
        END LOOP;
        CLOSE c;
        
        -- Audit
        INSERT INTO audit_log(event_type, event_data)
        VALUES ('INTEREST_ACCRUED', 
                'Date: ' || p_business_date || 
                ', Processed: ' || v_count_processed || 
                ', Total: ' || v_total_posted ||
                ', Skipped: ' || v_count_skipped ||
                ', Elapsed: ' || ROUND(EXTRACT(SECOND FROM (SYSTIMESTAMP - v_start_time)), 2) || 's');
        
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            IF c%ISOPEN THEN CLOSE c; END IF;
            ROLLBACK;
            INSERT INTO audit_log(event_type, event_data)
            VALUES ('INTEREST_ERROR', 
                    'Date: ' || p_business_date || 
                    ', Error: ' || SQLERRM || 
                    ', Stack: ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
            COMMIT;
            RAISE;
    END accrue_daily;
    
    FUNCTION get_accrual_summary(p_business_date IN DATE) RETURN accrual_summary IS
        v_summary accrual_summary;
    BEGIN
        SELECT COUNT(*), NVL(SUM(amount), 0), 0, 0
        INTO v_summary.accounts_processed, v_summary.total_interest_posted,
             v_summary.skipped_accounts, v_summary.elapsed_seconds
        FROM interest_postings 
        WHERE business_date = p_business_date;
        
        RETURN v_summary;
    END;
    
END interest_pkg;
/
```

### 3.3 Java entegrasyonu

`InterestRepository.java`:

```java
@Repository
public class InterestRepository {
    
    private final JdbcTemplate jdbc;
    
    public InterestRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }
    
    public void runDailyAccrual(LocalDate businessDate) {
        SimpleJdbcCall call = new SimpleJdbcCall(jdbc)
            .withCatalogName("INTEREST_PKG")
            .withProcedureName("ACCRUE_DAILY")
            .withSchemaName("BANKING_DEV");
        
        Map<String, Object> params = Map.of(
            "P_BUSINESS_DATE", java.sql.Date.valueOf(businessDate)
        );
        
        try {
            call.execute(params);
        } catch (DataAccessException e) {
            if (e.getMessage().contains("ORA-20102")) {
                throw new AlreadyProcessedException("Accrual already done for " + businessDate);
            }
            throw e;
        }
    }
    
    public BigDecimal calculateCompoundInterest(BigDecimal balance, BigDecimal rate, int days) {
        return jdbc.queryForObject(
            "SELECT BANKING_DEV.INTEREST_PKG.CALCULATE_COMPOUND_INTEREST(?, ?, ?) FROM DUAL",
            BigDecimal.class, balance, rate, days
        );
    }
    
    public AccrualSummary getSummary(LocalDate businessDate) {
        return jdbc.queryForObject("""
            SELECT 
                COUNT(*) AS accounts_processed,
                NVL(SUM(amount), 0) AS total_interest
            FROM BANKING_DEV.interest_postings 
            WHERE business_date = ?
            """, (rs, n) -> new AccrualSummary(
                rs.getInt("accounts_processed"),
                rs.getBigDecimal("total_interest")
            ), java.sql.Date.valueOf(businessDate));
    }
}
```

### 3.4 Test

`InterestRepositoryIT.java`:

```java
@SpringBootTest
@ActiveProfiles("oracle")
@Testcontainers
class InterestRepositoryIT {
    
    @Autowired InterestRepository interestRepo;
    @Autowired AccountRepository accountRepo;
    
    @Test
    @Sql("/test-data/100-active-accounts.sql")
    void shouldAccrueInterestForActiveAccounts() {
        LocalDate businessDate = LocalDate.now().minusDays(1);
        
        interestRepo.runDailyAccrual(businessDate);
        
        AccrualSummary summary = interestRepo.getSummary(businessDate);
        assertThat(summary.accountsProcessed()).isEqualTo(100);
        assertThat(summary.totalInterest()).isPositive();
    }
    
    @Test
    @Sql("/test-data/100-active-accounts.sql")
    void shouldRejectDuplicateAccrual() {
        LocalDate businessDate = LocalDate.now().minusDays(1);
        
        interestRepo.runDailyAccrual(businessDate);
        
        assertThatThrownBy(() -> interestRepo.runDailyAccrual(businessDate))
            .isInstanceOf(AlreadyProcessedException.class);
    }
    
    @Test
    void calculateCompoundInterestShouldMatchExpected() {
        // 10000 * (1 + 0.085/365)^30 - 10000 ≈ 70.04
        BigDecimal result = interestRepo.calculateCompoundInterest(
            new BigDecimal("10000.00"), 
            new BigDecimal("8.5"), 
            30
        );
        
        assertThat(result).isCloseTo(new BigDecimal("70.04"), 
            within(new BigDecimal("0.10")));
    }
    
    @Test
    void shouldRejectFutureBusinessDate() {
        LocalDate future = LocalDate.now().plusDays(1);
        
        assertThatThrownBy(() -> interestRepo.runDailyAccrual(future))
            .isInstanceOf(DataAccessException.class)
            .hasMessageContaining("ORA-20101");
    }
}
```

### 3.5 Deliverables

- [ ] `interest_pkg` spec + body deploy edildi
- [ ] Java tarafından `SimpleJdbcCall` ile çağrılabiliyor
- [ ] Idempotency: aynı businessDate'te iki kere çağrım → exception
- [ ] Bulk insert + update FORALL ile (loop+DML değil)
- [ ] 4 integration test geçiyor
- [ ] Defterine: BULK COLLECT LIMIT 1000 ile memory yönetimi, autonomous transaction kararı (audit için kullansam mı?)

---

## Görev 4 — `eod_reconciliation_pkg` (1 gün)

Topic 4.4'teki tam package'i implement et. Mismatch detection + bulk insert + audit pattern.

```sql
CREATE OR REPLACE PACKAGE banking_dev.eod_reconciliation_pkg AS
    
    PROCEDURE reconcile_balances(p_business_date IN DATE);
    
    FUNCTION get_mismatch_count(p_business_date IN DATE) RETURN NUMBER;
    
    PROCEDURE resolve_mismatch(
        p_mismatch_id IN RAW, 
        p_note IN VARCHAR2
    );
    
END eod_reconciliation_pkg;
/

CREATE OR REPLACE PACKAGE BODY banking_dev.eod_reconciliation_pkg AS
    
    PROCEDURE reconcile_balances(p_business_date IN DATE) IS
        TYPE mismatch_record IS RECORD (
            account_id          RAW(16),
            stored_balance      NUMBER,
            calculated_balance  NUMBER
        );
        TYPE mismatch_table IS TABLE OF mismatch_record;
        v_mismatches mismatch_table;
        v_count NUMBER;
        v_start TIMESTAMP := SYSTIMESTAMP;
    BEGIN
        -- Authoritative balance from journal_lines vs stored balance
        SELECT account_id, stored_balance, calculated_balance
        BULK COLLECT INTO v_mismatches
        FROM (
            SELECT 
                a.id AS account_id,
                a.balance_amount AS stored_balance,
                NVL((
                    SELECT SUM(CASE 
                        WHEN jl.direction = 'CREDIT' THEN jl.amount 
                        WHEN jl.direction = 'DEBIT' THEN -jl.amount 
                    END)
                    FROM journal_lines jl
                    WHERE jl.account_id = a.id
                      AND jl.journal_occurred_at <= p_business_date + INTERVAL '1' DAY
                ), 0) AS calculated_balance
            FROM accounts a
            WHERE a.status != 'CLOSED'
        )
        WHERE stored_balance != calculated_balance;
        
        v_count := v_mismatches.COUNT;
        
        -- Bulk insert mismatches
        FORALL i IN 1 .. v_count
            INSERT INTO reconciliation_mismatches(
                business_date, account_id, stored_balance, 
                calculated_balance, diff_amount
            ) VALUES (
                p_business_date,
                v_mismatches(i).account_id,
                v_mismatches(i).stored_balance,
                v_mismatches(i).calculated_balance,
                v_mismatches(i).stored_balance - v_mismatches(i).calculated_balance
            );
        
        -- Audit
        INSERT INTO audit_log(event_type, event_data)
        VALUES ('EOD_RECONCILIATION',
                'Date: ' || p_business_date || 
                ', Mismatches: ' || v_count ||
                ', Elapsed: ' || ROUND(EXTRACT(SECOND FROM (SYSTIMESTAMP - v_start)), 2));
        
        COMMIT;
    EXCEPTION
        WHEN DUP_VAL_ON_INDEX THEN
            ROLLBACK;
            INSERT INTO audit_log(event_type, event_data)
            VALUES ('EOD_RECONCILIATION_DUPLICATE', 'Date already reconciled: ' || p_business_date);
            COMMIT;
        WHEN OTHERS THEN
            ROLLBACK;
            INSERT INTO audit_log(event_type, event_data)
            VALUES ('EOD_RECONCILIATION_ERROR', SQLERRM || ' Stack: ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
            COMMIT;
            RAISE;
    END reconcile_balances;
    
    FUNCTION get_mismatch_count(p_business_date IN DATE) RETURN NUMBER IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count
        FROM reconciliation_mismatches
        WHERE business_date = p_business_date
          AND resolved_at IS NULL;
        RETURN v_count;
    END;
    
    PROCEDURE resolve_mismatch(p_mismatch_id IN RAW, p_note IN VARCHAR2) IS
    BEGIN
        UPDATE reconciliation_mismatches 
        SET resolved_at = SYSTIMESTAMP, resolution_note = p_note
        WHERE id = p_mismatch_id;
        
        IF SQL%ROWCOUNT = 0 THEN
            RAISE_APPLICATION_ERROR(-20200, 'Mismatch not found: ' || RAWTOHEX(p_mismatch_id));
        END IF;
        
        INSERT INTO audit_log(event_type, event_data)
        VALUES ('MISMATCH_RESOLVED', 'ID: ' || RAWTOHEX(p_mismatch_id) || ', Note: ' || p_note);
        
        COMMIT;
    END;
    
END eod_reconciliation_pkg;
/
```

### Test: Kasten boz, yakala

```java
@Test
@Sql("/test-data/1000-accounts.sql")
void shouldDetectIntentionalBalanceMismatch() {
    // 5 hesabın stored balance'ını manuel boz
    jdbc.update("""
        UPDATE BANKING_DEV.accounts 
        SET balance_amount = balance_amount + 1000 
        WHERE ROWNUM <= 5
        """);
    
    // EOD recon çalıştır
    eodRecon.run(LocalDate.now());
    
    // 5 mismatch beklenir
    Integer count = jdbc.queryForObject(
        "SELECT COUNT(*) FROM BANKING_DEV.reconciliation_mismatches WHERE business_date = ?",
        Integer.class, java.sql.Date.valueOf(LocalDate.now())
    );
    assertThat(count).isEqualTo(5);
}
```

### Deliverables

- [ ] `eod_reconciliation_pkg` deploy
- [ ] Kasten 5 mismatch yarat → package 5'ini de yakaladı mı?
- [ ] Bulk insert FORALL pattern
- [ ] Resolve workflow (operator manual fix)
- [ ] Defterine: Authoritative source — accounts.balance_amount mı, SUM(journal_lines) mı? Senin kararın, gerekçesi

---

## Görev 5 — `fraud_check_pkg` (1 gün)

4 fraud rule + window function pattern.

```sql
CREATE OR REPLACE PACKAGE banking_dev.fraud_check_pkg AS
    PROCEDURE run_rules(p_lookback_minutes IN NUMBER DEFAULT 60);
    FUNCTION get_high_risk_count RETURN NUMBER;
END fraud_check_pkg;
/

CREATE OR REPLACE PACKAGE BODY banking_dev.fraud_check_pkg AS
    
    -- Rule 1: Same account 5+ transactions in 1 minute
    PROCEDURE rule_high_frequency(p_since IN TIMESTAMP) IS
    BEGIN
        INSERT INTO fraud_alerts(transaction_id, account_id, rule_code, severity, score, evidence)
        SELECT 
            SYS_GUID(),
            account_id,
            'HIGH_FREQUENCY',
            CASE 
                WHEN tx_count >= 10 THEN 'CRITICAL'
                WHEN tx_count >= 7 THEN 'HIGH'
                ELSE 'MEDIUM'
            END,
            LEAST(tx_count * 10, 100),
            'Tx count: ' || tx_count || ' in 1 min window'
        FROM (
            SELECT 
                jl.account_id,
                COUNT(*) AS tx_count,
                MIN(je.occurred_at) AS first_tx,
                MAX(je.occurred_at) AS last_tx
            FROM journal_lines jl
            JOIN journal_entries je ON je.id = jl.journal_entry_id 
                AND je.occurred_at = jl.journal_occurred_at
            WHERE je.occurred_at > p_since
              AND jl.direction = 'DEBIT'
            GROUP BY jl.account_id, 
                     TRUNC(je.occurred_at, 'MI')
            HAVING COUNT(*) >= 5
        )
        WHERE NOT EXISTS (
            SELECT 1 FROM fraud_alerts fa 
            WHERE fa.account_id = account_id 
              AND fa.rule_code = 'HIGH_FREQUENCY'
              AND fa.detected_at > p_since
        );
    END;
    
    -- Rule 2: Cumulative debit > 10000 in 24h
    PROCEDURE rule_large_cumulative(p_since IN TIMESTAMP) IS
    BEGIN
        INSERT INTO fraud_alerts(transaction_id, account_id, rule_code, severity, score, evidence)
        SELECT 
            SYS_GUID(),
            account_id,
            'LARGE_CUMULATIVE_24H',
            'HIGH',
            70,
            'Total debit 24h: ' || total_debit
        FROM (
            SELECT 
                jl.account_id,
                SUM(jl.amount) AS total_debit
            FROM journal_lines jl
            JOIN journal_entries je ON je.id = jl.journal_entry_id
            WHERE je.occurred_at > SYSTIMESTAMP - INTERVAL '24' HOUR
              AND jl.direction = 'DEBIT'
            GROUP BY jl.account_id
            HAVING SUM(jl.amount) >= 10000
        );
    END;
    
    -- Rule 3-4: ATM rapid withdraw, first foreign transfer (Skipped for brevity)
    
    PROCEDURE run_rules(p_lookback_minutes IN NUMBER) IS
        v_since TIMESTAMP := SYSTIMESTAMP - NUMTODSINTERVAL(p_lookback_minutes, 'MINUTE');
        v_alert_count NUMBER;
    BEGIN
        rule_high_frequency(v_since);
        rule_large_cumulative(v_since);
        
        SELECT COUNT(*) INTO v_alert_count 
        FROM fraud_alerts 
        WHERE detected_at > v_since;
        
        INSERT INTO audit_log(event_type, event_data)
        VALUES ('FRAUD_RULES_RUN', 'Window: ' || p_lookback_minutes || 'm, Alerts: ' || v_alert_count);
        
        COMMIT;
    END;
    
    FUNCTION get_high_risk_count RETURN NUMBER IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count 
        FROM fraud_alerts 
        WHERE severity IN ('HIGH', 'CRITICAL')
          AND detected_at > SYSTIMESTAMP - INTERVAL '24' HOUR;
        RETURN v_count;
    END;
END;
/
```

### Test

100 transaction kasten yarat aynı karttan 1 dk içinde → rule_high_frequency yakalamalı.

### Deliverables

- [ ] 2 rule implement (high frequency, large cumulative)
- [ ] Window function pattern (Topic 4.3)
- [ ] Idempotency NOT EXISTS check
- [ ] Test: 100 tx 1dk → HIGH_FREQUENCY alert
- [ ] Defterine: Banking fraud rules — kuralı ne kadar agresif ayarlarsan false positive ↑, kaçırılan dolandırıcılık ↓

---

## Görev 6 — 1M-row reporting optimization (1 gün)

### 6.1 Test data yükle

```sql
-- 100 owner, 10000 transaction her birinde
INSERT INTO accounts(id, owner_id, currency, balance_amount, status)
SELECT SYS_GUID(), SYS_GUID(), 'TRY', DBMS_RANDOM.VALUE(1000, 100000), 'ACTIVE'
FROM DUAL CONNECT BY LEVEL <= 100;

-- 1M transaction
INSERT /*+ APPEND */ INTO journal_lines(id, journal_entry_id, journal_occurred_at, account_id, direction, amount, currency)
SELECT 
    SYS_GUID(), SYS_GUID(),
    SYSTIMESTAMP - DBMS_RANDOM.VALUE(0, 365),
    (SELECT id FROM (SELECT id FROM accounts ORDER BY DBMS_RANDOM.VALUE) WHERE ROWNUM = 1),
    CASE WHEN DBMS_RANDOM.VALUE(0, 1) > 0.5 THEN 'DEBIT' ELSE 'CREDIT' END,
    DBMS_RANDOM.VALUE(10, 5000),
    'TRY'
FROM DUAL CONNECT BY LEVEL <= 1000000;

COMMIT;

EXEC DBMS_STATS.GATHER_TABLE_STATS('BANKING_DEV', 'JOURNAL_LINES', cascade => TRUE);
```

### 6.2 Yavaş sorgu

```sql
-- Müşterilerin son 30 günlük net hareketi
SELECT 
    a.owner_id,
    SUM(CASE WHEN jl.direction = 'CREDIT' THEN jl.amount ELSE -jl.amount END) AS net_movement
FROM accounts a
JOIN journal_lines jl ON jl.account_id = a.id
JOIN journal_entries je ON je.id = jl.journal_entry_id
WHERE je.occurred_at > SYSDATE - 30
GROUP BY a.owner_id;
```

### 6.3 Adım 1: Baseline (index'siz)

```sql
EXPLAIN PLAN FOR ... ;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

SET TIMING ON
-- run query
SET TIMING OFF
```

**Bekle:** Full table scan, ~30-60 sn.

### 6.4 Adım 2: Composite index

```sql
CREATE INDEX idx_jl_acc_date ON journal_lines(account_id, journal_occurred_at) LOCAL;
```

EXPLAIN PLAN tekrar. Plan'da `INDEX RANGE SCAN` görmeli. Sürede ölç.

**Bekle:** 5-10 sn.

### 6.5 Adım 3: Materialized view

```sql
CREATE MATERIALIZED VIEW mv_owner_monthly_summary
BUILD IMMEDIATE
REFRESH FAST ON DEMAND
ENABLE QUERY REWRITE
AS
SELECT 
    a.owner_id,
    TRUNC(je.occurred_at, 'MONTH') AS month,
    SUM(CASE WHEN jl.direction = 'CREDIT' THEN jl.amount ELSE -jl.amount END) AS net_amount,
    COUNT(*) AS tx_count
FROM accounts a
JOIN journal_lines jl ON jl.account_id = a.id
JOIN journal_entries je ON je.id = jl.journal_entry_id
GROUP BY a.owner_id, TRUNC(je.occurred_at, 'MONTH');

EXEC DBMS_MVIEW.REFRESH('mv_owner_monthly_summary', 'C');
```

Reporting endpoint MV'den okur. **Bekle:** <100ms.

### 6.6 Defterine — comparison table

| Adım | Plan | Süre |
|---|---|---|
| Baseline | Full Table Scan | __ sn |
| +Index | Index Range Scan | __ sn |
| +MV | MView Query Rewrite | __ ms |

Hızlanma çarpanı: ?x

---

## Görev 7 — SKIP LOCKED job worker (1 gün)

```sql
CREATE TABLE BANKING_DEV.transfer_jobs (
    id          RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    payload     CLOB NOT NULL,
    status      VARCHAR2(20) DEFAULT 'PENDING' NOT NULL,
    priority    NUMBER(2) DEFAULT 5,
    created_at  TIMESTAMP DEFAULT SYSTIMESTAMP,
    started_at  TIMESTAMP,
    completed_at TIMESTAMP,
    worker_id   VARCHAR2(100),
    
    CONSTRAINT chk_job_status CHECK (status IN ('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED'))
);

CREATE INDEX idx_jobs_pending 
    ON BANKING_DEV.transfer_jobs(status, priority DESC, created_at) 
    WHERE status = 'PENDING';
```

Java worker:

```java
@Component
public class TransferJobWorker {
    
    private final JdbcTemplate jdbc;
    private final String workerId;
    private final TransactionTemplate txTemplate;
    
    public TransferJobWorker(JdbcTemplate jdbc, PlatformTransactionManager txManager) {
        this.jdbc = jdbc;
        this.workerId = "worker-" + UUID.randomUUID().toString().substring(0, 8);
        this.txTemplate = new TransactionTemplate(txManager);
        this.txTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }
    
    public Optional<TransferJob> takeNext() {
        return txTemplate.execute(status -> {
            return jdbc.query("""
                SELECT id, payload FROM BANKING_DEV.transfer_jobs 
                WHERE status = 'PENDING'
                ORDER BY priority DESC, created_at
                FOR UPDATE SKIP LOCKED 
                FETCH FIRST 1 ROWS ONLY
                """, rs -> {
                    if (!rs.next()) return Optional.<TransferJob>empty();
                    
                    UUID jobId = UUID.nameUUIDFromBytes(rs.getBytes("id"));
                    String payload = rs.getString("payload");
                    
                    jdbc.update("""
                        UPDATE BANKING_DEV.transfer_jobs 
                        SET status = 'PROCESSING', started_at = SYSTIMESTAMP, worker_id = ?
                        WHERE id = ?
                        """, workerId, rs.getBytes("id"));
                    
                    return Optional.of(new TransferJob(jobId, payload));
                });
        });
    }
    
    public void markCompleted(UUID jobId) {
        jdbc.update("""
            UPDATE BANKING_DEV.transfer_jobs 
            SET status = 'COMPLETED', completed_at = SYSTIMESTAMP 
            WHERE id = ?
            """, idToRaw(jobId));
    }
}
```

### Test: 100 job + 10 worker

```java
@Test
void tenWorkersShouldDistributeJobsFairly() throws InterruptedException {
    // Insert 100 jobs
    for (int i = 0; i < 100; i++) {
        jdbc.update("INSERT INTO BANKING_DEV.transfer_jobs(payload) VALUES (?)", 
            "{\"job\": " + i + "}");
    }
    
    // 10 worker thread
    ExecutorService pool = Executors.newFixedThreadPool(10);
    AtomicInteger processed = new AtomicInteger(0);
    Map<String, AtomicInteger> perWorker = new ConcurrentHashMap<>();
    
    CountDownLatch done = new CountDownLatch(100);
    
    for (int i = 0; i < 10; i++) {
        pool.submit(() -> {
            while (true) {
                Optional<TransferJob> job = worker.takeNext();
                if (job.isEmpty()) break;
                
                String wId = currentWorkerId();
                perWorker.computeIfAbsent(wId, k -> new AtomicInteger()).incrementAndGet();
                
                processJob(job.get());
                processed.incrementAndGet();
                done.countDown();
            }
        });
    }
    
    done.await(30, TimeUnit.SECONDS);
    pool.shutdown();
    
    assertThat(processed).hasValue(100);
    
    // Her worker yaklaşık eşit pay (5-15 job arası)
    perWorker.values().forEach(count -> 
        assertThat(count.get()).isBetween(5, 20));
}
```

---

## Görev 8 — Kasten kırma: Deadlock + ORA-01555 (yarım gün)

### Deadlock reproduction

İki SQL session:

```sql
-- Session 1
SET AUTOCOMMIT OFF;
SELECT * FROM accounts WHERE id = (SELECT id FROM accounts WHERE ROWNUM = 1) FOR UPDATE;

-- Session 2 (parallel)
SELECT * FROM accounts WHERE id = (SELECT id FROM accounts WHERE ROWNUM = 2) FOR UPDATE;

-- Session 1
SELECT * FROM accounts WHERE id = (SELECT id FROM accounts WHERE ROWNUM = 2) FOR UPDATE;
-- waits...

-- Session 2
SELECT * FROM accounts WHERE id = (SELECT id FROM accounts WHERE ROWNUM = 1) FOR UPDATE;
-- ORA-00060: deadlock detected
```

**Defterine:** Hangi session ölür? Lock graph nasıl çözüldü?

### ORA-01555 reproduction

```sql
-- as sys
ALTER SYSTEM SET UNDO_RETENTION = 30;  -- 30 sn (çok düşük)

-- Session 1: long cursor
DECLARE
    CURSOR c IS SELECT * FROM journal_lines;   -- 1M satır
    v_record journal_lines%ROWTYPE;
BEGIN
    OPEN c;
    DBMS_LOCK.SLEEP(60);  -- 60 sn bekle, undo segment overwrite olsun
    LOOP
        FETCH c INTO v_record;
        EXIT WHEN c%NOTFOUND;
    END LOOP;
    CLOSE c;
END;
/

-- Session 2: massive update
UPDATE journal_lines SET amount = amount + 1;
COMMIT;
-- undo segment doluyor

-- Session 1 fetch'i ORA-01555 atar
```

---

## Acceptance criteria

- [ ] Oracle XE çalışıyor, Flyway migrations uygulandı
- [ ] `interest_pkg`, `eod_reconciliation_pkg`, `fraud_check_pkg` implement edildi
- [ ] Java tarafı 3 package'i çağırıyor
- [ ] Idempotency: aynı businessDate iki kere accrual → exception
- [ ] Reconciliation: kasten 5 mismatch → 5/5 detect
- [ ] Fraud: 100 tx 1 dk → HIGH_FREQUENCY alert
- [ ] 1M-row query optimization: baseline → +index → +MV (defterimde tablo)
- [ ] SKIP LOCKED job queue: 10 worker × 100 job fair distribute
- [ ] Deadlock canlı reproduction + jstack/Oracle session view'lar
- [ ] ORA-01555 canlı reproduction + UNDO_RETENTION fix
- [ ] Integration test'ler %95+ geçiyor
- [ ] Defterimde 50+ defter notu

## Kasten kırma görevleri özeti

1. **Index'siz 1M-row** scan time ölç → index ekle → 50x+
2. **Deadlock** ters lock order ile zorla → lock ordering fix
3. **ORA-01555** kısa UNDO_RETENTION + long cursor → fix
4. **Worker race** SKIP LOCKED olmadan → duplicate processing reproduce
5. **MV refresh skip** uzun query açıkken → STALE behavior gör

---

## Claude-verify prompt

```
Phase 4 mini-project'imi banking-grade kriterlere göre değerlendir. Eksiklerimi 
söyle, kod yazma:

1. Oracle migration:
   - Flyway oracle dialect 5 migration tam mı?
   - RAW(16) UUID + SYS_GUID() doğru kullanım?
   - Sequence cache 50 banking standardı?
   - INTERVAL partitioning (transactions, journal) var mı?
   - CHECK constraint'ler (status, direction, amount > 0) var mı?
   - Local index'ler partitioned tablolarda?
   - Materialized view log fast refresh için?

2. PL/SQL packages:
   - `interest_pkg`: BULK COLLECT + FORALL pattern uygulanmış mı?
   - `interest_pkg`: idempotency check (business_date duplicate) var mı?
   - `interest_pkg`: compound interest formülü doğru mu?
   - `eod_reconciliation_pkg`: stored vs calculated balance comparison?
   - `eod_reconciliation_pkg`: bulk insert mismatch'ler?
   - `fraud_check_pkg`: window function ile rule logic?
   - Tüm package'larda exception handler + audit log?
   - WHEN OTHERS THEN NULL anti-pattern yok mu?

3. Java integration:
   - SimpleJdbcCall ile package çağrılıyor mu?
   - Custom exception (AlreadyProcessedException) mapping?
   - Integration test TestContainers + Oracle XE ile?

4. Performance optimization:
   - 1M-row query EXPLAIN PLAN incelenmiş mi?
   - Composite index (account_id, occurred_at) eklenmiş mi?
   - Materialized view + query rewrite çalışıyor mu?
   - Öncesi/sonrası timing tablosu defterimde mi?

5. Job queue:
   - FOR UPDATE SKIP LOCKED pattern doğru mu?
   - REQUIRES_NEW transaction worker take için?
   - 10 worker fair distribution doğrulanmış mı?

6. Concurrency reproduction:
   - Deadlock canlı görüldü mü (Oracle v$lock veya jstack)?
   - Lock ordering fix uygulandı mı?
   - ORA-01555 canlı reproduce edildi mi?
   - UNDO_RETENTION ayarı düzeltildi mi?

7. Anti-pattern:
   - Loop içinde tek-satır DML var mı? (Olmamalı, FORALL)
   - WHEN OTHERS THEN NULL var mı? (Olmamalı)
   - SELECT * FROM ... WHEN/HAVING karmaşıklığı?
   - Hardcoded magic numbers (rate 8.5 anywhere)?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Defter notları (kendi cümlelerinle)

1. "Oracle XE + Spring Boot bağlantısında karşılaştığım 3 sorun ve çözümler: ____."
2. "PostgreSQL → Oracle migration: değişen 5+ syntax/feature: ____."
3. "RAW(16) UUID vs SYS_GUID() vs sequence: hangi senaryoda hangisi: ____."
4. "INTERVAL partitioning yokken vs varken accounts/journal tablosunun yıllık büyümesi: ____."
5. "BULK COLLECT LIMIT 1000 ile loop+single DML arasındaki performans farkı (rakam): ____."
6. "Autonomous transaction `interest_pkg`'de audit için kullansaydım hangi sorunu çözerdim: ____."
7. "Compound interest formülünün doğrulanmasında SQL formülün Java implementasyonu ile farkı: ____."
8. "EOD reconciliation: authoritative source kararı (stored vs calculated) + sebep: ____."
9. "Fraud rule false positive vs missed fraud trade-off banking için: ____."
10. "1M-row sorgusunda baseline → index → MV speedup factor (rakam): ____."
11. "Materialized view fast refresh için log neden gerekli: ____."
12. "SKIP LOCKED job queue ile distributed lock (DBMS_LOCK) farkı: ____."
13. "Deadlock reproduction sırasında Oracle hangi session'ı öldürür ve kriterler: ____."
14. "ORA-01555 önleme: UNDO_RETENTION ne kadar olmalı banking için: ____."
15. "Çalıştırdığım EXPLAIN PLAN'da en şaşırtıcı bulgu: ____."
