# Faz 3 — PHASE TEST

Faz 4'e geçmeden önce kendini sına.

## Pratik test

- [ ] `core-banking` mini-project'in 6 görevi de tamam
- [ ] JMH benchmark sonuçları defterimde tablo
- [ ] Canlı deadlock üretip jstack'le yakaladım
- [ ] MAT ile heap dump analizi yaptım
- [ ] JFR ile pinning event'i yakaladım
- [ ] Virtual thread vs platform thread fark gözlemlendi

## Konsept testi

### JMM & atomic
- [ ] Happens-before ilişkisini 3 cümleyle anlatabilirim
- [ ] `volatile`'in visibility sağladığını ama atomicity sağlamadığını biliyorum
- [ ] CAS ve ABA problemini açıklayabilirim
- [ ] `AtomicStampedReference` ile ABA çözümünü gördüm
- [ ] `LongAdder`'ın yüksek contention'da `AtomicLong`'tan iyi olmasının sebebini biliyorum

### Locks
- [ ] `synchronized` vs `ReentrantLock` 5 farkı sayabilirim
- [ ] `tryLock(timeout)` deadlock önleme için neden değerli
- [ ] `ReentrantReadWriteLock` ne zaman avantajlı
- [ ] `StampedLock` optimistic read pattern'ini anlatabilirim
- [ ] Deadlock 4 koşulunu sayabilirim

### Executor
- [ ] `ThreadPoolExecutor` 4 ana parametresini biliyorum
- [ ] RejectedExecutionHandler 4 stratejisini ve banking için seçimi
- [ ] ForkJoinPool common pool tuzağını biliyorum

### CompletableFuture
- [ ] `thenApply` vs `thenCompose` vs `thenCombine` farkı
- [ ] `allOf` vs `anyOf` ne zaman hangisini
- [ ] Custom executor passing önemi (default common pool tuzağı)

### Concurrent collections
- [ ] `ConcurrentHashMap` `merge`, `compute`, `computeIfAbsent` atomic
- [ ] `BlockingQueue` ailesinin her birinin en uygun senaryosu
- [ ] `CountDownLatch` (one-shot) vs `CyclicBarrier` (reusable) farkı
- [ ] Bounded vs unbounded queue: production karar

### Virtual threads
- [ ] Platform thread vs virtual thread maliyet farkı
- [ ] Pinning ne, hangi 3 durumda olur
- [ ] JFR ile pinning nasıl yakalanır
- [ ] `synchronized` yerine `ReentrantLock` virtual thread'le neden iyi
- [ ] Spring Boot 3.2+ virtual thread enable

### Structured concurrency
- [ ] `ShutdownOnFailure` ve `ShutdownOnSuccess` farkı
- [ ] `joinUntil` ile timeout
- [ ] CompletableFuture ile karşılaştırma — banking için karar

### JVM internals
- [ ] Generational hypothesis ve young/old GC
- [ ] G1 vs ZGC vs Parallel — banking için karar matrisi
- [ ] Xms = Xmx pattern'i neden production'da
- [ ] HeapDumpOnOutOfMemoryError, GC log, JFR — production gereklilikleri

### Performance tools
- [ ] `jcmd Thread.print` ile deadlock detection
- [ ] JFR continuous recording production-safe
- [ ] MAT Leak Suspects / Dominator Tree / Path to GC Roots akışı
- [ ] async-profiler ile flame graph okuma

### JMH
- [ ] Naive `System.nanoTime` benchmark'ın 5 sorunu
- [ ] Constant folding, DCE'den kaçınma yolları
- [ ] `@Fork(2)+` minimum sebebi
- [ ] Banking benchmark sonuçlarım: BigDecimal vs long, virtual vs platform

## Banking domain anlama

- [ ] Concurrent transfer race condition'ı `@Version` veya pessimistic locking ile çözüyorum
- [ ] Rate limiter pattern'i biliyorum (token bucket, Semaphore)
- [ ] Deadlock A→B + B→A senaryosunu lock ordering ile çözebilirim
- [ ] Memory leak hunting workflow'unu uygulamış oldum
- [ ] Production'da bir sorun gelse hangi araca başvuracağımı biliyorum

## Soft skills

- [ ] Stack trace'i okuyabiliyorum, deadlock satırını tespit edebiliyorum
- [ ] Defter notlarım Phase 3 boyunca güncel
- [ ] Bir senior dev'le concurrency tartışırken kendi cümlemle anlatabiliyorum

---

Hepsine "evet" → **Faz 4'e geç → 04-sql-oracle/**

Cevap "hayır" varsa: ilgili topic'in kavramlar bölümünü tekrar oku, mini task'i yeniden dene.

---

## Bonus — mid-junior'a geçtiğinin işareti

Phase 3'ü bitirdiğinde, şu cümleleri rahatça söyleyebilirsen mid-level concurrency seviyesindesin:

- "Production'da deadlock olursa nasıl teşhis edip çözeceğimi biliyorum."
- "Memory leak şüphesinde heap dump alıp MAT'la analiz edebilirim."
- "Virtual thread'i hangi senaryoda kullanacağıma karar verebiliyorum."
- "JFR ile production'da performans sorununu canlı yakalayabilirim."
- "JMH benchmark yazıp confidence interval ile karar destekleyebilirim."

Bunlar mid-level Java developer'ın bilmesi gereken şeyler. Phase 3 sonrasında bunları **derinden** biliyorsun.

Phase 4 SQL/Oracle daha kuru ama TR bank tarafında **mutlaka** sorulan konular: index, PL/SQL, partitioning. Phase 3'ten sonra rahat geçeceksin.
