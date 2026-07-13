# Topic 4.6 — DB Concurrency & Locking

## Hedef

Oracle ve PostgreSQL'in concurrency mekaniklerini banking-grade seviyede öğrenmek. SELECT FOR UPDATE varyantları (NOWAIT, WAIT n, SKIP LOCKED), enqueue lock'lar (Oracle), MVCC ile locking ilişkisi, deadlock detection ve banking için queue worker pattern.

## Süre

Okuma: 1.5 saat • Mini task: 2 saat • Test: 45 dk • Toplam: ~4.5 saat

## Önbilgi

- Topic 4.1-4.5 bitti
- Phase 2'deki locking topic'i (JPA tarafı) bitti
- Concurrent transaction kavramına aşinasın

---

## Kavramlar

### 1. Isolation level'lar — DB perspektifinden

Phase 2'de JPA tarafından gördük. Şimdi **DB tarafından** bak.

#### Oracle default: READ COMMITTED

- Reader'lar commit edilmiş veriyi görür
- **MVCC ile read-without-blocking** — writer reader'ı bloklamaz
- Dirty read yok (uncommitted reader olmaz)
- Non-repeatable read **olabilir** (aynı sorgu iki kez = farklı sonuç)

#### Oracle SERIALIZABLE

```sql
ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;
```

Transaction izolasyonu **tam**. Phantom read yok. Ama:

```
ORA-08177: can't serialize access for this transaction
```

Çakışma durumunda Oracle bu hatayı atar — **retry** etmek senin işin.

#### PostgreSQL — 3 isolation level

- READ COMMITTED (default)
- REPEATABLE READ
- SERIALIZABLE — true serializable (SSI algorithm)

**PostgreSQL READ COMMITTED:** Her statement kendi snapshot'ını alır.

**PostgreSQL REPEATABLE READ:** Transaction başında snapshot. İlerde değişmez.

**PostgreSQL SERIALIZABLE:** SSI (Serializable Snapshot Isolation) — concurrent transaction'lar arası dependency tespit, çakışma olunca biri reject:

```
ERROR: could not serialize access due to read/write dependencies among transactions
```

**Banking pratiği:** Çoğunluk READ COMMITTED. Critical scenarios (eşzamanlı bakiye check + withdraw) için SERIALIZABLE veya explicit lock.

### 2. MVCC vs locking — temel fark

**Old school DB:** Reader writer'ı bloklar. Banking için yetersiz.

**MVCC (Oracle, PostgreSQL):**
- Reader writer'ı bloklamaz
- Writer reader'ı bloklamaz
- Writer writer'ı bloklar (aynı satırda)

Her satırın **birden fazla versiyonu** yaşar (kısa süreliğine). Reader'lar uygun versiyonu görür.

```
T1: BEGIN; SELECT balance FROM accounts WHERE id = 1;   -- 100 görür
T2: BEGIN; UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;          -- READ COMMITTED: 200 (snapshot her stmt)
                                                         -- REPEATABLE READ: 100 (snapshot başta)
```

### 3. SELECT FOR UPDATE — explicit row lock

```sql
BEGIN;
SELECT * FROM accounts WHERE id = '...' FOR UPDATE;
-- Bu satır lock — başka transaction UPDATE/DELETE/SELECT FOR UPDATE bekler

UPDATE accounts SET balance_amount = balance_amount - 100 WHERE id = '...';
COMMIT;
```

**Banking örneği — concurrent transfer:**

```sql
BEGIN;
SELECT * FROM accounts WHERE id = :from_id FOR UPDATE;   -- lock from
SELECT * FROM accounts WHERE id = :to_id FOR UPDATE;     -- lock to
UPDATE accounts SET balance_amount = balance_amount - :amount WHERE id = :from_id;
UPDATE accounts SET balance_amount = balance_amount + :amount WHERE id = :to_id;
COMMIT;
```

**Tehlike:** İki concurrent transfer A→B ve B→A → **deadlock**. Lock ordering ile çöz (her zaman küçük ID'yi önce lock'la).

### 4. SELECT FOR UPDATE varyantları

#### NOWAIT

```sql
SELECT * FROM accounts WHERE id = '...' FOR UPDATE NOWAIT;
```

Eğer lock available değilse **anında hata** (ORA-00054 / SQLSTATE 55P03). Beklemez.

**Kullanım:** Optimistic check. Lock'lanmışsa "şu an dolu, sonra dene" cevabı.

#### WAIT n

```sql
SELECT * FROM accounts WHERE id = '...' FOR UPDATE WAIT 5;
```

5 saniye bekle, alınmazsa hata. **Banking timeout pattern.**

#### SKIP LOCKED

```sql
SELECT * FROM job_queue WHERE status = 'PENDING' 
ORDER BY created_at 
FOR UPDATE SKIP LOCKED 
FETCH FIRST 1 ROWS ONLY;
```

Lock'lanmış satırları **atla**, sonraki available olanı al. **Worker pool için altın pattern.**

**Banking örneği — Job queue worker:**

```sql
-- Worker 1, Worker 2, ... aynı queue'dan iş çekiyor
BEGIN;

SELECT id FROM transfer_jobs 
WHERE status = 'PENDING' 
ORDER BY priority DESC, created_at 
FOR UPDATE SKIP LOCKED 
FETCH FIRST 1 ROWS ONLY;

-- Worker bu job'u alır, başkaları skip edip diğerini alır
UPDATE transfer_jobs SET status = 'PROCESSING' WHERE id = :selected_id;

COMMIT;

-- İş yap...

BEGIN;
UPDATE transfer_jobs SET status = 'COMPLETED' WHERE id = :selected_id;
COMMIT;
```

10 worker, 10 ayrı job alır — **kavga etmez**. Lock contention yok.

### 5. Lock modes

#### Oracle lock types

- **TM (table-level):** DDL veya foreign key
- **TX (transaction-level):** Row lock — SELECT FOR UPDATE, UPDATE, DELETE
- **UL (user-level):** DBMS_LOCK ile manuel

#### Lock granularity

- Row-level (Oracle default): Sadece etkilenen satır
- Page-level: Yok Oracle'da
- Table-level: LOCK TABLE veya DDL

```sql
LOCK TABLE accounts IN EXCLUSIVE MODE;
-- Tüm accounts tablosu lock — DİKKAT, banking'de ciddi yavaşlatır
```

### 6. DBMS_LOCK — application-level lock

Application-defined named lock'lar.

```sql
DECLARE
    v_lock_handle VARCHAR2(128);
    v_result NUMBER;
BEGIN
    DBMS_LOCK.ALLOCATE_UNIQUE('EOD_RECONCILIATION', v_lock_handle);
    v_result := DBMS_LOCK.REQUEST(v_lock_handle, DBMS_LOCK.X_MODE, 0, FALSE);
    -- v_result = 0 → lock alındı
    -- v_result = 1 → timeout
    
    IF v_result = 0 THEN
        -- Critical section: EOD reconciliation
        eod_reconciliation_pkg.reconcile_balances(SYSDATE);
        
        DBMS_LOCK.RELEASE(v_lock_handle);
    ELSE
        RAISE_APPLICATION_ERROR(-20100, 'EOD already running');
    END IF;
END;
/
```

**Banking örneği:** EOD job'unun **iki kez aynı anda** çalışmasını önle. ShedLock pattern'in DB karşılığı.

### 7. Deadlock detection — Oracle

Oracle **otomatik deadlock detection** yapar (4 sn poll). Tespit edince **bir transaction'ı rollback** eder, hata:

```
ORA-00060: deadlock detected while waiting for resource
```

**`v$lock` ve `dba_blockers/dba_waiters`** view'larıyla aktif lock'ları sorgula:

```sql
SELECT s1.username AS waiting_user, s2.username AS blocking_user, l1.lock_type
FROM v$lock l1, v$lock l2, v$session s1, v$session s2
WHERE l1.id1 = l2.id1
  AND l1.id2 = l2.id2
  AND l1.request > 0
  AND l2.request = 0
  AND l1.sid = s1.sid
  AND l2.sid = s2.sid;
```

Production'da DBA bu sorguyu kullanır. Sen biliyor ol.

### 8. PostgreSQL'de locking

PostgreSQL benzer ama bazı farklar:

```sql
SELECT * FROM accounts WHERE id = '...' FOR UPDATE;       -- standard
SELECT * FROM accounts WHERE id = '...' FOR UPDATE NOWAIT;  -- NOWAIT
SELECT * FROM accounts WHERE id = '...' FOR UPDATE SKIP LOCKED;  -- 9.5+
```

**Advisory locks:** Oracle DBMS_LOCK karşılığı:

```sql
SELECT pg_advisory_lock(12345);   -- lock numarası
-- ...
SELECT pg_advisory_unlock(12345);
```

**Deadlock:** PostgreSQL aynen Oracle gibi otomatik detect eder, biri rollback olur.

### 9. Banking pattern'ler

#### Pattern 1: Job queue with SKIP LOCKED

Worker'lar paralel çalışsın, lock kavgası olmasın.

```sql
-- Worker
BEGIN;
SELECT id, payload FROM transfer_jobs
WHERE status = 'PENDING' AND scheduled_at <= SYSTIMESTAMP
ORDER BY priority DESC, created_at
FOR UPDATE SKIP LOCKED
FETCH FIRST 1 ROWS ONLY;

UPDATE transfer_jobs SET status = 'PROCESSING', worker_id = :worker_id WHERE id = :id;
COMMIT;
```

#### Pattern 2: Distributed singleton (DBMS_LOCK)

EOD job'unun tek instance çalışması:

```sql
DECLARE
    v_handle VARCHAR2(128);
    v_result NUMBER;
BEGIN
    DBMS_LOCK.ALLOCATE_UNIQUE('EOD_DAILY_JOB', v_handle);
    v_result := DBMS_LOCK.REQUEST(v_handle, DBMS_LOCK.X_MODE, 0, FALSE);
    
    IF v_result != 0 THEN
        DBMS_OUTPUT.PUT_LINE('EOD already running');
        RETURN;
    END IF;
    
    BEGIN
        eod_pkg.run_full_cycle();
        DBMS_LOCK.RELEASE(v_handle);
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_LOCK.RELEASE(v_handle);
            RAISE;
    END;
END;
/
```

PostgreSQL `pg_advisory_lock` karşılığı yapar.

#### Pattern 3: Optimistic check (NOWAIT)

```sql
BEGIN;
SELECT * FROM accounts WHERE id = '...' FOR UPDATE NOWAIT;
-- Hata: ORA-00054 → "şu an meşgul, sonra dene"
EXCEPTION
    WHEN OTHERS THEN
        IF SQLCODE = -54 THEN
            RAISE_APPLICATION_ERROR(-20200, 'Account busy, try again');
        END IF;
        RAISE;
END;
```

#### Pattern 4: Transfer with lock ordering

```sql
PROCEDURE transfer(p_from RAW, p_to RAW, p_amount NUMBER) IS
    v_first RAW(16);
    v_second RAW(16);
BEGIN
    -- Lock ordering — küçük ID önce, deadlock önleme
    IF p_from < p_to THEN
        v_first := p_from;
        v_second := p_to;
    ELSE
        v_first := p_to;
        v_second := p_from;
    END IF;
    
    SELECT * FROM accounts WHERE id = v_first FOR UPDATE;
    SELECT * FROM accounts WHERE id = v_second FOR UPDATE;
    
    UPDATE accounts SET balance_amount = balance_amount - p_amount WHERE id = p_from;
    UPDATE accounts SET balance_amount = balance_amount + p_amount WHERE id = p_to;
END;
/
```

### 10. Lock contention monitoring

**Oracle v$session_wait:**

```sql
SELECT event, sid, wait_time, seconds_in_wait
FROM v$session_wait
WHERE event LIKE 'enq:%'
ORDER BY seconds_in_wait DESC;
```

`enq: TX - row lock contention` → row lock için bekleyenler.

**PostgreSQL pg_locks:**

```sql
SELECT relation::regclass, mode, pid, granted
FROM pg_locks
WHERE NOT granted;
```

Granted = false satırlar → bekleyenler.

### 11. SSI (Serializable Snapshot Isolation) — PostgreSQL

PostgreSQL SERIALIZABLE özel — **snapshot izolasyon + dependency tracking**. Çakışma olmadan optimistic; çakışma varsa **runtime'da reject**.

```
T1: BEGIN; SELECT SUM(balance) FROM accounts;     -- 1000
T2: BEGIN; SELECT SUM(balance) FROM accounts;     -- 1000
T1: UPDATE accounts SET balance = balance + 100;
T2: UPDATE accounts SET balance = balance - 100;
T1: COMMIT;
T2: COMMIT;   -- ERROR: could not serialize
```

**Banking örneği:** "Tüm hesapların toplamı X mi" garantisi — SSI ile guarantees.

Maliyet: Tracking overhead, retry burden.

### 12. Read replicas — write-read separation

Heavy reporting query'leri **read-only replica**'ya gönder. Primary'i serbestleş.

```yaml
spring:
  datasource:
    primary:
      url: jdbc:oracle:thin:@primary-host:1521:ORCL
    replica:
      url: jdbc:oracle:thin:@replica-host:1521:ORCL
```

```java
@Service
@Transactional(readOnly = true)   // routing decision
class ReportingService {
    @ReadOnly
    public List<DailySummary> dailyReport(LocalDate date) { ... }
}
```

Reader transaction primary'i etkilemez. Replica lag (~saniyeler) — analytical query'lere kabul edilebilir.

### 13. Anti-pattern'ler

**Anti-pattern 1: Table-level lock**

```sql
LOCK TABLE accounts IN EXCLUSIVE MODE;
```

Tüm tablo lock → tüm operasyon serial. Banking için **felaket**.

**Anti-pattern 2: Long transaction**

```sql
BEGIN;
SELECT * FROM accounts FOR UPDATE;
-- ... 5 dakika iş ...
-- başka thread'ler bekliyor
COMMIT;
```

Transaction kısa olsun. Lock tutulduğu süre = başka thread'in bekleme süresi.

**Anti-pattern 3: NOWAIT yerine sonsuz bekleme**

Web request flow'da SELECT FOR UPDATE: kullanıcı 30 saniye bekleyemez. **NOWAIT veya WAIT n** ile timeout koy.

**Anti-pattern 4: Lock ordering yok**

İki account paralel UPDATE → deadlock. **Her zaman aynı sıra.**

**Anti-pattern 5: Application-level retry yok**

ORA-00060 (deadlock), ORA-08177 (serialization), 55P03 (NOWAIT) — **retry mantığı şart.**

```java
@Retryable(value = {DeadlockLoserDataAccessException.class}, maxAttempts = 3,
           backoff = @Backoff(delay = 100, multiplier = 2))
public Transfer execute(...) { ... }
```

---

## Önemli olabilecek araştırma kaynakları

- Oracle Concurrency Control documentation
- PostgreSQL Concurrency Control documentation (SSI section)
- "Designing Data-Intensive Applications" (Kleppmann) — Chapter 7 (Transactions)
- Tom Kyte — Oracle locking deep dives
- PostgreSQL wiki — SSI explanation
- "Use The Index, Luke" — concurrent updates section

---

## Mini task'ler

### Task 4.6.1 — SELECT FOR UPDATE basic (15 dk)

İki SQL session aç. Session 1:
```sql
BEGIN;
SELECT * FROM accounts WHERE id = '...' FOR UPDATE;
```

Session 2:
```sql
UPDATE accounts SET balance_amount = 0 WHERE id = '...';
-- BLOCKS — bekler
```

Session 1 commit edince Session 2 ilerler. **Defterine** zaman akışı yaz.

### Task 4.6.2 — NOWAIT (15 dk)

Session 2'de NOWAIT:
```sql
SELECT * FROM accounts WHERE id = '...' FOR UPDATE NOWAIT;
-- ORA-00054 anında
```

### Task 4.6.3 — SKIP LOCKED job queue (45 dk)

`transfer_jobs` tablo yarat:

```sql
CREATE TABLE transfer_jobs(
    id RAW(16) DEFAULT SYS_GUID(),
    payload CLOB,
    status VARCHAR2(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP
);
```

10 job insert et. 3 worker (3 ayrı session veya Java thread) — her biri:

```sql
BEGIN;
SELECT id FROM transfer_jobs WHERE status = 'PENDING'
FOR UPDATE SKIP LOCKED
FETCH FIRST 1 ROWS ONLY;

UPDATE transfer_jobs SET status = 'PROCESSING' WHERE id = :id;
COMMIT;
```

Worker'lar farklı job'ları alıyor mu? **Defterine** akış yaz.

### Task 4.6.4 — Deadlock reproduction + lock ordering (45 dk)

İki session:

```
Session 1: SELECT * FROM accounts WHERE id = 'A' FOR UPDATE;
Session 2: SELECT * FROM accounts WHERE id = 'B' FOR UPDATE;
Session 1: SELECT * FROM accounts WHERE id = 'B' FOR UPDATE;   -- bekler
Session 2: SELECT * FROM accounts WHERE id = 'A' FOR UPDATE;   -- bekler
```

ORA-00060 deadlock. Bir session rollback olur.

Lock ordering ile çöz: her ikisi de **küçük ID'i önce** kilitle. Deadlock kayboldu mu?

### Task 4.6.5 — DBMS_LOCK ile distributed singleton (30 dk)

EOD job'unu DBMS_LOCK ile koru. İki paralel çağrı dene, ikincisi hata almalı.

### Task 4.6.6 — pg_advisory_lock PostgreSQL (15 dk)

Eğer PostgreSQL'in de varsa:

```sql
SELECT pg_advisory_lock(12345);
-- başka session'da:
SELECT pg_advisory_lock(12345);   -- bekler

-- ilk session:
SELECT pg_advisory_unlock(12345);
```

---

## Test yazma rehberi

### Test 4.6.1 — Concurrent SELECT FOR UPDATE

```java
@Test
void concurrentSelectForUpdateShouldSerialize() throws InterruptedException {
    UUID accId = createAccount("1000");
    
    AtomicLong session2WaitMs = new AtomicLong();
    CountDownLatch session1Holding = new CountDownLatch(1);
    CountDownLatch session1Released = new CountDownLatch(1);
    
    Thread session1 = new Thread(() -> {
        transactionTemplate.execute(status -> {
            jdbc.queryForObject("SELECT id FROM accounts WHERE id = ? FOR UPDATE",
                String.class, accId.toString());
            session1Holding.countDown();
            try {
                session1Released.await();
            } catch (InterruptedException e) {}
            return null;
        });
    });
    
    Thread session2 = new Thread(() -> {
        try {
            session1Holding.await();
            long start = System.currentTimeMillis();
            transactionTemplate.execute(status -> {
                jdbc.queryForObject("SELECT id FROM accounts WHERE id = ? FOR UPDATE",
                    String.class, accId.toString());
                return null;
            });
            session2WaitMs.set(System.currentTimeMillis() - start);
        } catch (InterruptedException e) {}
    });
    
    session1.start();
    session2.start();
    Thread.sleep(500);   // session 2 beklesin
    session1Released.countDown();   // session 1 commit
    session1.join();
    session2.join();
    
    assertThat(session2WaitMs.get()).isGreaterThan(400);   // session 1'in commit'ine bekledi
}
```

### Test 4.6.2 — SKIP LOCKED parallel workers

```java
@Test
void skipLockedWorkersShouldDistributeJobs() throws InterruptedException {
    insertJobs(100);
    
    ExecutorService exec = Executors.newFixedThreadPool(10);
    AtomicInteger processed = new AtomicInteger();
    CountDownLatch done = new CountDownLatch(100);
    
    for (int i = 0; i < 10; i++) {
        exec.submit(() -> {
            while (true) {
                Optional<UUID> jobId = takeNextJob();
                if (jobId.isEmpty()) break;
                processJob(jobId.get());
                processed.incrementAndGet();
                done.countDown();
            }
        });
    }
    
    done.await(30, TimeUnit.SECONDS);
    
    assertThat(processed).hasValue(100);
    // 10 worker ~10 job/worker dağılmış olmalı
}

private Optional<UUID> takeNextJob() {
    return jdbc.query("""
        SELECT id FROM transfer_jobs WHERE status = 'PENDING'
        FOR UPDATE SKIP LOCKED FETCH FIRST 1 ROWS ONLY
        """, rs -> rs.next() ? Optional.of(UUID.fromString(rs.getString(1))) : Optional.empty());
}
```

---

## Claude-verify prompt

```
DB concurrency ve locking kodumu banking-grade kriterlere göre değerlendir:

1. Isolation level:
   - Default READ COMMITTED tercih edilmiş mi?
   - SERIALIZABLE'a geçilen yerlerde retry logic var mı?
   - PostgreSQL SSI semantics anlaşılmış mı?

2. SELECT FOR UPDATE:
   - Web request flow'da NOWAIT veya WAIT n ile timeout koyulmuş mu?
   - SKIP LOCKED job queue pattern'i kullanılmış mı?
   - Lock ordering ile deadlock prevention?

3. Job queue:
   - SKIP LOCKED ile paralel worker desteği var mı?
   - Worker crash sonrası job recovery (timeout-based) var mı?

4. Distributed singleton:
   - DBMS_LOCK (Oracle) veya pg_advisory_lock (PostgreSQL) ile job mutex var mı?
   - EOD/scheduled job'lar tek instance garantisi alıyor mu?

5. Lock contention:
   - v$session_wait veya pg_locks ile monitoring var mı?
   - Long transaction varsayılan değil, kısa transaction patterned mi?

6. Deadlock handling:
   - Application-level retry logic (ORA-00060, ORA-08177) var mı?
   - Lock ordering (deterministic order by ID)?
   - Spring @Retryable veya manual loop?

7. Anti-pattern:
   - Table-level lock (LOCK TABLE) var mı? (Olmamalı)
   - Long transaction (5+ saniye) var mı?
   - Application retry yok mu?

8. Read replica:
   - Heavy reporting query'leri replica'ya yönlendirilmiş mi?
   - @Transactional(readOnly = true) routing açık mı?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] SELECT FOR UPDATE'in NOWAIT, WAIT n, SKIP LOCKED varyantlarını biliyorum
- [ ] SKIP LOCKED ile paralel worker job queue pattern'ini uyguladım
- [ ] Deadlock'u kasten reproduction yaptım (iki session, ters lock sırası)
- [ ] Lock ordering ile deadlock'u önledim
- [ ] DBMS_LOCK veya pg_advisory_lock ile distributed singleton pattern'i denedim
- [ ] v$session_wait veya pg_locks ile blocking session'ları sorguladım
- [ ] Application-level retry (Spring @Retryable) deadlock için var
- [ ] SERIALIZABLE isolation'ın çakışma durumunda retry gerektirdiğini biliyorum
- [ ] Read replica routing kavramına aşinayım
- [ ] Long transaction'ın banking için tehlikesini biliyorum

---

## Defter notları

1. "MVCC ile traditional locking farkı (reader blocked vs not): ____."
2. "SELECT FOR UPDATE varyantları (NOWAIT/WAIT n/SKIP LOCKED) ne zaman hangisi: ____."
3. "SKIP LOCKED job queue pattern'inin banking için neden ideal: ____."
4. "Deadlock'u lock ordering ile nasıl önlerim: ____."
5. "DBMS_LOCK / pg_advisory_lock distributed singleton: ____."
6. "Oracle SERIALIZABLE vs PostgreSQL SSI: ____."
7. "Long transaction'ın banking için yarattığı sorunlar: ____."
8. "v$session_wait / pg_locks ile blocking analysis: ____."
9. "Application retry stratejisi (deadlock + serialization fail): ____."
10. "Read replica routing strategy banking için ne zaman değer: ____."
