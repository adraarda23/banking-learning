# Topic 3.5 — CompletableFuture & Async Composition

## Hedef

`CompletableFuture` API'sini banking-grade kullanabilmek. `supplyAsync` vs `runAsync`, `thenApply` vs `thenCompose` vs `thenCombine` farkları, `allOf` vs `anyOf` koşullarını net ayırmak, exception handling (`exceptionally`, `handle`, `whenComplete`), timeout (`orTimeout`, `completeOnTimeout`), **custom executor pass etmenin neden zorunlu olduğunu** (common pool tuzağı), `CompletionException` unwrap pattern'i. Banking örneği: **3 FX rate provider'dan paralel fetch + anyOf first-wins + fallback**.

## Süre

Okuma: 2.5 saat • Mini task'ler: 3 saat • Test: 1 saat • Toplam: ~6.5 saat

## Önbilgi

- Topic 3.1-3.4 tamamlandı (JMM, sync, locks, executor)
- Lambda ve method reference syntax tanıdık
- `Future` interface'i (`get`, `cancel`) tanıdık
- `ExecutorService` ile basit task submit yapabiliyorsun

---

## Kavramlar

### 1. `Future` neden yetmiyor?

```java
Future<Money> future = executor.submit(() -> fxService.fetchRate("USD/TRY"));
Money rate = future.get();   // ❌ BLOCKING
```

`Future` API'sinin sınırları:

- `get()` blocking — caller thread bekler
- Chain edemezsin — "result geldiğinde X yap" diye non-blocking pattern yok
- Exception handling primitive — try-catch `get()` etrafında
- Multiple future combine zor — manuel logic
- Cancel semantik kısıtlı

**`CompletableFuture` (Java 8+)** bunu çözer:
- `CompletionStage` interface'i implementer, chain'lenebilir
- `thenApply`, `thenAccept`, `thenRun` — sonuca callback
- `thenCompose` — async chain
- `allOf` / `anyOf` — combinator'lar
- `exceptionally` / `handle` — error transform
- `orTimeout` — declarative timeout

---

### 2. `CompletableFuture` yaratma yolları

#### Hazır değer

```java
var ready = CompletableFuture.completedFuture(Money.of("100", "TRY"));
// ya da
var failed = CompletableFuture.failedFuture(new RuntimeException("boom"));
```

Test/mock için kullanışlı.

#### Manuel complete

```java
var future = new CompletableFuture<Money>();
// başka yerde:
future.complete(Money.of("100", "TRY"));
// veya
future.completeExceptionally(new RuntimeException("fail"));
```

Custom async source'lardan (örn. callback-based API'leri future'a çevirme) kullanılır.

#### `supplyAsync` — değer üretir

```java
CompletableFuture<Money> future = CompletableFuture.supplyAsync(() -> 
    fxService.fetchRate("USD/TRY")
);
```

Default `ForkJoinPool.commonPool()` üzerinde çalışır. **Banking'de yasak** (sonraki bölüm).

#### `runAsync` — değer dönmez (`Runnable`)

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> 
    auditService.log("Transfer completed")
);
```

Fire-and-forget benzeri ama future'ı await edebilirsin.

---

### 3. Custom executor pass etme — banking zorunluluğu

```java
CompletableFuture.supplyAsync(() -> fetchRate(), myExecutor);
```

**Neden zorunlu:**

- Default common pool boyutu: `processors - 1` (8 core'da 7 thread)
- **Tüm parallel stream + tüm CompletableFuture** common pool'u paylaşır
- IO bound iş (FX fetch HTTP çağrısı) common pool thread'ini **bloke eder**
- Diğer parallel iş yavaşlar veya durur
- Cascading slowdown / outage

**Banking pratiği — domain başına ayrı pool, her async çağrıda pass:**

```java
@Component
public class FxService {
    private final ExecutorService fxExecutor;  // Topic 3.4'te yarattık
    
    public CompletableFuture<Money> fetchRateAsync(CurrencyPair pair) {
        return CompletableFuture.supplyAsync(() -> fetchRate(pair), fxExecutor);
    }
}
```

**Hangi async metotlar default executor kullanır?**

`thenApplyAsync`, `thenAcceptAsync`, `thenRunAsync`, `thenComposeAsync`, `thenCombineAsync`, ... — yani **`Async` ekli her metot**.

Hepsine executor pass etmen zorunlu:

```java
future.thenApplyAsync(rate -> convertAmount(rate), myExecutor);
```

`Async` ekli olmayan `thenApply`, `thenAccept` — **çağıran thread'de** veya **önceki future'ın bittiği thread'de** çalışır. Hızlı transformasyonlar için kabul edilebilir, ama davranışı bilmek lazım.

---

### 4. `thenApply` vs `thenCompose` vs `thenCombine`

#### `thenApply(Function<T, U>)` — sync transform

```java
CompletableFuture<Money> rateFuture = fxService.fetchRateAsync("USD/TRY");

CompletableFuture<Money> convertedAmount = rateFuture.thenApply(rate -> {
    return Money.of(rate.amount().multiply(new BigDecimal("100")), rate.currency());
});
```

`Function<T, U>` — `T` alır, `U` döner. **Sync** transformasyon (lambda blocking olmamalı).

`Optional.map` benzeri zihinsel model.

#### `thenCompose(Function<T, CompletableFuture<U>>)` — async chain

```java
CompletableFuture<Account> accountFuture = accountService.findAsync(id);

CompletableFuture<Money> balanceConverted = accountFuture.thenCompose(account -> {
    return fxService.fetchRateAsync(account.currency())
        .thenApply(rate -> rate.multiply(account.balance().amount(), HALF_EVEN));
});
```

`Function<T, CompletableFuture<U>>` — `T` alır, **future** döner. Pratik: "ilk async iş bitti, sonucuna göre **başka** bir async iş başlat".

`Optional.flatMap` benzeri.

**`thenApply` vs `thenCompose` farkı:**

- `thenApply(rate -> heavyComputation)` → `CompletableFuture<Money>` — sync iş
- `thenApply(rate -> callOtherService())` → `CompletableFuture<CompletableFuture<Money>>` — **nested**, kötü
- `thenCompose(rate -> callOtherService())` → `CompletableFuture<Money>` — düzleştirilmiş

Eğer lambda CompletableFuture döndürüyorsa, `thenCompose` kullan. Aksi halde `thenApply`.

#### `thenCombine(other, BiFunction<T, U, R>)` — iki future birleştir

```java
CompletableFuture<Money> usdRate = fxService.fetchRateAsync("USD/TRY");
CompletableFuture<Money> eurRate = fxService.fetchRateAsync("EUR/TRY");

CompletableFuture<Money> spread = usdRate.thenCombine(eurRate, (usd, eur) -> {
    return usd.subtract(eur);
});
```

İki future paralel çalışır, **her ikisi bittiğinde** combiner çalışır.

3+ future: `allOf` daha temiz (sonraki madde).

---

### 5. `thenAccept` ve `thenRun` — yan etki

```java
future.thenAccept(rate -> log.info("Got rate: {}", rate));    // T → void

future.thenRun(() -> log.info("Future completed"));            // no input → void
```

Side-effect (log, metric, notify) için. **Return value yok**. Chain'in sonu.

---

### 6. `allOf` — hepsi bittiğinde

```java
var f1 = fxService.fetchRateAsync("USD/TRY");
var f2 = fxService.fetchRateAsync("EUR/TRY");
var f3 = fxService.fetchRateAsync("GBP/TRY");

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);

CompletableFuture<List<Money>> rates = all.thenApply(v -> List.of(
    f1.join(), f2.join(), f3.join()
));
```

**`allOf(...)`** signature `CompletableFuture<Void>` döner — sonuçları **kendin toplaman** gerekir (`f1.join()`, `f2.join()`...).

**`join()` vs `get()`:** Aynı şey ama `join` checked exception fırlatmaz (`CompletionException` wraps). Chain içinde `join` tercih.

**Exception handling — hepsi bittiğinde:**

```java
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.exceptionally(ex -> {
    log.error("At least one rate failed", ex);
    return null;
});
```

**Dikkat:** `allOf` herhangi biri fail ederse **`exceptionally`** çağrılır ama **diğerleri çalışmaya devam eder**. Cancel istiyorsan manuel cancelation gerekir.

**Banking örneği — bağımsız async iş paraleli:**

```java
public CompletableFuture<TransferReport> generateReport(TransferId id) {
    var transferF = transferService.fetchAsync(id);
    var senderF = customerService.fetchSenderAsync(id);
    var receiverF = customerService.fetchReceiverAsync(id);
    var auditF = auditService.fetchTrailAsync(id);
    
    return CompletableFuture.allOf(transferF, senderF, receiverF, auditF)
        .thenApply(v -> new TransferReport(
            transferF.join(),
            senderF.join(),
            receiverF.join(),
            auditF.join()
        ));
}
```

4 paralel sorgu, hepsi bittiğinde rapor oluştur. Sekansiyel (4 × 100ms = 400ms) vs paralel (~100ms).

---

### 7. `anyOf` — ilk bittiğinde

```java
var ecbF = ecbProvider.fetchRateAsync("USD/TRY");
var tcmbF = tcmbProvider.fetchRateAsync("USD/TRY");
var reutersF = reutersProvider.fetchRateAsync("USD/TRY");

CompletableFuture<Object> first = CompletableFuture.anyOf(ecbF, tcmbF, reutersF);
CompletableFuture<Money> rate = first.thenApply(r -> (Money) r);
```

**`anyOf(...)` `CompletableFuture<Object>`** döner — type-safe değil, cast gerek. **İlk** completes eden (success **veya** exception) future'ın sonucu gelir.

**Banking örneği — first-wins FX fetch:**

```java
public CompletableFuture<Money> fetchRateFastest(CurrencyPair pair) {
    var providers = List.of(ecbProvider, tcmbProvider, reutersProvider);
    
    var futures = providers.stream()
        .map(p -> p.fetchRateAsync(pair).exceptionally(ex -> null))
        .toList();
    
    return CompletableFuture.anyOf(futures.toArray(CompletableFuture[]::new))
        .thenApply(r -> (Money) r);
}
```

3 provider'dan paralel iste, en hızlı cevap veren kullanılır. Diğerleri **devam eder** (background'da bitirir, sonuçları discard).

**Dikkat:**
- `anyOf` exception'ı da "completed" sayar. İlk **başarısız** olan kazanırsa, downstream exception alır.
- Bunu önlemek için her future'ın `exceptionally(ex -> null)` ile fallback'i lazım, sonra null filter.

**Daha iyi `firstSuccessful` pattern:**

```java
public <T> CompletableFuture<T> firstSuccessful(List<CompletableFuture<T>> futures) {
    var result = new CompletableFuture<T>();
    var remaining = new AtomicInteger(futures.size());
    
    for (var f : futures) {
        f.whenComplete((value, ex) -> {
            if (ex == null) {
                result.complete(value);  // ilk success kazansın
            } else if (remaining.decrementAndGet() == 0) {
                result.completeExceptionally(new RuntimeException("All failed"));
            }
        });
    }
    
    return result;
}
```

---

### 8. Exception handling — `exceptionally`, `handle`, `whenComplete`

#### `exceptionally(Function<Throwable, T>)` — error → fallback value

```java
CompletableFuture<Money> rate = fxService.fetchRateAsync("USD/TRY")
    .exceptionally(ex -> {
        log.warn("FX fetch failed, using cached", ex);
        return cachedFxService.lastKnownRate("USD/TRY");
    });
```

Exception varsa lambda çağrılır, dönen değer success result olur. Exception **swallowed**, downstream success görür.

#### `handle(BiFunction<T, Throwable, R>)` — her durumu işle

```java
CompletableFuture<Result> result = rate.handle((value, ex) -> {
    if (ex != null) {
        metrics.increment("fx.fetch.failure");
        return Result.failure(ex.getMessage());
    }
    metrics.increment("fx.fetch.success");
    return Result.success(value);
});
```

Hem success hem exception path'ini aynı yerde yönet. Result wrapper pattern.

#### `whenComplete(BiConsumer<T, Throwable>)` — yan etki, sonucu değiştirme

```java
rate.whenComplete((value, ex) -> {
    if (ex != null) {
        log.error("FX fetch failed", ex);
    } else {
        log.info("FX fetched: {}", value);
    }
});
```

Sadece **gözlem** — sonuç downstream'e olduğu gibi geçer. Exception varsa hâlâ exception yayılır.

#### `exceptionallyCompose` — async fallback

```java
rate.exceptionallyCompose(ex -> {
    log.warn("Primary failed, calling backup", ex);
    return backupFxService.fetchRateAsync("USD/TRY");
});
```

Fallback'in kendisi async ise. (Java 12+)

---

### 9. `CompletionException` — wrapping problemi

```java
try {
    Money rate = future.get();
} catch (ExecutionException e) {
    Throwable cause = e.getCause();   // gerçek exception
    // ...
}
```

`Future.get()` her exception'ı **`ExecutionException`** içine wrap eder.

`CompletableFuture.join()` ise **`CompletionException`** içine wrap eder (unchecked).

`thenApply` lambda içinde fırlatılan exception, `exceptionally` lambda'sına **wrapped** gelir:

```java
future.exceptionally(ex -> {
    Throwable cause = ex.getCause();   // ← unwrap
    if (cause instanceof InsufficientFundsException ife) {
        return Money.zero(TRY);
    }
    throw new RuntimeException(cause);
});
```

**Banking pratiği — utility wrapper:**

```java
public static Throwable unwrap(Throwable t) {
    while (t instanceof CompletionException || t instanceof ExecutionException) {
        t = t.getCause();
    }
    return t;
}

// Kullanım:
future.exceptionally(ex -> {
    var cause = unwrap(ex);
    if (cause instanceof InsufficientFundsException) {
        return fallback;
    }
    throw new RuntimeException(cause);
});
```

---

### 10. Timeout — `orTimeout`, `completeOnTimeout`

Java 9'da geldi (öncesinde manual `ScheduledExecutorService` ile yapılırdı).

#### `orTimeout(long, TimeUnit)` — exception ile timeout

```java
CompletableFuture<Money> rate = fxService.fetchRateAsync("USD/TRY")
    .orTimeout(2, TimeUnit.SECONDS);
```

2 saniyede tamamlanmazsa **`TimeoutException`** ile complete olur.

#### `completeOnTimeout(T value, long, TimeUnit)` — fallback değer ile

```java
CompletableFuture<Money> rate = fxService.fetchRateAsync("USD/TRY")
    .completeOnTimeout(Money.of("33.50", "TRY"), 2, TimeUnit.SECONDS);
```

2 saniyede gelmezse `Money.of("33.50", "TRY")` kullanır. Exception fırlatmaz.

**Banking pratiği — timeout + fallback chain:**

```java
public CompletableFuture<Money> fetchRateSafe(CurrencyPair pair) {
    return primaryFxProvider.fetchRateAsync(pair)
        .orTimeout(500, TimeUnit.MILLISECONDS)
        .exceptionallyCompose(ex -> {
            log.warn("Primary FX failed/timeout: {}", ex.getMessage());
            return backupFxProvider.fetchRateAsync(pair)
                .orTimeout(1, TimeUnit.SECONDS)
                .exceptionally(ex2 -> cachedFxService.lastKnownRate(pair));
        });
}
```

Primary 500ms timeout, fail/timeout → backup 1s timeout → fail → cached. **Defense in depth**.

---

### 11. Banking örneği — parallel FX fetch (3 provider anyOf)

```java
@Component
public class ParallelFxFetcher {
    private final List<FxProvider> providers;
    private final ExecutorService fxExecutor;
    
    public ParallelFxFetcher(List<FxProvider> providers, 
                              @Qualifier("fxExecutor") ExecutorService fxExecutor) {
        this.providers = providers;
        this.fxExecutor = fxExecutor;
    }
    
    public CompletableFuture<Money> fetchFastest(CurrencyPair pair) {
        if (providers.isEmpty()) {
            return CompletableFuture.failedFuture(
                new IllegalStateException("No FX providers configured"));
        }
        
        var result = new CompletableFuture<Money>();
        var failuresRemaining = new AtomicInteger(providers.size());
        
        for (var provider : providers) {
            CompletableFuture.supplyAsync(() -> provider.fetchRate(pair), fxExecutor)
                .orTimeout(2, TimeUnit.SECONDS)
                .whenComplete((rate, ex) -> {
                    if (ex == null && rate != null) {
                        result.complete(rate);     // ilk success kazandı
                    } else if (failuresRemaining.decrementAndGet() == 0) {
                        result.completeExceptionally(
                            new FxFetchException("All providers failed")
                        );
                    }
                });
        }
        
        return result;
    }
}
```

**Davranış:**

- 3 provider paralel istenir (her biri kendi timeout'u ile)
- İlk **başarılı** olan kazanır, downstream onunla çalışır
- Hepsi fail ise `FxFetchException`
- Geç gelen başarılı sonuçlar **discard** edilir
- Toplam latency = en hızlı provider'ın latency'si (genellikle 200-500ms)

**Sequential alternative (her provider sırayla):** En kötü durumda 3 × 2 = 6 saniye. Paralel 2 saniye max.

---

### 12. CompletableFuture pipelining örneği — banking transfer

```java
public CompletableFuture<TransferResult> executeTransfer(TransferRequest req) {
    return accountService.findAsync(req.fromAccountId())
        .thenCombineAsync(
            accountService.findAsync(req.toAccountId()),
            (from, to) -> validate(from, to, req.amount()),
            validationExecutor
        )
        .thenComposeAsync(validated -> 
            fraudService.checkAsync(validated)
                .orTimeout(500, TimeUnit.MILLISECONDS)
                .exceptionally(ex -> FraudCheckResult.skipped("timeout")),
            fraudExecutor
        )
        .thenComposeAsync(fraud -> {
            if (fraud.isBlocked()) {
                return CompletableFuture.failedFuture(new FraudBlockedException());
            }
            return transferService.executeAsync(req);
        }, transferExecutor)
        .thenComposeAsync(result -> 
            CompletableFuture.allOf(
                auditService.recordAsync(result),
                notificationService.notifyAsync(result)
            ).thenApply(v -> result),
            auditExecutor
        )
        .orTimeout(10, TimeUnit.SECONDS)
        .exceptionally(ex -> TransferResult.failure(unwrap(ex).getMessage()));
}
```

**Akış:**

1. Source ve target account paralel fetch
2. Validation (sync, fast)
3. Fraud check (async, 500ms timeout, fail → skipped)
4. Fraud OK ise transfer execute
5. Audit + notification paralel
6. Toplam 10s timeout
7. Exception → TransferResult.failure

Her async iş **kendi executor**'unda. Pipeline okunabilir, fail-soft yerleri belli.

---

### 13. Anti-pattern'ler

**Anti-pattern 1: Default common pool kullanma**

```java
CompletableFuture.supplyAsync(() -> fetchRate());   // ❌ common pool
```

Banking'de **her async**'e custom executor pass et.

**Anti-pattern 2: `thenApply` içinde `CompletableFuture` dönmek**

```java
future.thenApply(rate -> anotherService.fetchAsync(rate));
// → CompletableFuture<CompletableFuture<X>> — nested mess
```

Çözüm: `thenCompose`.

**Anti-pattern 3: `join()` chain ortasında**

```java
future
    .thenApply(rate -> {
        var other = otherFuture.join();   // ❌ blocking, pool thread'i tutar
        return combine(rate, other);
    });
```

Çözüm: `thenCombine` veya `allOf`.

**Anti-pattern 4: Exception swallow + log only**

```java
future.exceptionally(ex -> null);   // ❌ caller "success" gördü, hata kayboldu
```

Çözüm: Result wrapper veya rethrow.

**Anti-pattern 5: `get()` timeout'suz**

```java
future.get();   // ❌ sonsuza kadar
```

`get(timeout, unit)` veya `orTimeout` üst seviyede.

**Anti-pattern 6: `CompletableFuture` chain'de side-effect üst üste**

```java
future
    .thenAccept(rate -> log.info("got rate"))
    .thenAccept(v -> metric.increment());   // ❌ v hep null (thenAccept void)
```

Çözüm: `whenComplete` ile birleştir veya `thenApply` ile değer geri al.

**Anti-pattern 7: Mutable state lambda içinde**

```java
List<Money> results = new ArrayList<>();
futures.forEach(f -> f.thenAccept(results::add));   // ❌ race
```

Çözüm: `allOf` + `join()` ile result al, ya da `ConcurrentLinkedQueue` kullan.

**Anti-pattern 8: Cancel çağırmayı düşünmek**

`future.cancel(true)` — `CompletableFuture` için `mayInterruptIfRunning` **etkisi yok** (Future API'sinin garantisi gevşek). Çalışan task'i durdurmaz, sadece dependent chain'i fail eder. Banking'de bunu bilen ol.

---

### 14. Reactive streams (kısa not)

`CompletableFuture` **tek değer** (one-shot). Akış (stream of values) için:

- **Project Reactor**: `Mono<T>` (0-1 değer) ve `Flux<T>` (0-N değer)
- **RxJava**: `Single`, `Observable`
- **Spring WebFlux**: Reactor üstüne kurulu

Banking projende reactive'e geçmek **büyük yatırım** — error semantics, debugging, ecosystem değişir. Çoğu TR bankası **CompletableFuture + virtual thread** ile yetiniyor (Java 21+). Reactive bir bonus.

Bu topic'te reactive'e girmiyoruz. `CompletableFuture` master'ı seviyene gelmek banking için yeterli.

---

## Önemli olabilecek araştırma kaynakları

- `CompletableFuture` JavaDoc (resmi)
- "Modern Java in Action" Manning, Chapter 16 (CompletableFuture)
- "Java Concurrency in Practice" — kısmen, eski (CF Java 8'de geldi)
- Heinz Kabutz — CompletableFuture pitfalls articles
- Tomasz Nurkiewicz — "Mastering CompletableFuture" talk
- Project Reactor docs (reference)
- Adam Bien — async patterns
- "Java Async Programming" Aleksey Shipilev articles
- `CompletionStage` interface JavaDoc

---

## Mini task'ler

### Task 3.5.1 — Basic supply + thenApply (30 dk)

`completable/SimpleAsync.java`:

- Custom executor (Topic 3.4'teki fxExecutor)
- `CompletableFuture.supplyAsync(() -> fetchRate(), executor)`
- `thenApply(rate -> rate.multiply(BigDecimal.TEN))`
- `thenAccept(result -> log)`
- Main'de `future.join()`

Test'inde mock service ile fixed rate dön, sonuç doğruluğunu assert et.

### Task 3.5.2 — `thenCompose` chain (30 dk)

`completable/ChainedAsync.java`:

- `accountService.findAsync(id)` → `Account`
- `thenCompose(account -> fxService.fetchRateAsync(account.currency()))` → `Money`
- `thenApply(rate -> convert(account.balance(), rate))` → `Money`
- Final: TRY cinsinden balance

Test: mock'larla bütün chain bitiyor mu?

### Task 3.5.3 — `thenCombine` iki paralel (30 dk)

`completable/ParallelCombine.java`:

- `usdRateFuture = fxService.fetchRateAsync("USD/TRY")`
- `eurRateFuture = fxService.fetchRateAsync("EUR/TRY")`
- `thenCombine` → `Money diff = usd.subtract(eur)`

Test'inde mock ile timing: hem usd hem eur 200ms işlem sürüyor. Toplam ~200ms (sequential 400ms'den hızlı) olduğunu logla.

### Task 3.5.4 — `allOf` ile multi-fetch (45 dk)

`completable/AllOfReport.java`:

- 4 async fetch (transfer, sender, receiver, audit)
- `allOf` ile combine
- `thenApply` ile `TransferReport` record yarat
- `join()` ile sonucu al

Test: 4 mock service paralel, total time max(individual).

### Task 3.5.5 — `anyOf` first-wins FX provider (45 dk)

`completable/AnyOfFx.java`:

- 3 mock FxProvider (`ecb`, `tcmb`, `reuters`)
- Her birinin latency'si random 100-500ms
- `anyOf` ile ilk başarılı al
- Hangi provider kazandığını log

Test: 100 kez koş, hangi provider en çok kazanır? (en hızlı latency'liye yakın olmalı)

### Task 3.5.6 — `firstSuccessful` (failure'ı atla) (45 dk)

`completable/FirstSuccessful.java`:

- 3 provider, biri kasıtlı exception fırlatıyor
- Naif `anyOf` denersen exception kazanabilir
- Custom `firstSuccessful` pattern (yukarıdaki kod) implement et
- Test: exception'lı provider varken yine de başarılı sonuç dönüyor

### Task 3.5.7 — Exception handling tüm yöntemler (45 dk)

`completable/ExceptionDemo.java`:

- `supplyAsync` lambda exception fırlatıyor
- 4 farklı handling:
  - `exceptionally(ex -> fallback)`
  - `handle((v, ex) -> ...)`
  - `whenComplete((v, ex) -> log)`
  - `try { future.get() } catch (ExecutionException)`
- `CompletionException` unwrap utility yaz, test et

### Task 3.5.8 — Timeout (`orTimeout`, `completeOnTimeout`) (30 dk)

`completable/TimeoutDemo.java`:

- 1s'de bitecek bir task
- `orTimeout(500ms)` → `TimeoutException`
- `completeOnTimeout(fallback, 500ms)` → fallback değer
- Karşılaştır, log

### Task 3.5.9 — Banking pipeline (60 dk)

`completable/TransferPipeline.java`:

Yukarıdaki `executeTransfer` pipeline'ını yaz:

1. Source + target account paralel
2. Validate (sync)
3. Fraud check (500ms timeout, soft fail)
4. Transfer execute
5. Audit + notify paralel
6. 10s overall timeout
7. Exception → TransferResult.failure

Mock'larla test, baştan sona log'la.

### Task 3.5.10 — Common pool blocking gözlem (30 dk)

`completable/CommonPoolBlocking.java`:

- Default executor ile 16 `supplyAsync` task (her biri 2s sleep)
- Aynı anda `Stream.parallel().mapToInt(...)`
- Common pool dolu → parallel stream **bloke**
- Custom executor ile aynı kodu yaz, izole çalış
- Defterine sonucu yaz

---

## Test yazma rehberi

### Test 3.5.1 — Basic chain

```java
@Test
void supplyAndApplyChain() throws Exception {
    var executor = Executors.newSingleThreadExecutor();
    var future = CompletableFuture
        .supplyAsync(() -> Money.of("100", "TRY"), executor)
        .thenApply(m -> m.add(Money.of("50", "TRY")));
    
    var result = future.get(2, TimeUnit.SECONDS);
    assertThat(result).isEqualTo(Money.of("150.00", "TRY"));
    
    executor.shutdown();
}
```

### Test 3.5.2 — allOf timing

```java
@Test
void allOfRunsInParallel() throws Exception {
    var executor = Executors.newFixedThreadPool(4);
    var start = System.nanoTime();
    
    var f1 = CompletableFuture.supplyAsync(() -> { 
        sleep(200); return "a"; 
    }, executor);
    var f2 = CompletableFuture.supplyAsync(() -> { 
        sleep(200); return "b"; 
    }, executor);
    var f3 = CompletableFuture.supplyAsync(() -> { 
        sleep(200); return "c"; 
    }, executor);
    
    CompletableFuture.allOf(f1, f2, f3).get(1, TimeUnit.SECONDS);
    long elapsedMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    
    // Sequential would be 600ms; parallel ~200ms
    assertThat(elapsedMs).isLessThan(500);
    
    executor.shutdown();
}

private void sleep(long ms) {
    try { Thread.sleep(ms); } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

### Test 3.5.3 — anyOf first wins

```java
@Test
void anyOfReturnsFirstCompletion() throws Exception {
    var executor = Executors.newFixedThreadPool(3);
    
    var slow = CompletableFuture.supplyAsync(() -> { sleep(500); return "slow"; }, executor);
    var fast = CompletableFuture.supplyAsync(() -> { sleep(100); return "fast"; }, executor);
    var mid = CompletableFuture.supplyAsync(() -> { sleep(300); return "mid"; }, executor);
    
    var winner = CompletableFuture.anyOf(slow, fast, mid).get(1, TimeUnit.SECONDS);
    assertThat(winner).isEqualTo("fast");
    
    executor.shutdown();
}
```

### Test 3.5.4 — exceptionally fallback

```java
@Test
void exceptionallyProvidesFallback() throws Exception {
    var future = CompletableFuture.<Money>supplyAsync(() -> {
        throw new RuntimeException("FX provider down");
    });
    
    var result = future
        .exceptionally(ex -> Money.of("33.50", "TRY"))
        .get(1, TimeUnit.SECONDS);
    
    assertThat(result).isEqualTo(Money.of("33.50", "TRY"));
}
```

### Test 3.5.5 — orTimeout

```java
@Test
void orTimeoutFiresTimeoutException() {
    var future = CompletableFuture.supplyAsync(() -> {
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
        return "late";
    }).orTimeout(100, TimeUnit.MILLISECONDS);
    
    assertThatThrownBy(() -> future.get(2, TimeUnit.SECONDS))
        .isInstanceOf(ExecutionException.class)
        .hasCauseInstanceOf(TimeoutException.class);
}
```

### Test 3.5.6 — firstSuccessful pattern

```java
@Test
void firstSuccessfulIgnoresFailures() throws Exception {
    var executor = Executors.newFixedThreadPool(3);
    
    var failing = CompletableFuture.<Money>supplyAsync(() -> {
        throw new RuntimeException("p1 down");
    }, executor);
    var slow = CompletableFuture.supplyAsync(() -> {
        sleep(300); return Money.of("33.50", "TRY");
    }, executor);
    var fast = CompletableFuture.supplyAsync(() -> {
        sleep(100); return Money.of("33.40", "TRY");
    }, executor);
    
    var fetcher = new ParallelFxFetcher();
    var result = fetcher.firstSuccessful(List.of(failing, slow, fast))
        .get(2, TimeUnit.SECONDS);
    
    assertThat(result).isEqualTo(Money.of("33.40", "TRY"));
    
    executor.shutdown();
}
```

---

## Claude-verify prompt

```
Aşağıdaki CompletableFuture kodum banking-grade async kullanım perspektifinde. 
Lütfen değerlendir ve EKSİKLERİ söyle, kod yazma:

1. Custom executor passing:
   - Her `supplyAsync`, `thenApplyAsync`, `thenComposeAsync`, vb. çağrısında 
     custom executor pass edilmiş mi?
   - Default common pool kullanılan yer var mı (yasak)?

2. thenApply vs thenCompose:
   - thenApply içinde CompletableFuture dönen lambda var mı (yasak)?
   - thenCompose async chain için kullanılmış mı?
   - thenCombine iki paralel future birleştirmek için kullanılmış mı?

3. allOf:
   - allOf sonrası f1.join() / f2.join() pattern doğru kullanılmış mı?
   - Exception path düşünülmüş mü (allOf'da bir tane fail ise behavior)?

4. anyOf:
   - anyOf'un exception'ı da "completed" saydığı bilinmiyor mu?
   - firstSuccessful pattern (failure'ı atla) implement edilmiş mi?
   - Banking örneği (3 FX provider) yazılmış mı?

5. Exception handling:
   - exceptionally, handle, whenComplete üçü de denenmiş mi?
   - CompletionException unwrap utility var mı?
   - Banking'de exception swallow + log only anti-pattern'i farkında mı?

6. Timeout:
   - orTimeout veya completeOnTimeout kullanılmış mı?
   - Fallback chain (primary timeout → backup → cached) var mı?

7. Banking pipeline:
   - Multi-step transfer pipeline (validate → fraud → execute → audit + notify) yazılmış mı?
   - Her async iş için domain-specific executor pass edilmiş mi?
   - Toplam timeout (10s) ile sarmalanmış mı?

8. Common pool tuzağı:
   - parallelStream + supplyAsync default ile bloke gösterilmiş mi?
   - Custom executor ile izolasyon yapılmış mı?

9. Anti-pattern kontrolü:
   - join() chain ortasında blocking call?
   - Mutable state lambda içinde?
   - Future.get() timeout'suz?
   - thenAccept zinciri yanlış kullanılmış mı?

10. Test coverage:
    - allOf timing testi (parallel < sequential)?
    - anyOf first-wins testi?
    - exceptionally fallback testi?
    - orTimeout testi?
    - CompletionException unwrap testi?

Her madde için PASS / FAIL / EKSIK. Kod yazma, sadece eksiklikleri söyle.
```

---

## Tamamlama kriterleri

- [ ] `supplyAsync`, `runAsync` farkını biliyorum
- [ ] **Her async** çağrıda custom executor pass ediyorum
- [ ] `thenApply` / `thenCompose` / `thenCombine` üçünün farkını örnekle açıklayabiliyorum
- [ ] `allOf` ile multi-fetch yazdım, sonuçları `join()` ile topladım
- [ ] `anyOf` first-wins pattern'ini yazdım
- [ ] `firstSuccessful` (failure'ı atla) custom pattern'i implement ettim
- [ ] `exceptionally` / `handle` / `whenComplete` üçünü senaryolarla denedim
- [ ] `CompletionException` unwrap utility'sini yazdım
- [ ] `orTimeout` ve `completeOnTimeout` ikisini de kullandım
- [ ] Banking transfer pipeline'ı yazdım (validate → fraud → execute → audit + notify)
- [ ] `parallelStream` + common pool blocking'i reproduce ettim, kendi executor ile izole ettim
- [ ] `join()` chain ortasında blocking call yapmıyorum

Hepsi onaylı → Topic 3.6'ya geç → [06-concurrent-collections/](../06-concurrent-collections/index.md)

---

## Defter notları

1. "`Future` API'sinin yetmediği 3 yer: ____, ____, ____."
2. "`thenApply` ve `thenCompose` farkı: ____. Hangi durumda hangisi: ____."
3. "`thenCombine` ile `allOf` arasındaki fark: ____."
4. "`anyOf`'un exception tehlikesi: ____. Çözüm: ____."
5. "Default common pool tuzağı: ____. Banking kuralı: ____."
6. "`exceptionally` ile `handle` farkı: ____."
7. "`whenComplete` neden yan etki için (sonucu değiştirmez): ____."
8. "`CompletionException` ve `ExecutionException` ne zaman çıkar, unwrap nasıl: ____."
9. "`orTimeout` ve `completeOnTimeout` farkı: ____."
10. "Banking transfer pipeline'da fail-soft yerleri: ____ ve ____ (fraud, audit). Fail-hard yerleri: ____ (transfer execute)."
