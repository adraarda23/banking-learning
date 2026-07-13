# Topic 3.8 — Structured Concurrency

## Hedef

Java 21'in **structured concurrency** API'sini (preview/incubator) öğrenmek. `StructuredTaskScope` ile **scope-bounded concurrent task'lar** yönetmek. `ShutdownOnFailure`, `ShutdownOnSuccess` ve custom `Joiner` pattern'lerini banking örnekleriyle uygulamak. `CompletableFuture` ile karşılaştırarak ne zaman hangisini seçeceğini bilmek.

## Süre

Okuma: 1.5 saat • Mini task: 1.5 saat • Test: 45 dk • Toplam: ~4 saat

## Önbilgi

- Topic 3.7 (Virtual Threads) bitti
- `CompletableFuture` (Topic 3.5) biliyorsun
- `ExecutorService` lifecycle (shutdown, awaitTermination) aşinasın
- Java 21+ kurulu, `--enable-preview` flag kullanmaya açıksın

---

## Kavramlar

### 1. Unstructured concurrency — bugünkü problem

Klasik `CompletableFuture` veya `ExecutorService` örneği:

```java
ExecutorService exec = Executors.newCachedThreadPool();
Future<Account> fromFuture = exec.submit(() -> repo.findById(from));
Future<Account> toFuture = exec.submit(() -> repo.findById(to));
Future<FxRate> rateFuture = exec.submit(() -> fxClient.getRate(...));

try {
    Account fromAcc = fromFuture.get();   // bloks
    Account toAcc = toFuture.get();
    FxRate rate = rateFuture.get();
    // ...
} catch (ExecutionException e) {
    // birinin failure'unu gör
}
// peki diğerleri hâlâ çalışıyor mu?
```

**Problemler:**

1. **Cancellation kontrolsüz:** Biri fail edince diğerleri **otomatik durmaz**. Manuel `cancel` çağırman gerekir, çoğu zaman unutulur.

2. **Lifecycle takip etmez parent-child:** Subtask'lar parent scope'tan bağımsız çalışır. Parent return etse bile subtask'lar arka planda çalışmaya devam edebilir.

3. **Stack trace dağınık:** Subtask exception'ları farklı thread'de fırlar, root cause takibi zor.

4. **Resource leak riski:** ExecutorService shutdown unutursa thread leak.

5. **Error handling fragmente:** Her future için ayrı `try-catch`, kolay yanlış yapılır.

### 2. Structured concurrency — "scope = lifetime"

**Prensip:** Subtask'ların **yaşam süresi** parent scope'la **sınırlı**. Scope sonlanmadan tüm subtask'lar tamamlanmış veya iptal edilmiş olur.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    
    Subtask<Account> fromTask = scope.fork(() -> repo.findById(from));
    Subtask<Account> toTask = scope.fork(() -> repo.findById(to));
    Subtask<FxRate> rateTask = scope.fork(() -> fxClient.getRate(...));
    
    scope.join();                     // hepsini bekle
    scope.throwIfFailed();            // biri fail ettiyse fırlat
    
    // Buraya gelinen yer = hepsi başarılı
    Account fromAcc = fromTask.get();
    Account toAcc = toTask.get();
    FxRate rate = rateTask.get();
    
    return doTransfer(fromAcc, toAcc, rate);
}
// Scope bitiminde tüm subtask'lar tamamlanmış (veya iptal edilmiş)
```

**Garantiler:**

1. `try-with-resources` blok bitiminde scope kapanır — **hiçbir subtask kaçamaz**
2. **`ShutdownOnFailure`:** Bir subtask fail ederse **diğerleri otomatik iptal** edilir (Thread.interrupt çağrılır)
3. Parent thread `join()` ile **mantıksal olarak çocuk task'ların bitmesini bekler**
4. **Hierarchical** — bir scope içinde başka scope açabilirsin, doğal ağaç

### 3. `ShutdownOnFailure` — banking en yaygın senaryo

"Tüm subtask'lar başarılı olmalı, biri fail ederse hepsini iptal et."

```java
public TransferResult executeTransfer(UUID fromId, UUID toId, Money amount) 
        throws InterruptedException, ExecutionException {
    
    try (var scope = new StructuredTaskScope.ShutdownOnFailure("transfer-scope", 
            Thread.ofVirtual().factory())) {
        
        Subtask<Account> fromTask = scope.fork(() -> accountService.findById(fromId));
        Subtask<Account> toTask = scope.fork(() -> accountService.findById(toId));
        Subtask<FraudScore> fraudTask = scope.fork(() -> fraudService.check(fromId, amount));
        
        scope.join();                      // bekle
        scope.throwIfFailed();             // hata varsa fırlat
        
        // Hepsi başarılı
        Account from = fromTask.get();
        Account to = toTask.get();
        FraudScore fraud = fraudTask.get();
        
        if (fraud.isRisky()) {
            throw new HighRiskTransferException();
        }
        
        return executeTransferInternal(from, to, amount);
    }
}
```

**Süreçte ne olur:**

1. `scope.fork()` her bir görevi **virtual thread**'de başlatır
2. 3 task paralel çalışır
3. `scope.join()` hepsinin bitmesini bekler **veya** bir fail olunca diğerlerini cancel eder
4. `scope.throwIfFailed()` ilk fail'i `ExecutionException` ile fırlatır
5. `try-with-resources` bitiminde scope kapanır, hâlâ çalışan thread varsa interrupt

### 4. `ShutdownOnSuccess` — ilk başarılı yeter

"Birden fazla provider'a sor, ilk doğru cevap veren kazanır, diğerlerini iptal et."

```java
public FxRate getFxRate(Currency from, Currency to) 
        throws InterruptedException, ExecutionException {
    
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<FxRate>()) {
        
        scope.fork(() -> tcmbProvider.getRate(from, to));
        scope.fork(() -> ecbProvider.getRate(from, to));
        scope.fork(() -> fixerProvider.getRate(from, to));
        
        scope.join();
        return scope.result();   // ilk başarılı olanın sonucu
    }
}
```

**Banking örneği — paralel KYC bürosu sorgulaması:**

```java
public KycResult performKyc(UUID customerId) 
        throws InterruptedException, ExecutionException {
    
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<KycResult>()) {
        
        scope.fork(() -> kkbBureau.check(customerId));     // KKB
        scope.fork(() -> findeksBureau.check(customerId)); // Findeks
        scope.fork(() -> internalBureau.check(customerId));
        
        scope.join();
        return scope.result();   // hangisi önce dönerse
    }
}
```

`result()` ilk başarılı sonucu döner. **Diğerleri otomatik iptal**.

Eğer **hepsi fail ederse**: `result()` `ExecutionException` fırlatır.

### 5. Custom Joiner (Java 24+ API'si)

Java 21'de `ShutdownOnFailure` ve `ShutdownOnSuccess` mevcut. Java 24 (preview) ile custom `Joiner` ile **kendi shutdown stratejini** yazabiliyorsun:

```java
// Pseudocode — Java 24 preview API
var scope = StructuredTaskScope.open(
    Joiner.allSuccessfulOrThrow()   // tüm başarılı sonuçları topla
);
```

Burada **Java 21'in stable API'si** ile devam edeceğiz. Custom shutdown logic için manuel `state.cancel()` çağrısı yapabilirsin.

### 6. Subtask state machine

```java
Subtask<T> task = scope.fork(...);

task.state();   // UNAVAILABLE → SUCCESS / FAILED
task.get();     // SUCCESS ise sonuç, değilse IllegalStateException
task.exception();   // FAILED ise exception
```

State'ler:
- `UNAVAILABLE` — henüz tamamlanmadı
- `SUCCESS` — başarılı
- `FAILED` — exception fırlattı

`scope.join()` SONRA `get()` çağırılmalı.

### 7. ScopedValue ile bağlamı taşıma

Structured concurrency ile **ScopedValue mükemmel uyum**:

```java
private static final ScopedValue<UserContext> USER_CTX = ScopedValue.newInstance();

public TransferResult transfer(UUID userId, Money amount) throws Exception {
    ScopedValue.where(USER_CTX, new UserContext(userId)).call(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            scope.fork(this::stepA);   // bu task USER_CTX.get() ile erişir
            scope.fork(this::stepB);
            scope.join();
            scope.throwIfFailed();
            return computeResult();
        }
    });
}

Account stepA() {
    UserContext ctx = USER_CTX.get();   // parent scope'tan otomatik aktarılır
    return doStepA(ctx);
}
```

**ScopedValue** scope-bounded — `ThreadLocal`'ın aksine virtual thread'lerle iyi çalışır, scope sonunda otomatik unbinding.

### 8. Exception handling

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> riskyOp1());
    scope.fork(() -> riskyOp2());
    
    scope.join();
    
    try {
        scope.throwIfFailed();
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();   // gerçek hata
        if (cause instanceof InsufficientFundsException) {
            // banking-specific handling
        }
        throw cause;
    }
}
```

`throwIfFailed()` `ExecutionException`'ın içinde gerçek `cause`'u sarıyor. Unwrap'lemen gerekir.

**Multi-failure** durumunda: `ShutdownOnFailure` ilk fail'i tutar, diğerleri kaybolur. Hepsini toplamak istersen custom joiner veya manuel toplama yapman gerekir.

### 9. Timeout handling

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(...);
    scope.fork(...);
    
    boolean finished = scope.joinUntil(Instant.now().plusSeconds(5));
    if (!finished) {
        // 5 sn içinde bitmedi → scope close + cancel
        throw new TimeoutException();
    }
    scope.throwIfFailed();
}
```

`joinUntil(Instant)` belirli zamana kadar bekler, geçerse `InterruptedException` veya scope kapanması.

**Banking örneği:** FX rate fetch için 2 sn timeout. 2 sn'de cevap gelmezse fallback.

### 10. CompletableFuture vs Structured Concurrency — karar matrisi

| Kriter | CompletableFuture | Structured Concurrency |
|---|---|---|
| Karmaşık async chain (compose, map) | ✓ iyi | sınırlı |
| Lifecycle bounded | dağınık | ✓ otomatik |
| Exception propagation | wrap'li | daha net |
| Stack trace | parçalı | daha temiz |
| Cancellation | manuel | ✓ otomatik |
| Java version | her | 21+ preview |
| Banking use case | reactive pipeline | scoped batch |

**Pratik kural:**

- **CompletableFuture** — reactive chain, complex flow, library API'leri (CompletionStage)
- **Structured Concurrency** — banking transaction'larında "şu 3 şeyi paralel yap, hepsi bitince devam et" pattern'i

### 11. Banking pattern'leri — gerçek senaryolar

#### Pattern 1: Fan-out validation

Transfer öncesi paralel validation:
- Account exists check
- Daily limit check
- Fraud score check
- KYC status check

```java
public void preTransferChecks(UUID userId, UUID accountId, Money amount) 
        throws InterruptedException, ExecutionException {
    
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<Boolean> accountExists = scope.fork(() -> accountService.exists(accountId));
        Subtask<DailyLimit> limit = scope.fork(() -> limitService.getDailyLimit(userId));
        Subtask<FraudScore> fraud = scope.fork(() -> fraudService.score(userId, amount));
        Subtask<KycStatus> kyc = scope.fork(() -> kycService.status(userId));
        
        scope.join();
        scope.throwIfFailed();
        
        if (!accountExists.get()) throw new AccountNotFoundException(accountId);
        if (limit.get().exceededBy(amount)) throw new DailyLimitExceededException();
        if (fraud.get().isRisky()) throw new HighRiskException();
        if (!kyc.get().isApproved()) throw new KycNotApprovedException();
    }
}
```

#### Pattern 2: Multi-source data aggregation

Customer dashboard — paralel veri çekme:

```java
public CustomerDashboard buildDashboard(UUID customerId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<List<Account>> accountsTask = scope.fork(() -> accountService.listByOwner(customerId));
        Subtask<List<Card>> cardsTask = scope.fork(() -> cardService.listByOwner(customerId));
        Subtask<List<Loan>> loansTask = scope.fork(() -> loanService.listByCustomer(customerId));
        Subtask<CreditScore> scoreTask = scope.fork(() -> kkbBureau.getScore(customerId));
        
        scope.join();
        scope.throwIfFailed();
        
        return new CustomerDashboard(
            accountsTask.get(), 
            cardsTask.get(), 
            loansTask.get(), 
            scoreTask.get()
        );
    }
}
```

Sequential olsa: account (200ms) + cards (300ms) + loans (250ms) + score (500ms) = 1250ms.
Parallel: max(200, 300, 250, 500) = 500ms.

#### Pattern 3: Hedged request (race for first response)

External service'in yedeği varsa, ikisine birden sor, hızlı dönen kazansın.

```java
public FxRate getFxRate(Currency from, Currency to) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<FxRate>()) {
        scope.fork(() -> primaryProvider.getRate(from, to));
        scope.fork(() -> secondaryProvider.getRate(from, to));
        
        scope.join();
        return scope.result();   // hangisi önce dönerse
    }
}
```

### 12. Anti-pattern'ler

**Anti-pattern 1: Scope dışına subtask sızdırmak**

```java
// ❌ KÖTÜ
public Future<Account> badPattern(UUID id) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<Account> task = scope.fork(() -> repo.findById(id));
        // scope.join() çağırmadan return
        return CompletableFuture.completedFuture(task.get());   // hata!
    }
}
```

Scope kapatılır, subtask hâlâ devam ediyor olabilir, `task.get()` "join'lemediğin için" `IllegalStateException` fırlatır.

**Kural:** Scope içindeki subtask'lar **scope içinde tamamlanmalı**. Sonuç değer (Account, Money) dön, Future/Task değil.

**Anti-pattern 2: `join()` öncesi `get()`**

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<Account> task = scope.fork(...);
    Account a = task.get();   // ❌ henüz join'lenmedi
    scope.join();
}
```

`join()` sonrası `get()` çağrılmalı.

**Anti-pattern 3: Try-with-resources unutmak**

```java
var scope = new StructuredTaskScope.ShutdownOnFailure();   // ❌ try-with-resources yok
scope.fork(...);
scope.join();
// scope.close() çağrılmazsa thread leak
```

**Anti-pattern 4: Nested scope mantığı bozmak**

```java
try (var outer = new StructuredTaskScope.ShutdownOnFailure()) {
    outer.fork(() -> {
        try (var inner = new StructuredTaskScope.ShutdownOnSuccess<...>()) {
            // OK — nested doğru
        }
    });
}
```

Nested scope **doğru** yapı. Fakat eğer iç scope dışarıya **subtask sızdırırsa** (return etmek üzere) yapı bozulur.

**Anti-pattern 5: Long-running daemon task**

Structured concurrency **scope-bounded**. "Always-on background task" için yanlış araç. ExecutorService veya `@Scheduled` kullan.

---

## Önemli olabilecek araştırma kaynakları

- JEP 453: Structured Concurrency (Java 21 preview)
- JEP 480: Structured Concurrency (Java 23 preview, revised)
- "Structured Concurrency in Java" — Inside Java talks
- Project Loom proposal — Structured Concurrency section
- Ron Pressler — JVM Language Summit talks
- `--enable-preview` flag dokümantasyonu
- Trio (Python) — structured concurrency'nin orijinal mucidi, fikir babası

---

## Mini task'ler

### Task 3.8.1 — Fan-out validation (45 dk)

`core-banking`'te transfer öncesi 3-4 paralel validation:

```java
@Service
public class TransferValidator {
    
    public ValidationResult validate(UUID userId, UUID fromId, UUID toId, Money amount) 
            throws InterruptedException, ExecutionException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Subtask<Boolean> fromExists = scope.fork(() -> accountService.exists(fromId));
            Subtask<Boolean> toExists = scope.fork(() -> accountService.exists(toId));
            Subtask<DailyLimit> limit = scope.fork(() -> limitService.getDailyLimit(userId));
            Subtask<FraudScore> fraud = scope.fork(() -> fraudService.score(fromId, amount));
            
            scope.join();
            scope.throwIfFailed();
            
            return new ValidationResult(
                fromExists.get(), toExists.get(), 
                limit.get(), fraud.get()
            );
        }
    }
}
```

`--enable-preview` flag ile compile et:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <release>21</release>
        <compilerArgs><arg>--enable-preview</arg></compilerArgs>
    </configuration>
</plugin>
```

### Task 3.8.2 — Hedged FX rate provider (45 dk)

Topic 3.7'deki FX provider kodunu **structured concurrency** ile yeniden yaz:

```java
public FxRate getFxRate(Currency from, Currency to) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<FxRate>()) {
        providers.forEach(p -> scope.fork(() -> p.getRate(from, to)));
        scope.join();
        return scope.result();
    }
}
```

3 farklı provider mock'la, her birine farklı latency ver (50ms, 200ms, 1s). En hızlının kazandığını test'le doğrula.

### Task 3.8.3 — Customer dashboard aggregator (45 dk)

`CustomerDashboardService.build(customerId)` — accounts, cards, loans, credit score paralel çek.

Sequential vs structured concurrency süresini ölç. **Defterine yaz.**

### Task 3.8.4 — Timeout handling (30 dk)

```java
public KycResult kycWithTimeout(UUID customerId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<KycResult>()) {
        scope.fork(() -> kkbBureau.check(customerId));
        scope.fork(() -> findeksBureau.check(customerId));
        
        if (!scope.joinUntil(Instant.now().plusSeconds(3))) {
            throw new KycTimeoutException("3 saniye içinde sonuç gelmedi");
        }
        return scope.result();
    }
}
```

Bir provider 5 sn'de cevap versin (timeout aşar), diğeri 1 sn'de versin → 1 sn'de result al, diğer iptal.

### Task 3.8.5 — ScopedValue + scope kombinasyonu (30 dk)

User context'i scope içinde **otomatik** paylaş:

```java
private static final ScopedValue<UUID> CURRENT_USER = ScopedValue.newInstance();

public CustomerDashboard buildDashboard(UUID userId) throws Exception {
    return ScopedValue.where(CURRENT_USER, userId).call(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            scope.fork(() -> {
                UUID u = CURRENT_USER.get();   // parent scope'tan alır
                return accountService.findByUser(u);
            });
            // ...
            scope.join();
            scope.throwIfFailed();
            return new CustomerDashboard(...);
        }
    });
}
```

---

## Test yazma rehberi

### Test 3.8.1 — ShutdownOnFailure cancellation

```java
@Test
void shutdownOnFailureShouldCancelOtherTasks() throws Exception {
    AtomicBoolean task2Completed = new AtomicBoolean(false);
    AtomicBoolean task2Interrupted = new AtomicBoolean(false);
    
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        scope.fork(() -> { throw new RuntimeException("fast failure"); });
        scope.fork(() -> {
            try {
                Thread.sleep(2000);
                task2Completed.set(true);
            } catch (InterruptedException e) {
                task2Interrupted.set(true);
            }
            return null;
        });
        
        scope.join();
        try {
            scope.throwIfFailed();
        } catch (ExecutionException expected) {
            // bekleniyor
        }
    }
    
    assertThat(task2Completed).isFalse();
    assertThat(task2Interrupted).isTrue();   // diğer task interrupt edilmiş
}
```

### Test 3.8.2 — ShutdownOnSuccess race

```java
@Test
void shutdownOnSuccessShouldReturnFirstResult() throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
        scope.fork(() -> { Thread.sleep(100); return "slow"; });
        scope.fork(() -> { Thread.sleep(50); return "fast"; });
        scope.fork(() -> { Thread.sleep(200); return "slowest"; });
        
        scope.join();
        String result = scope.result();
        
        assertThat(result).isEqualTo("fast");
    }
}

@Test
void shutdownOnSuccessShouldThrowIfAllFail() throws InterruptedException {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
        scope.fork(() -> { throw new RuntimeException("err1"); });
        scope.fork(() -> { throw new RuntimeException("err2"); });
        
        scope.join();
        
        assertThatThrownBy(scope::result).isInstanceOf(ExecutionException.class);
    }
}
```

### Test 3.8.3 — joinUntil timeout

```java
@Test
void joinUntilShouldReturnFalseOnTimeout() throws InterruptedException {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        scope.fork(() -> { Thread.sleep(5000); return "slow"; });
        
        boolean finished = scope.joinUntil(Instant.now().plusMillis(100));
        
        assertThat(finished).isFalse();
    }
}
```

### Test 3.8.4 — Subtask state

```java
@Test
void subtaskStateShouldReflectExecutionResult() throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<String> successful = scope.fork(() -> "ok");
        Subtask<String> failed = scope.fork(() -> { throw new RuntimeException("fail"); });
        
        scope.join();
        
        // success task hâlâ çalışmış olabilir
        assertThat(successful.state()).isIn(Subtask.State.SUCCESS, Subtask.State.UNAVAILABLE);
        assertThat(failed.state()).isEqualTo(Subtask.State.FAILED);
        assertThat(failed.exception()).isInstanceOf(RuntimeException.class);
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki Java structured concurrency kodumu banking-grade kriterlere göre 
değerlendir. Sadece eksik veya yanlışları işaretle:

1. Scope kullanımı:
   - `try-with-resources` ile scope açılmış mı?
   - `--enable-preview` flag eklenmiş mi (Java 21'de)?
   - Subtask'lar scope dışına sızdırılmış mı (Future return)? (Olmamalı)

2. ShutdownOnFailure pattern'i:
   - "Hepsi başarılı olmalı" senaryosunda doğru seçilmiş mi?
   - `scope.join()` öncesi `subtask.get()` çağrılmış mı? (Yanlış sıralama)
   - `scope.throwIfFailed()` çağrılmış mı?

3. ShutdownOnSuccess pattern'i:
   - "İlk başarılı yeter" senaryosunda doğru seçilmiş mi?
   - `scope.result()` ile sonuç alınmış mı?
   - Tüm fail senaryosunda exception handling var mı?

4. Banking pattern'leri:
   - Fan-out validation (paralel pre-checks) yapılmış mı?
   - Multi-source aggregation (dashboard) için kullanılmış mı?
   - Hedged request (FX, KYC) için ShutdownOnSuccess var mı?

5. Timeout:
   - `joinUntil(Instant)` ile timeout handling var mı?
   - Timeout sonrası proper exception fırlatılmış mı?

6. ScopedValue ile kombinasyon:
   - Parent scope'tan child task'lara context aktarımı ScopedValue ile mi?
   - ThreadLocal hâlâ kullanılıyor mu? (Modern code'da ScopedValue tercih)

7. Anti-pattern:
   - Subtask scope dışına sızdırılmış mı (CompletableFuture return)?
   - Try-with-resources unutulmuş mu?
   - `scope.join()` öncesi `get()` çağrılmış mı?
   - Long-running daemon task structured scope'a sokulmuş mu? (Olmamalı)

8. Exception handling:
   - `throwIfFailed()` `ExecutionException`'ı unwrap edilmiş mi?
   - Domain-specific exception'lar handle edilmiş mi?

9. CompletableFuture vs Structured Concurrency:
   - Doğru senaryo için doğru araç seçilmiş mi?
   - Complex async chain (compose, map) için CompletableFuture mu?
   - Scope-bounded batch operation için Structured mu?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] `ShutdownOnFailure` ile fan-out validation yazdım
- [ ] `ShutdownOnSuccess` ile hedged FX provider yazdım
- [ ] Customer dashboard paralel aggregation çalışıyor
- [ ] `joinUntil` ile timeout handling implement ettim
- [ ] ScopedValue ile parent → child context aktarımı yaptım
- [ ] Try-with-resources scope kullanımına alıştım
- [ ] `--enable-preview` flag projemde aktif
- [ ] Cancellation davranışını canlı gözlemledim (failure → interrupt diğer task'lar)
- [ ] Anti-pattern'lerden (subtask scope dışına sızdırma) kaçınıyorum
- [ ] CompletableFuture vs Structured Concurrency arası karar verebiliyorum

---

## Defter notları

1. "Unstructured concurrency'nin 5 problemi: ____."
2. "`ShutdownOnFailure` ne zaman, `ShutdownOnSuccess` ne zaman: ____."
3. "`scope.join()` ile `scope.throwIfFailed()` farkı: ____."
4. "Subtask'ı scope dışına sızdırmamamın sebebi: ____."
5. "Hedged request pattern (banking örneği): ____."
6. "`joinUntil(Instant)` ile timeout nasıl handle edilir: ____."
7. "ScopedValue + StructuredTaskScope kombinasyonu neden iyi: ____."
8. "Long-running daemon task neden structured scope'a girmez: ____."
9. "CompletableFuture vs Structured Concurrency karar matrisi: ____."
10. "Custom Joiner (Java 24) ile ShutdownOnFailure/Success arasındaki fark: ____."
