# Topic 2.3 — Transaction Management

## Hedef

`@Transactional` annotation'ının **iç mekanizmasını** (Spring proxy, AOP, advisor) öğrenmek. 7 propagation davranışını ve 4 isolation level'ı banking senaryoları üzerinden ezberlemeden — sebep–sonuç olarak kavramak. Self-invocation problemini, rollback default davranışını (checked vs unchecked), readOnly, timeout, rollbackFor / noRollbackFor parametrelerini hatasız anlatabilmek. Banking'in en sık mülakat sorusu olan "para transferi transaction tasarımı"nı (debit + credit + audit log, mixed propagation ile) hatasız çizebilmek.

## Süre

Okuma: 2-2.5 saat • Mini task: 3 saat • Test: 1.5 saat • Toplam: ~7 saat

## Önbilgi

- Topic 2.1 (JPA Fundamentals) bitti — persistence context, entity state, flush biliyorsun
- Topic 2.2 (Spring Data JPA) bitti — `JpaRepository`, `@Query` rahat kullanabiliyorsun
- `@Transactional` annotation'ını gördün ama nasıl çalıştığını "auto-rollback" seviyesinde biliyorsun
- ACID kısaltmasını duydun, "Atomicity, Consistency, Isolation, Durability" diye ezberden söyleyebiliyorsun

---

## Kavramlar

### 1. ACID — banking için ne demek

Klasik tanımlar ezberlenir, banking pratiğine düşmez. Her birini somut bir senaryo ile zihne çakalım:

**Atomicity (atomik olma):** Bir işlem ya **tamamen** olur, ya **hiç** olmaz. Yarım kalmaz.

Banking: Müşteri 100 TL'yi A hesabından B'ye gönderiyor. SQL açısından bu işlem **iki UPDATE'tir**:

```sql
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
```

İlk UPDATE çalıştı, ikincide DB bağlantısı koptu. Atomicity yoksa: A'dan 100 düştü, B'ye geçmedi → **100 TL buharlaştı**. Bu bir bankada **dakikalar içinde regülatör cezası**.

Atomicity transaction ile sağlanır: ya iki UPDATE de commit, ya ikisi de rollback.

**Consistency (tutarlılık):** Transaction veritabanını **bir geçerli state'ten başka bir geçerli state'e** götürür. "Geçerli" = tüm constraint'ler, foreign key'ler, CHECK kuralları, business invariant'lar yerinde.

Banking: "Toplam debit = toplam credit" double-entry invariant'ı. Transaction commit ettiğinde bu eşitlik bozulamaz.

Consistency'i DB tek başına sağlamaz — uygulama da yardım eder. CHECK constraint (DB) + domain logic (uygulama) + transaction (atomicity).

**Isolation (yalıtım):** Eşzamanlı transaction'lar birbirini **karıştırmaz**. Bir transaction'ın yarım kalmış değişiklikleri başkasına görünmez (en azından dirty read yok).

Banking: A'nın bakiyesi 1000. Aynı anda iki çekme talebi: T1 "800 çek", T2 "500 çek". Isolation yoksa: ikisi de "1000 var" der, ikisi de geçer, bakiye -300'e düşer. Isolation: biri görür "200 kaldı", reddeder.

**Durability (kalıcılık):** Commit edildi → diske yazıldı → güç gitse bile **kayıp yok**. WAL (write-ahead log) ile sağlanır.

Banking: Sistem 2 saat aşağıdaydı, ama commit ettiğin transfer hâlâ orada. Aksi halde "transfer attım ama yok!" şikayetleri.

**Banking pratiği:** Atomicity ve Isolation günlük tasarımda en çok düşündüğün iki harf. Durability genelde DB'nin işi. Consistency hem DB hem uygulama. Mülakatta "ACID'i anlat" sorusunun ardından gelen "bir banking senaryosu ile" cümlesini bekle.

---

### 2. JDBC transaction — Spring'in altında ne var

`@Transactional` sihir gibi durur. Altında **JDBC `Connection`** var. Klasik bir transaction:

```java
Connection conn = dataSource.getConnection();
try {
    conn.setAutoCommit(false);                    // transaction başlat
    try (var ps = conn.prepareStatement("UPDATE accounts SET balance = balance - ? WHERE id = ?")) {
        ps.setBigDecimal(1, amount);
        ps.setString(2, fromId);
        ps.executeUpdate();
    }
    try (var ps = conn.prepareStatement("UPDATE accounts SET balance = balance + ? WHERE id = ?")) {
        ps.setBigDecimal(1, amount);
        ps.setString(2, toId);
        ps.executeUpdate();
    }
    conn.commit();                                // başarılı → commit
} catch (Exception e) {
    conn.rollback();                              // hata → rollback
    throw e;
} finally {
    conn.close();                                 // bağlantıyı pool'a iade et
}
```

Bu **çalışır** ama her servis method'una yazmak istemiyoruz. Spring `@Transactional` bunun **AOP wrapper**'ıdır.

---

### 3. Spring `@Transactional` proxy mekanizması

Spring `@Transactional` annotation'ı işaretli class veya method için bir **proxy** üretir. Proxy ilgili method'u sarar:

```
[caller] → [proxy] → (begin tx) → [gerçek bean.method()] → (commit/rollback) → [caller]
```

İki proxy türü:

**JDK dynamic proxy:** Class bir interface implement ediyorsa, Spring **interface bazlı proxy** üretir. `java.lang.reflect.Proxy` kullanır. Proxy nesnesi interface'i implement eder; her method çağrısı bir `InvocationHandler`'a düşer.

```java
public interface TransferService { Transfer execute(...); }

@Service
class TransferServiceImpl implements TransferService {
    @Transactional
    public Transfer execute(...) { ... }
}

// Spring proxy üretir: $Proxy42 implements TransferService
// Bean container'da TransferService olarak kayıtlı
```

**CGLIB proxy:** Class interface implement etmiyorsa, Spring **subclass proxy** üretir. CGLIB (Code Generation Library) ile gerçek class'tan extend eden bir subclass yaratır, method'ları override eder.

```java
@Service
class TransferService {                          // interface yok
    @Transactional
    public Transfer execute(...) { ... }
}

// Spring CGLIB ile: TransferService$$EnhancerBySpringCGLIB$$xxx extends TransferService
```

Spring Boot 2.6+ **default CGLIB** — interface olsa bile. Sebep: AOP davranışı her zaman tutarlı olsun. `spring.aop.proxy-target-class=false` ile JDK proxy'e döndürebilirsin.

**Önemli sonuç:** Proxy aslında **dışarıdaki çağrıları** dinler. **İçeriden** (aynı bean'in başka method'u) çağrılan method proxy'i bypass eder.

---

### 4. Self-invocation problemi (mülakat klasiği)

Şu kod ÇALIŞMAZ:

```java
@Service
class AccountService {
    
    public void doTransferAndAudit(...) {
        executeTransfer(...);       // ❌ aynı class içinden çağrı
        writeAuditLog(...);
    }
    
    @Transactional
    public void executeTransfer(...) {
        // ...
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeAuditLog(...) {
        // ...
    }
}
```

`doTransferAndAudit` çağrıldığında **bir TX başlamaz**. `executeTransfer` ve `writeAuditLog` `this.` üzerinden çağrıldığı için proxy devreye girmez.

**Sebep:** `doTransferAndAudit` içindeki `executeTransfer()` çağrısı **proxy nesnesine değil, gerçek `this`'e** gider. Proxy ortadan kayboldu → `@Transactional` ignored.

**Mülakat sorusu:** "Aynı service'in iki method'u var, biri diğerini çağırıyor, ikisi de `@Transactional` — niye ikinci method yeni TX açmıyor?" → Self-invocation.

**Çözümler:**

**A — Sınıfları ayır (en temiz):**

```java
@Service
class TransferOrchestrator {
    private final TransferService transferService;
    private final AuditService auditService;
    
    public void doTransferAndAudit(...) {
        transferService.executeTransfer(...);   // başka bean, proxy üzerinden ✓
        auditService.writeAuditLog(...);
    }
}
```

**B — Self-injection:**

```java
@Service
class AccountService {
    
    @Autowired
    @Lazy
    private AccountService self;       // proxy'i kendine inject et
    
    public void doTransferAndAudit(...) {
        self.executeTransfer(...);     // proxy üzerinden ✓
        self.writeAuditLog(...);
    }
    
    @Transactional
    public void executeTransfer(...) { ... }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeAuditLog(...) { ... }
}
```

`@Lazy` circular dependency'i bypass için. Hack gibi durur ama meşrudur.

**C — `AopContext.currentProxy()`:**

```java
@EnableAspectJAutoProxy(exposeProxy = true)
public class Config {}

// içinde:
((AccountService) AopContext.currentProxy()).executeTransfer(...);
```

Çalışır ama okunmaz, statik bağımlılık doğurur. **Tercih etme**.

**D — AspectJ load-time weaving:** Proxy'leri tamamen unut, AspectJ ile bytecode-level weaving. Çok güçlü ama kurulum yorucu. Banking'de yaygın değil.

**Banking pratiği:** A varyantı her zaman tercih. Transfer orchestrator + transfer service + audit service ayrımı zaten DDD ile uyumlu.

---

### 5. `@Transactional` annotation'ının tüm parametreleri

```java
@Transactional(
    propagation = Propagation.REQUIRED,            // default
    isolation = Isolation.DEFAULT,                  // DB default
    readOnly = false,                               // default
    timeout = -1,                                   // saniye, -1 = sınırsız
    rollbackFor = {},                               // ek class'lar
    noRollbackFor = {},                             // hariç tutulanlar
    transactionManager = "transactionManager"       // birden fazla DB varsa
)
public void method() { ... }
```

Şimdi her birini tek tek deşeceğiz.

---

### 6. Propagation — 7 davranış

`Propagation` = bir method çağrıldığında **mevcut transaction ile ne yapsın**.

#### 6.1. `REQUIRED` (default)

> "Mevcut TX varsa katıl, yoksa yeni başlat."

%90 senaryoda bunu kullanırsın. Service method'u TX açar, başka bir service'i çağırınca o da aynı TX'e katılır, hepsi atomik olur.

Banking örneği — transfer:

```java
@Service
class TransferService {
    
    @Transactional   // REQUIRED default — TX başlatır
    public void transfer(AccountId from, AccountId to, Money amount) {
        accountService.debit(from, amount);    // aynı TX'e katılır
        accountService.credit(to, amount);     // aynı TX'e katılır
        // hepsi commit → ya tümü, ya hiçbiri
    }
}

@Service
class AccountService {
    @Transactional   // REQUIRED — caller TX varsa katıl
    public void debit(AccountId id, Money amount) { ... }
    
    @Transactional
    public void credit(AccountId id, Money amount) { ... }
}
```

**Logical vs Physical TX:** REQUIRED'da iç method "logical transaction"a katılır ama physical olarak **tek TX vardır**. Rollback davranışı: iç method'da exception fırlarsa **dış TX'i de rollback only işaretler**. Dış katman commit denerse `UnexpectedRollbackException` patlar.

#### 6.2. `REQUIRES_NEW`

> "Mevcut TX'i askıya al, yeni bir bağımsız TX başlat, bitince eskisini devam ettir."

**İki ayrı physical TX**. İçerideki TX commit veya rollback olabilir — dışarıdaki etkilenmez.

Banking örneği — audit log:

```java
@Service
class TransferService {
    
    @Transactional   // ana TX
    public void transfer(...) {
        debit(from, amount);
        credit(to, amount);
        
        // BURADA bir hata olursa transfer rollback olmalı
        // AMA audit log YAZILMALI (regülatör için)
        auditService.logAttempt(...);     // REQUIRES_NEW
        
        // ...
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAttempt(...) {
        auditRepo.save(...);     // bağımsız TX, ana TX rollback olsa da yazılı kalır
    }
}
```

**Mülakat sorusu (klasik):** "Transfer başarısız olduğunda audit log da rollback oluyor — neden, nasıl düzeltirsin?" Cevap: `REQUIRES_NEW`.

**Tuzaklar:**

- REQUIRES_NEW her seferinde **yeni bir DB bağlantısı** alır. İç içe çok sayıda REQUIRES_NEW = connection pool tükenir, deadlock.
- Dış TX hâlâ bekliyor, içeride yeni TX başlattın → connection pool'dan 2 connection tutuyorsun. Pool size 2 ise hemen blok.
- İç TX commit olur ama dış TX rollback. **Audit log yazıldı, transfer atılmadı** durumu — bilinçli olmalı.

**Banking pratiği:**

- Audit log: REQUIRES_NEW (her zaman yazılmalı, dış TX'i bilmiyor)
- Failed transaction attempt log: REQUIRES_NEW (rollback olacak ama log kalsın)
- Notification (SMS, email kuyruğa atma): REQUIRES_NEW veya event-based async (Phase 4)

#### 6.3. `NESTED`

> "Aynı TX içinde bir **savepoint** oluştur. Hata olursa savepoint'e kadar rollback, dış TX devam edebilir."

JDBC `Connection.setSavepoint()` kullanır. **DB'nin savepoint desteği gerekir** (PostgreSQL, MySQL InnoDB, Oracle var; bazı eski DB'lerde yok).

```java
@Service
class BatchPaymentService {
    
    @Transactional   // REQUIRED — ana TX
    public BatchResult processBatch(List<Payment> payments) {
        var results = new ArrayList<PaymentResult>();
        for (Payment p : payments) {
            try {
                results.add(processSingle(p));    // NESTED — savepoint
            } catch (PaymentFailedException e) {
                // savepoint'e geri dönüldü, ana TX devam ediyor
                results.add(PaymentResult.failed(p, e.getMessage()));
            }
        }
        return new BatchResult(results);
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public PaymentResult processSingle(Payment p) {
        // ... rollback olursa sadece bu kayıt
    }
}
```

Banking senaryosu: 1000 transferlik batch işliyorsun. Birinde hesap kapalı → o tek transfer rollback, kalan 999'u devam et.

**REQUIRES_NEW ile farkı:**
- REQUIRES_NEW: iki ayrı physical TX, iki ayrı connection
- NESTED: tek physical TX, savepoint
- REQUIRES_NEW dış commit'ten bağımsız iç commit olur
- NESTED iç commit hâlâ dış commit'e bağlı — dış rollback → iç de rollback

Hibernate'le NESTED **tüm DB'lerde sorunsuz** çalışmaz. Hibernate 6 ile destek daha iyi ama yine de test et.

#### 6.4. `MANDATORY`

> "Mevcut TX varsa katıl, **yoksa exception fırlat**."

Banking örneği: "Bu method sadece bir TX içinde çalışabilir, doğrudan çağırırsan hata":

```java
@Transactional(propagation = Propagation.MANDATORY)
public void postJournalLine(JournalLine line) {
    // double-entry'nin yarısı — atomicity başka birinin sorumluluğu
    // bunu TX'siz çağırmak isteyene "açık TX olmadan kullanma" der
}
```

`IllegalTransactionStateException` fırlatır.

**Pratik:** API'yı sağlamlaştırmak için. Helper service'ler her zaman bir caller TX bekler.

#### 6.5. `SUPPORTS`

> "TX varsa katıl, yoksa TX'siz çalış."

Genelde okuma yapan helper'lar için:

```java
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public Optional<Account> findById(AccountId id) {
    // TX varsa o TX'in PC'sini kullan, yoksa TX'siz oku
}
```

Banking'de nadiren ihtiyaç olur. Daha çok altyapı kodu.

#### 6.6. `NOT_SUPPORTED`

> "Mevcut TX'i askıya al, TX'siz çalış, bitince eskiyi devam ettir."

Banking örneği — heavy reporting:

```java
@Service
class ReportingService {
    
    @Transactional   // ana TX
    public void generateReport(...) {
        // ...
        var summary = heavyAggregation();    // çok yavaş, lock tutmayalım
    }
    
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public Summary heavyAggregation() {
        // TX dışında — uzun süre lock tutmaz
        // ... 30 saniyelik query
    }
}
```

TX içindeyken uzun bir query çalıştırmak isolation/lock perspektifinden problemli. Bazen okuma için TX'siz çalıştırmak istenir.

**Tuzak:** İçeride yine DB call yapacaksın, autoCommit mode'da çalışacak. Repeatable read garantisi yok.

#### 6.7. `NEVER`

> "TX varsa **exception fırlat**, yoksa TX'siz çalış."

Çok nadiren — kesin TX dışı çalışması gereken (örn. native API'ye birebir geçen) kod için.

```java
@Transactional(propagation = Propagation.NEVER)
public void sendToExternalSystem(...) {
    // TX içinde external HTTP call YAPMA — DB lock'larını uzatma
}
```

Banking pratiği: External HTTP call'ları zaten TX dışında yapmalıyız (lock tutarak HTTP bekleme = felaket). NEVER ile compile-time-ish (runtime) garanti.

---

### Propagation özet tablosu

| Propagation | TX var | TX yok | Banking use case |
|---|---|---|---|
| REQUIRED | Katıl | Yeni başlat | Default — her şey |
| REQUIRES_NEW | Askıya al + yeni başlat | Yeni başlat | Audit log, notification |
| NESTED | Savepoint oluştur | Yeni başlat | Batch processing partial fail |
| MANDATORY | Katıl | Exception | TX zorunlu helper |
| SUPPORTS | Katıl | TX'siz | Esnek read helper |
| NOT_SUPPORTED | Askıya al + TX'siz | TX'siz | Heavy read TX dışı |
| NEVER | Exception | TX'siz | External call koruma |

---

### 7. Isolation level — 4 standart + DEFAULT

`Isolation` = eşzamanlı transaction'lar birbirini ne kadar görür.

ANSI SQL 4 isolation level tanımlar. Her biri belirli "read phenomenon"ları engeller:

| Phenomenon | Açıklama | Banking örneği |
|---|---|---|
| Dirty read | Başka TX'in henüz commit olmamış değişikliğini okumak | T1 bakiyeyi 1000 → 500 yaptı (commit YOK), T2 500 okuyup karar verdi, T1 rollback → T2 yanlış kararla devam |
| Non-repeatable read | Aynı TX içinde aynı satırı iki kez okuyunca farklı değer | T1 bakiyeyi okudu 1000, T2 100 yatırdı + commit, T1 tekrar okudu 1100 |
| Phantom read | Aynı TX içinde aynı sorguda yeni satırlar belirir | T1 "owner=X olan hesapları" listeledi (3 kayıt), T2 yeni hesap açtı + commit, T1 tekrar listeledi (4 kayıt) |

Lost update ve write skew gibi başkaları da var; bunları locking topic'inde göreceğiz.

#### Isolation level → phenomenon engelleme

| Level | Dirty read | Non-repeatable | Phantom |
|---|---|---|---|
| READ_UNCOMMITTED | ✗ olabilir | ✗ olabilir | ✗ olabilir |
| READ_COMMITTED | ✓ engellenir | ✗ olabilir | ✗ olabilir |
| REPEATABLE_READ | ✓ engellenir | ✓ engellenir | ✗ olabilir¹ |
| SERIALIZABLE | ✓ engellenir | ✓ engellenir | ✓ engellenir |

¹ MySQL InnoDB REPEATABLE_READ phantom'u da engeller (next-key locking). PostgreSQL REPEATABLE_READ snapshot isolation kullanır, phantom riskini ek mekanizma ile çözer.

#### Banking için pratik seçim

**PostgreSQL default: READ_COMMITTED.** Çoğu banking servisi bu seviyede çalışır.

- Dirty read engellenir (zaten istemeyiz)
- Aynı TX'te aynı satır iki kez okunca farklı değer **gelebilir** — ama transfer kısa sürdüğü için pratikte sorun olmaz
- Phantom okuyabilirsin — reporting query'lerinde dikkat

**REPEATABLE_READ ne zaman:** Bir TX içinde aynı veriyi birden fazla kez okuyup karar veriyorsan. Örn. "hesap bakiyesini oku, hesapla, tekrar oku, karşılaştır". Veya snapshot bazlı reporting.

**SERIALIZABLE ne zaman:** En sıkı. Concurrent transaction'lar **birbirine yol verir** veya **serialization failure** alır. Performans düşer. PostgreSQL'de SSI (Serializable Snapshot Isolation) implementasyonu çok zekidir. Banking'in en kritik para hareketlerinde nadiren kullanılır — genelde locking ile çözüm tercih.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalReconciliation() {
    // ...
}
```

**Mülakat sorusu (klasik):** "REPEATABLE_READ ile READ_COMMITTED arasında ne fark var?" → Non-repeatable read'i engelleyip engellememesi.

**Mülakat sorusu (zor):** "PostgreSQL'de REPEATABLE_READ kullanıyorsun. İki TX aynı satırı UPDATE etmeye çalışıyor. Ne olur?" → İkinci TX **serialization failure** alır (`could not serialize access due to concurrent update`), retry gerekir. (Bu topic 2.4'te detaylanacak.)

#### Isolation level set ederken dikkat

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void method() { ... }
```

- Spring her TX başında `connection.setTransactionIsolation(...)` çağırır.
- Bağlantı pool'a iade edildiğinde isolation **eski seviyeye dönmez** (default PostgreSQL JDBC driver). Sonraki TX bu connection'ı alırsa **yanlış isolation'la** çalışır. HikariCP'nin `connection.setTransactionIsolation`'ı bağlantı iadesinde reset etmesini sağla:

```yaml
spring.datasource.hikari.transaction-isolation: TRANSACTION_READ_COMMITTED  # connection-level default
```

Veya yeni Spring/Hibernate sürümlerinde `hibernate.connection.handling_mode` ile kontrollü.

---

### 8. `readOnly = true` — performans ve niyet

```java
@Transactional(readOnly = true)
public Account findById(AccountId id) { ... }
```

İki etkisi var:

**A — Hibernate optimizasyonu:** Persistence context **dirty checking yapmaz**. Snapshot tutmaz. Flush atılmaz. Memory + CPU tasarrufu.

**B — DB driver hint:** Bazı driver'lar `Connection.setReadOnly(true)` çağrısı yapar. DB hint olarak alabilir (Oracle, PostgreSQL). Read replica routing yapılıyorsa (Phase 9) replica'ya yönlendirme yapılır.

**Tuzak — readOnly gerçek bir lock değil:** Yanlışlıkla yazma yaparsan exception almazsın (Hibernate bazen). Sadece flush atmaz. Banking'de niyet beyanı olarak değerli.

**Banking pratiği:**

```java
@Service
@Transactional(readOnly = true)        // class-level default
public class AccountReportingService {
    
    public Page<Account> search(...) { ... }       // readOnly = true
    public Account getById(AccountId id) { ... }   // readOnly = true
    
    @Transactional                                   // method-level override
    public void recordReadAccess(AccountId id) {     // write için readOnly = false
        // ...
    }
}
```

Class-level read-only default, write için method-level override. Çok temiz.

---

### 9. `timeout` — TX süresi sınırı

```java
@Transactional(timeout = 5)        // 5 saniye
public void transfer(...) { ... }
```

5 saniye içinde commit veya rollback olmazsa `TransactionTimedOutException`. Sıradaki SQL call'ında patlar.

Banking pratiği:
- Para hareketleri **kısa** olmalı (< 1-2 saniye). Timeout 5 saniye = güvenlik kemeri.
- Long-running batch için ayrı bir TX manager veya REQUIRES_NEW + uzun timeout.
- Default sınırsız — production'da **kötü**. Her TX'e bir tavan koy.

`spring.transaction.default-timeout: 30` ile global default.

---

### 10. Rollback — checked vs unchecked exception

**Spring default rollback davranışı:**

- **Unchecked exception** (`RuntimeException`, `Error`) → **rollback** ✓
- **Checked exception** (`Exception` subclass, RuntimeException değil) → **commit** (!) ⚠

Evet, bu çoğu junior'ı şaşırtır. Sebep: EJB tarihi. Spring bu davranışı tutmuş.

```java
@Transactional
public void transfer(...) throws InsufficientFundsException {
    // InsufficientFundsException unchecked ise → rollback ✓
    // InsufficientFundsException checked (Exception extend) ise → commit (kötü!)
}
```

Banking'de domain exception'larını **unchecked** yap (RuntimeException extend). Hem rollback davranışı doğru, hem method signature gürültüsü yok.

```java
public class InsufficientFundsException extends RuntimeException { }
```

#### `rollbackFor` — checked'ı da rollback yap

```java
@Transactional(rollbackFor = SomeCheckedException.class)
public void method() throws SomeCheckedException {
    // ...
}
```

Veya hepsi için:

```java
@Transactional(rollbackFor = Exception.class)
```

Banking pratiği: Domain exception'lar zaten unchecked ise yeterli. External library checked exception fırlatıyorsa ve rollback istiyorsan `rollbackFor` ekle.

#### `noRollbackFor` — bazı durumlarda rollback istemiyorum

```java
@Transactional(noRollbackFor = NotificationFailedException.class)
public void transferAndNotify(...) {
    // transfer yap
    // notification gönderme dene — fail olursa transfer'i rollback ETME
}
```

Banking pratiği: Genelde **transfer fail = rollback** isteriz. noRollbackFor'u dikkatli kullan. Notification gibi yan etkileri zaten event-based (Phase 4) yapacağız.

---

### 11. Banking money transfer transaction tasarımı (klasik mülakat sorusu)

Hazırlan: "Bir para transferi servisi tasarla — debit, credit, audit log, notification."

**Tasarım:**

```java
@Service
public class TransferService {
    
    private final AccountRepository accountRepository;
    private final JournalRepository journalRepository;
    private final AuditService auditService;
    private final NotificationPublisher notificationPublisher;
    
    @Transactional                                    // REQUIRED — atomic transfer
    public Transfer execute(TransferRequest request) {
        var transferId = TransferId.generate();
        
        // 1. Try-catch içinde attempt log
        try {
            // 2. Doğrula
            if (request.from().equals(request.to())) {
                throw new SameAccountTransferException();
            }
            
            // 3. Hesapları çek
            var from = accountRepository.findById(request.from())
                .orElseThrow(() -> new AccountNotFoundException(request.from()));
            var to = accountRepository.findById(request.to())
                .orElseThrow(() -> new AccountNotFoundException(request.to()));
            
            // 4. Status kontrol
            if (from.getStatus() != ACTIVE) throw new AccountClosedException(from.getId());
            if (to.getStatus() != ACTIVE) throw new AccountClosedException(to.getId());
            
            // 5. Domain logic — debit + credit
            from.withdraw(request.amount(), transferId);   // InsufficientFundsException patlayabilir
            to.deposit(request.amount(), transferId);
            
            // 6. Persist
            accountRepository.save(from);
            accountRepository.save(to);
            
            // 7. Journal entry (double-entry)
            var journalEntry = JournalEntry.builder()
                .transferId(transferId)
                .addLine(from.getId(), DEBIT, request.amount())
                .addLine(to.getId(), CREDIT, request.amount())
                .build();
            journalRepository.save(journalEntry);
            
            // 8. Audit log — REQUIRES_NEW, transfer fail olsa bile yazılı kalsın
            // BURASI önemli: success log
            auditService.logSuccess(transferId, request);
            
            // 9. Notification — event publish (Phase 4 detay)
            // Şimdilik: event yayımla, listener async dispatch eder
            notificationPublisher.publish(new TransferCompleted(transferId, request));
            
            return new Transfer(transferId, ...);
            
        } catch (BankingException e) {
            // 10. Failed attempt log — REQUIRES_NEW
            // Ana TX rollback olacak ama log kalmalı
            auditService.logFailure(transferId, request, e);
            throw e;   // re-throw → ana TX rollback
        }
    }
}

@Service
public class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logSuccess(TransferId id, TransferRequest req) {
        auditRepo.save(AuditLog.success(id, req));
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logFailure(TransferId id, TransferRequest req, BankingException cause) {
        auditRepo.save(AuditLog.failure(id, req, cause.getMessage()));
    }
}
```

**Tasarım kararları:**

1. Ana TX `REQUIRED` (default) — transfer atomic.
2. Audit log `REQUIRES_NEW` — transfer rollback olsa bile log yazılır.
3. Domain exception'lar `RuntimeException` → otomatik rollback.
4. Notification **event** — TX'in dışına çık (Phase 4'te `@TransactionalEventListener` ile detay).
5. Idempotency-key bu örnekte gösterilmedi ama gerçek implementasyonda en başta.

**Mülakatçı follow-up:** "Audit log REQUIRES_NEW dedin. Audit DB'si yazılamazsa ne olur, transfer rollback mi?" Cevap: Audit log fail olduğunda transfer rollback olmamalı (audit log ana akışı bloklamaz). `try-catch` ile audit fail'ini yut, log'la (operator'a alert). Veya audit asynchronous yapı.

---

### 12. Nested transaction tuzakları

NESTED'ı kullanırken dikkat:

**Tuzak 1 — Driver desteği:**
Hibernate `Connection.setSavepoint()` kullanır. Bazı eski JDBC driver'larında destek eksik. PostgreSQL, MySQL, Oracle modern sürümlerde sorun yok.

**Tuzak 2 — Performans:**
Her NESTED bir savepoint sayar. Loop içinde 10000 NESTED = 10000 savepoint = DB yorulur. NESTED'ı bilinçli kullan, batch için alternatif yöntemler düşün.

**Tuzak 3 — Hata yakalama:**
İçeride exception fırlarsa savepoint rollback olur. Ama sen exception'ı yakalamazsan dış TX'e iletilir, **tüm ana TX rollback** olur.

```java
@Transactional
public void batch(List<X> items) {
    for (X x : items) {
        try {
            processOne(x);   // NESTED
        } catch (BusinessException e) {
            // sadece bu kayıt rollback, devam
            logError(x, e);
        }
        // catch yoksa → ana TX rollback
    }
}
```

**Tuzak 4 — JPA flush davranışı:**
Hibernate PC'sini NESTED için tam doğru yönetmeyebilir. NESTED rollback sonrası PC kirli kalabilir. Banking için: NESTED kullanıyorsan kapsamlı test, gerekirse manuel `em.clear()` ile temizle.

**Banking pratiği:** Mümkünse NESTED yerine REQUIRES_NEW veya batch'i parçala. NESTED'ı tek bir sebep için kullan: "tek TX'te kalmak istiyorum ama belli bir alt-iş rollback olsa diğerleri devam etsin."

---

### 13. Transaction içinde external call ETME — neden

```java
@Transactional
public void transfer(...) {
    // ... DB iş
    
    swiftClient.sendMessage(...);   // ❌ HTTP call TX içinde
    
    // ... DB iş devam
}
```

**Niye kötü:**

1. **DB lock'ları açıkken HTTP bekliyorsun.** External 30 saniye yanıt verirse, lock'lar 30 saniye duruyor. Diğer transaction'lar bekliyor. Connection pool tükeniyor.

2. **External fail olursa rollback davranışı belirsiz.** SWIFT mesajı gitti mi? Eğer sen TX'i commit edersen DB ile external desync olur.

3. **Timeout zincirleme patlama:** External 30 sn yanıt, TX timeout 10 sn → TX timeout patlar, ama external mesajı belki gönderildi.

**Doğrusu:** External call'ları **TX'in dışına çıkar**.

```java
@Transactional
public Transfer execute(...) {
    // ... saf DB iş
    return transferRepo.save(transfer);
}

// Caller seviyesi:
public void executeAndNotify(...) {
    var transfer = transferService.execute(...);   // TX commit
    swiftClient.sendMessage(transfer);              // TX dışında
}
```

Veya **`@TransactionalEventListener(phase = AFTER_COMMIT)`** ile event-driven (Phase 4 detay).

---

### 14. Transaction içinde event publish — `@TransactionalEventListener`

Spring 4.2+ event listener'ı transaction phase'ine bağlayabilir:

```java
@Service
class TransferService {
    private final ApplicationEventPublisher publisher;
    
    @Transactional
    public void execute(...) {
        // ... iş
        publisher.publishEvent(new TransferCompleted(transferId, ...));
        // event PC'de bekler, commit sonrası listener'a gider
    }
}

@Component
class NotificationListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onTransferCompleted(TransferCompleted event) {
        // Burası TX commit ETTİKTEN sonra çalışır
        // Hata fırlarsa ana TX'e dönmez (zaten commit oldu)
    }
}
```

**Phase seçenekleri:**
- `BEFORE_COMMIT` — commit'ten hemen önce (validation gibi)
- `AFTER_COMMIT` (default) — commit başarılıysa
- `AFTER_ROLLBACK` — rollback olursa
- `AFTER_COMPLETION` — her durumda (commit veya rollback)

Banking pratiği: Notification, audit-log-async, downstream-system-sync için AFTER_COMMIT. Çok yaygın pattern.

---

### 15. Banking anti-pattern'leri

**Anti-pattern 1: Controller'a `@Transactional`**

```java
@RestController
class TransferController {
    @PostMapping
    @Transactional   // ❌ TX scope = HTTP request süresi
    public TransferResponse transfer(...) { ... }
}
```

Sorunlar: TX HTTP süresi boyunca açık (lazy loading, serialization vs hepsi TX içinde). Service ayrımı kaybolur. Connection 50 ms yerine 500 ms tutulur.

**Doğru:** `@Transactional` service'te.

**Anti-pattern 2: Domain method'una `@Transactional`**

Domain class'lar Spring bilmemeli. `@Transactional` orchestration sorumluluğu, domain'in değil.

**Anti-pattern 3: `catch (Exception e) { ... }` ile TX yutmak**

```java
@Transactional
public void transfer(...) {
    try {
        from.withdraw(...);
        to.deposit(...);
    } catch (Exception e) {
        log.error("Failed", e);   // ❌ exception yutuldu, TX commit olur
    }
}
```

Spring rollback'i sadece **exception fırlarsa** veya `setRollbackOnly()` ile işaretlenirse yapar. Yutarsan commit olur. Yarım transfer DB'ye yazılır.

**Doğru:** Re-throw veya `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

**Anti-pattern 4: Uzun TX**

```java
@Transactional
public void process() {
    var list = repo.findAll();              // 1M kayıt
    for (var item : list) {
        externalApi.call(item);              // her biri 200 ms
        repo.save(item);
    }
}
```

TX 200000 saniye açık kalır. Lock'lar, connection, PC memory — felaket.

**Doğru:** Batch'i parçala, her batch ayrı TX. External call TX dışı.

**Anti-pattern 5: `@Transactional` her yere**

`AccountController` → `AccountService` (@Transactional) → `BalanceCalculator` (@Transactional) → ... her layer'da. Karmaşıklığı patlatır. Default REQUIRED nested davranışı bilinmeyince debug zor.

**Doğru:** TX boundary'sini bilinçli koy. Genelde service entry point'i.

**Anti-pattern 6: Self-invocation tuzaklı kod (tekrar)**

```java
@Service
class AccountService {
    public void publicMethod() {
        privateTxMethod();   // ❌ proxy bypass
    }
    
    @Transactional
    public void privateTxMethod() { ... }
}
```

Yukarıda detaylandı. Ayrı bean'e taşı.

**Anti-pattern 7: Read-only method'da yazma**

```java
@Transactional(readOnly = true)
public Account getAndUpdateLastSeen(AccountId id) {
    var account = repo.findById(id).orElseThrow();
    account.setLastSeenAt(Instant.now());     // ❌ readOnly = true ama yazıyorsun
    return account;
}
```

Hibernate flush atmayabilir, değişiklik kaybolabilir. Veya driver `Connection.setReadOnly(true)` yapmışsa DB hata fırlatır.

**Doğru:** Niyet yazma → `readOnly = false`.

---

## Önemli olabilecek araştırma kaynakları

- Spring Framework Reference — Transaction Management bölümü
- "High-Performance Java Persistence" Vlad Mihalcea — transaction internals
- "Java Transaction Design Strategies" Mark Richards (kısa kitap)
- PostgreSQL Documentation — Concurrency Control
- "Designing Data-Intensive Applications" Martin Kleppmann — Chapter 7 (Transactions)
- Baeldung — Spring Transactional Propagation
- Thorben Janssen blog — JPA transaction posts
- Marco Behler — "Spring Boot Transactions Tutorial"

---

## Mini task'ler

### Task 2.3.1 — Self-invocation problemi reprodüksiyonu (30 dk)

`core-banking` projesine bir test class'ı ekle. İki versiyon:

**A — broken:**

```java
@Service
class TransferAuditServiceBroken {
    
    @Autowired AuditLogRepository auditRepo;
    
    public void doTransferAndLog(TransferRequest req) {
        executeTransfer(req);             // self-invocation
        writeAuditLog(req);
    }
    
    @Transactional
    void executeTransfer(TransferRequest req) {
        // ... balance updates
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void writeAuditLog(TransferRequest req) {
        auditRepo.save(new AuditLog(req));
    }
}
```

**B — fixed:** Ayrı `AuditService` class.

Bir SQL log'lu integration test yaz, A versiyonunda `BEGIN`-`COMMIT` görmeyeceksin (TX yok). B'de iki ayrı TX göreceksin.

Defterine yaz: "Self-invocation problemi neden olur, nasıl tespit ederim, nasıl düzeltirim."

### Task 2.3.2 — Propagation deney matrisi (45 dk)

`PropagationExperiment` test class'ı yaz. Her propagation tipi için:

1. Dış TX olmadan inner method çağır → davranışı gözlemle
2. Dış TX içinde inner method çağır → davranışı gözlemle

Beklenen sonuçları **önce tahmin et**, sonra test'i çalıştır:

| Propagation | TX yok | TX var |
|---|---|---|
| REQUIRED | yeni TX | join |
| REQUIRES_NEW | yeni TX | suspend + new |
| NESTED | yeni TX | savepoint |
| MANDATORY | exception | join |
| SUPPORTS | no TX | join |
| NOT_SUPPORTED | no TX | suspend + no TX |
| NEVER | no TX | exception |

Her satır için bir test yaz. Davranışı log'da (BEGIN, COMMIT, ROLLBACK) gör.

### Task 2.3.3 — REQUIRES_NEW audit log senaryosu (45 dk)

`AuditService` yaz:

```java
@Service
class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logFailure(TransferId id, String reason) {
        auditRepo.save(new AuditLog(id, "FAILED", reason, Instant.now()));
    }
}
```

`TransferService.execute(...)` içinde:

```java
@Transactional
public Transfer execute(TransferRequest req) {
    try {
        // ... transfer logic
        if (from.getBalance().isLessThan(req.amount())) {
            auditService.logFailure(transferId, "INSUFFICIENT_FUNDS");
            throw new InsufficientFundsException(...);
        }
        // ...
    } catch (...) {
        throw ...;
    }
}
```

Test: yetersiz bakiyeli transfer at → ana TX rollback olmalı (account balance değişmemeli) **ama** audit log yazılmış olmalı. SQL log'da iki ayrı TX gör.

### Task 2.3.4 — Isolation level deneyleri (60 dk)

`@SpringBootTest` ile iki paralel thread başlat. Her birinde farklı TX. Şu phenomenon'ları **reprodüksiyon** dene:

**Dirty read (READ_UNCOMMITTED ile):**
- T1: balance'ı 1000 → 500 yap (commit YOK, sleep 2 sn)
- T2: aynı hesabın balance'ını oku
- T1: rollback
- T2 ne okudu? READ_UNCOMMITTED'da 500, READ_COMMITTED'da 1000

Not: PostgreSQL READ_UNCOMMITTED'ı destekler **ama internal olarak READ_COMMITTED gibi davranır**. PostgreSQL'de dirty read'i reprodüksiyon edemezsin. Anekdot olarak bil.

**Non-repeatable read (READ_COMMITTED vs REPEATABLE_READ):**
- T1: balance oku (1000), sleep 2 sn, balance tekrar oku
- T2: balance'ı 1000 → 1500 yap + commit (T1 sleeperken)
- T1 ikinci okuma: READ_COMMITTED'da 1500, REPEATABLE_READ'de 1000

Bunu **reprodüksiyon yap**. Test'i çalıştır, sonucu gör. Defterine yaz.

**Phantom read:**
- T1: `WHERE owner_id = X` ile hesap listele (3 satır), sleep
- T2: aynı owner'a yeni hesap aç + commit
- T1: aynı sorgu tekrar (4 satır?)

PostgreSQL REPEATABLE_READ'de snapshot isolation, phantom da görünmez (MVCC). MySQL InnoDB REPEATABLE_READ'de next-key lock ile phantom blok.

### Task 2.3.5 — Rollback davranışı: checked vs unchecked (30 dk)

İki exception class:

```java
public class CheckedBusinessException extends Exception { }
public class UncheckedBusinessException extends RuntimeException { }
```

İki service method:

```java
@Transactional
public void methodWithChecked() throws CheckedBusinessException {
    accountRepo.save(...);
    throw new CheckedBusinessException();
}

@Transactional
public void methodWithUnchecked() {
    accountRepo.save(...);
    throw new UncheckedBusinessException();
}
```

Her ikisini integration test'le çağır. DB'de kayıt **var mı** kontrol et.

- methodWithChecked → kayıt var (commit oldu!) ⚠
- methodWithUnchecked → kayıt yok (rollback)

Sonra `@Transactional(rollbackFor = CheckedBusinessException.class)` ekle, tekrar dene. Bu sefer kayıt yok.

Defter notu: "Spring default rollback davranışı şu yüzden RuntimeException-only..."

### Task 2.3.6 — TransferService transaction tasarımı (60 dk)

`core-banking` `TransferService`'ini yukarıdaki "banking money transfer transaction tasarımı" bölümündeki yapıya getir:

- Ana TX REQUIRED
- AuditService REQUIRES_NEW
- Domain exception'lar RuntimeException
- Notification için event publish (`@TransactionalEventListener(AFTER_COMMIT)` listener da ekle)
- timeout = 10 saniye
- readOnly = false (default)

`AuditService` class'ında `logSuccess` ve `logFailure` method'ları olsun, ikisi de REQUIRES_NEW.

Test: 
- Başarılı transfer → 1 audit (success), event yayımlandı, balance'lar değişti
- Insufficient funds → 1 audit (failure), event yayımlanmadı, balance'lar değişmedi

### Task 2.3.7 — `@TransactionalEventListener` ile notification (45 dk)

```java
public record TransferCompleted(TransferId id, AccountId from, AccountId to, Money amount, Instant at) { }

@Service
class TransferService {
    private final ApplicationEventPublisher publisher;
    
    @Transactional
    public Transfer execute(TransferRequest req) {
        // ...
        publisher.publishEvent(new TransferCompleted(...));
        return transfer;
    }
}

@Component
class TransferNotificationListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onTransferCompleted(TransferCompleted event) {
        log.info("Notification: transfer {} from {} to {}", event.id(), event.from(), event.to());
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onTransferFailed(TransferCompleted event) {
        // Bu çağrılmaz çünkü rollback olursa event publish edilse de listener
        // sadece success phase'i ile bağlıysa çalışmaz. Test et.
    }
}
```

Test:
- Başarılı transfer → AFTER_COMMIT listener çalıştı
- Insufficient funds (rollback) → AFTER_COMMIT listener **çalışmadı**

### Task 2.3.8 — readOnly performans gözlemi (30 dk)

`AccountReportingService`'i iki versiyon yaz:

```java
@Service
public class AccountReportingService {
    
    @Transactional   // readOnly = false (default)
    public List<Account> listV1(OwnerId ownerId) {
        return accountRepo.findByOwnerId(ownerId);
    }
    
    @Transactional(readOnly = true)
    public List<Account> listV2(OwnerId ownerId) {
        return accountRepo.findByOwnerId(ownerId);
    }
}
```

Hibernate statistics aç. 10000 kez her ikisini çağır. `EntityInsertCount`, `EntityUpdateCount`, `FlushCount` farkını gözlemle.

readOnly = true'da flush count 0 olmalı. Defterine yaz.

---

## Test yazma rehberi

### Test 2.3.1 — Self-invocation tespiti

```java
@SpringBootTest
@Testcontainers
class SelfInvocationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired TransferAuditServiceBroken brokenService;
    @Autowired TransferAuditServiceFixed fixedService;
    @Autowired AuditLogRepository auditRepo;
    
    @Test
    void brokenServiceShouldNotOpenTransactionForInnerMethod() {
        // Hibernate statistics ile TX sayımı veya SQL log capture
        var statsBefore = getCommitCount();
        
        brokenService.doTransferAndLog(testRequest());
        
        var statsAfter = getCommitCount();
        
        // Beklenen: 0 commit (TX yok) veya 1 (default TX'de hepsi)
        // Asla 2 commit (REQUIRES_NEW devreye girmedi)
        assertThat(statsAfter - statsBefore).isLessThan(2);
    }
    
    @Test
    void fixedServiceShouldOpenSeparateTransactions() {
        var statsBefore = getCommitCount();
        
        fixedService.doTransferAndLog(testRequest());
        
        var statsAfter = getCommitCount();
        assertThat(statsAfter - statsBefore).isEqualTo(2);   // outer + inner REQUIRES_NEW
    }
}
```

### Test 2.3.2 — REQUIRES_NEW audit log persistence

```java
@Test
void auditLogShouldPersistEvenWhenTransferFails() {
    var fromAccount = createAccountWithBalance(BigDecimal.valueOf(100));
    var toAccount = createAccountWithBalance(BigDecimal.ZERO);
    
    var req = new TransferRequest(fromAccount.getId(), toAccount.getId(), 
                                   Money.of("1000.00", "TRY"), "test");
    
    // Insufficient funds expected
    assertThatThrownBy(() -> transferService.execute(req))
        .isInstanceOf(InsufficientFundsException.class);
    
    // Audit log YAZILMIŞ olmalı (REQUIRES_NEW)
    var logs = auditLogRepo.findAll();
    assertThat(logs).hasSize(1);
    assertThat(logs.get(0).getStatus()).isEqualTo("FAILED");
    assertThat(logs.get(0).getReason()).contains("INSUFFICIENT_FUNDS");
    
    // Balance'lar DEĞİŞMEMİŞ olmalı (ana TX rollback)
    var fromReloaded = accountRepo.findById(fromAccount.getId()).orElseThrow();
    assertThat(fromReloaded.getBalance()).isEqualTo(Money.of("100.00", "TRY"));
}
```

### Test 2.3.3 — Non-repeatable read reprodüksiyon

```java
@Test
void nonRepeatableReadShouldOccurInReadCommitted() throws Exception {
    var accountId = createAccount("TRY", "1000.00").getId();
    
    var firstRead = new AtomicReference<BigDecimal>();
    var secondRead = new AtomicReference<BigDecimal>();
    
    var t1 = new Thread(() -> {
        transactionTemplate(Isolation.READ_COMMITTED).execute(status -> {
            firstRead.set(accountRepo.findById(accountId).orElseThrow().getBalanceAmount());
            sleep(2000);
            secondRead.set(accountRepo.findById(accountId).orElseThrow().getBalanceAmount());
            return null;
        });
    });
    
    var t2 = new Thread(() -> {
        sleep(500);   // T1'in ilk okumasından sonra
        transactionTemplate(Isolation.READ_COMMITTED).execute(status -> {
            var a = accountRepo.findById(accountId).orElseThrow();
            a.setBalanceAmount(new BigDecimal("1500.00"));
            return accountRepo.save(a);
        });
    });
    
    t1.start(); t2.start();
    t1.join(); t2.join();
    
    // READ_COMMITTED'da firstRead != secondRead
    assertThat(firstRead.get()).isEqualByComparingTo("1000.00");
    assertThat(secondRead.get()).isEqualByComparingTo("1500.00");
}

@Test
void repeatableReadShouldPreventNonRepeatable() {
    // Aynı setup, Isolation.REPEATABLE_READ
    // firstRead = secondRead = 1000.00 olmalı
}
```

### Test 2.3.4 — Rollback davranışı

```java
@Test
void uncheckedExceptionShouldTriggerRollback() {
    var startCount = accountRepo.count();
    
    assertThatThrownBy(() -> service.saveAndThrowUnchecked())
        .isInstanceOf(UncheckedBusinessException.class);
    
    assertThat(accountRepo.count()).isEqualTo(startCount);   // rollback
}

@Test
void checkedExceptionShouldNotTriggerDefaultRollback() {
    var startCount = accountRepo.count();
    
    assertThatThrownBy(() -> service.saveAndThrowChecked())
        .isInstanceOf(CheckedBusinessException.class);
    
    assertThat(accountRepo.count()).isEqualTo(startCount + 1);   // commit (!) — kötü
}

@Test
void checkedWithRollbackForShouldTriggerRollback() {
    var startCount = accountRepo.count();
    
    assertThatThrownBy(() -> service.saveAndThrowCheckedWithRollbackFor())
        .isInstanceOf(CheckedBusinessException.class);
    
    assertThat(accountRepo.count()).isEqualTo(startCount);   // rollback
}
```

### Test 2.3.5 — `@TransactionalEventListener` AFTER_COMMIT

```java
@Test
void listenerShouldRunAfterCommit() {
    var counter = new AtomicInteger(0);
    
    // Listener'ı counter increment edecek şekilde mock veya spy
    
    transferService.execute(validTransferRequest());
    
    await().atMost(2, SECONDS).untilAsserted(() -> {
        assertThat(counter.get()).isEqualTo(1);   // bir kez çalıştı
    });
}

@Test
void listenerShouldNotRunOnRollback() {
    var counter = new AtomicInteger(0);
    
    assertThatThrownBy(() -> transferService.execute(invalidRequest()))
        .isInstanceOf(BankingException.class);
    
    sleep(1000);
    assertThat(counter.get()).isZero();   // listener çalışmadı (AFTER_COMMIT phase)
}
```

---

## Claude-verify prompt

```
Aşağıdaki Spring transaction management kodumu banking-grade kriterlere göre 
değerlendir. Eksikleri işaretle, kod yazma:

1. @Transactional yerleştirme:
   - Service katmanında mı (controller veya domain'de değil)?
   - Class-level mi yoksa method-level mi — bilinçli mi?
   - Read-only method'larda readOnly = true var mı?

2. Propagation:
   - Default REQUIRED kullanılmış mı?
   - REQUIRES_NEW audit log gibi "her zaman yazılmalı" işler için kullanılmış mı?
   - Self-invocation problemi var mı (aynı bean içinden @Transactional method çağrısı)?
   - Eğer var, ayrı bean'e taşınmış mı?

3. Isolation:
   - Default kullanılmış mı (genelde READ_COMMITTED)?
   - Daha sıkı isolation gerekiyorsa gerekçesi yorum olarak var mı?

4. Rollback davranışı:
   - Domain exception'lar RuntimeException extend ediyor mu?
   - Checked exception fırlatan method'larda rollbackFor eklenmiş mi?
   - Exception yutan catch blokları var mı (anti-pattern)?

5. readOnly:
   - Reporting servislerinin tümü readOnly = true mu?
   - readOnly method içinde yanlışlıkla yazma yapan kod var mı?

6. Timeout:
   - @Transactional'lara timeout konmuş mu (en azından class-level default)?
   - Long-running TX riski olan yerlerde özel timeout var mı?

7. Transaction içinde external call:
   - HTTP / Kafka / SMS call'ları TX içinde mi (yapılmamalı)?
   - Notification için @TransactionalEventListener(AFTER_COMMIT) kullanılmış mı?
   - REQUIRES_NEW veya event-based yaklaşım tercih edilmiş mi?

8. NESTED kullanımı:
   - NESTED bilinçli mi (REQUIRES_NEW alternatif)?
   - Loop içinde NESTED kullanılıyorsa performans değerlendirilmiş mi?

9. Banking transfer transaction tasarımı:
   - Debit + credit + journal_entry aynı TX'te mi (atomicity)?
   - Audit log REQUIRES_NEW mu?
   - Notification AFTER_COMMIT mi (TX dışında)?
   - Idempotency key TX başında check ediliyor mu?

10. Anti-pattern:
    - Controller'a @Transactional konmuş mu? (Olmamalı)
    - Domain class'ına @Transactional konmuş mu? (Olmamalı)
    - Self-invocation pattern var mı?
    - "catch (Exception e) { log... }" ile TX yutan kod var mı?
    - 1M+ kayıt üzerinde tek TX ile çalışan kod var mı?

11. Test coverage:
    - Self-invocation reprodüksiyon test'i var mı?
    - REQUIRES_NEW audit log test'i (transfer fail + audit kalma) var mı?
    - Rollback davranışı test edildi mi (checked vs unchecked)?
    - Isolation level test'leri var mı (non-repeatable read reprodüksiyon)?

Her madde için PASS / FAIL / EKSIK işaretle, kanıt göster, kod yazma.
```

---

## Tamamlama kriterleri

- [ ] `@Transactional` proxy mekanizmasını (JDK vs CGLIB) 2 dakikada anlatabilirim
- [ ] Self-invocation problemini reprodüksiyon ettim ve fix uyguladım
- [ ] 7 propagation tipi için banking use case'i söyleyebilirim
- [ ] REQUIRES_NEW audit log pattern'ini `core-banking` projesinde implemente ettim
- [ ] 4 isolation level ve 3 phenomenon (dirty, non-repeatable, phantom) eşleştirmesini biliyorum
- [ ] PostgreSQL'de non-repeatable read'i reprodüksiyon ettim ve REPEATABLE_READ ile engelledim
- [ ] Checked vs unchecked exception default rollback davranışını test ile gördüm
- [ ] Domain exception'larım `RuntimeException` extend ediyor
- [ ] `TransferService` ana TX REQUIRED + audit REQUIRES_NEW + notification AFTER_COMMIT pattern'inde
- [ ] `@Transactional(timeout = N)` ile timeout koymanın önemini biliyorum
- [ ] Transaction içinde external HTTP call yapmamamın 3 sebebini sayabilirim
- [ ] `@TransactionalEventListener(AFTER_COMMIT)` ile event-based notification yazdım
- [ ] Anti-pattern listesi rahat — özellikle "controller'a @Transactional", self-invocation, exception yutma

---

## Defter notları

1. "Spring `@Transactional` proxy: ____ (JDK vs CGLIB ne zaman hangisi)."
2. "Self-invocation problemi neden olur, 2 ayrı çözüm: ____ ve ____."
3. "Propagation.REQUIRES_NEW vs NESTED farkı: ____, banking use case farkı: ____."
4. "Bir TX REQUIRES_NEW açtı, dış TX rollback oldu — iç TX commit mi rollback mi: ____."
5. "READ_COMMITTED ile REPEATABLE_READ farkı: ____ (engellediği phenomenon)."
6. "PostgreSQL REPEATABLE_READ ile MySQL InnoDB REPEATABLE_READ farkı: ____."
7. "Checked exception default'ta rollback YAPMAZ çünkü ____. Banking'de domain exception'ları nasıl yazarım: ____."
8. "TX içinde external HTTP call yapmamamın 3 sebebi: ____, ____, ____."
9. "`@TransactionalEventListener(AFTER_COMMIT)` ne zaman kullanılır, alternatifleri: ____."
10. "Banking transfer'inde ana TX, audit, notification için propagation seçimleri: ____."
