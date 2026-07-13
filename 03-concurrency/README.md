# Faz 3 — Concurrency & JVM

## Hedef

Banking backend'inin en kritik teknik fazı. Java concurrency primitives'ini, JMM (Java Memory Model)'i, lock'ları, executor framework'ü, virtual thread'leri ve JVM internals'i banking-grade seviyede öğrenmek.

Bu faz **TR bank mülakatlarında en çok belirleyici olan** kısım — "thread-safe yap", "deadlock üret ve çöz", "GC tune et", "memory leak bul" tipi sorular burada.

Sonunda elinde: race condition'a karşı dirençli, virtual thread'le ölçeklenebilen, deadlock-free, profiling araçlarıyla optimize edilmiş bir banking servisi.

## Süre

6 hafta (günde 2-3 saat). Phase 3 **en büyük faz**. Sabırlı ol. Her kavramı **canlı kod ile** denemeden geçme.

## Topic sırası

1. **[JMM & Memory Model](./01-jmm-memory-model/README.md)** — happens-before, volatile, cache coherency, instruction reordering
2. **[Synchronization Primitives](./02-synchronization-primitives/README.md)** — synchronized, volatile, atomic family, CAS, ABA problem
3. **[Lock Family](./03-locks/README.md)** — ReentrantLock, ReadWrite, StampedLock, Condition, deadlock fix
4. **[Executor Framework](./04-executor-framework/README.md)** — ThreadPoolExecutor, queue strategies, ForkJoinPool, custom thread factory
5. **[CompletableFuture & Async](./05-completable-future/README.md)** — async chains, exception handling, timeout, banking FX fetch
6. **[Concurrent Collections](./06-concurrent-collections/README.md)** — ConcurrentHashMap, BlockingQueue, CountDownLatch, Semaphore
7. **[Virtual Threads (Loom)](./07-virtual-threads-loom/README.md)** — Project Loom, pinning, ScopedValue
8. **[Structured Concurrency](./08-structured-concurrency/README.md)** — StructuredTaskScope, parallel KYC
9. **[JVM Internals](./09-jvm-internals/README.md)** — heap, GC algorithms, JIT, tiered compilation
10. **[Performance Tools](./10-performance-tools/README.md)** — JFR, async-profiler, MAT, jstack, GC log
11. **[JMH Benchmarking](./11-jmh-benchmarking/README.md)** — micro-benchmarks, blackhole, fork, banking benchmarks

Sonra:
- **[Mini-project](./mini-project/README.md)** — rate limiter, virtual thread benchmark, deadlock reproduce+fix, memory leak with MAT
- **[PHASE_TEST.md](./PHASE_TEST.md)**

## Faz 3'ün sonunda olman gereken yer

- [ ] "Happens-before ilişkisi nedir, volatile bunu nasıl sağlar?"
- [ ] "synchronized ve ReentrantLock arasındaki 5 fark nedir?"
- [ ] "ABA problem nedir, AtomicStampedReference ile nasıl çözülür?"
- [ ] "İki concurrent transfer A→B + B→A deadlock'a neden olur, lock ordering ile nasıl çözersin?"
- [ ] "CompletableFuture.allOf ile anyOf arasında ne zaman hangisini kullanırsın?"
- [ ] "ConcurrentHashMap.computeIfAbsent atomicity vs putIfAbsent farkı?"
- [ ] "Virtual thread pinning ne, nasıl tespit eder ve çözersin?"
- [ ] "G1 vs ZGC ne zaman hangisini seçersin?"
- [ ] "Memory leak'i MAT ile nasıl bulursun?"
- [ ] "JMH benchmark'ında blackhole neden gerekli?"

Hepsine "evet" → Phase 4'e geç (SQL & Oracle).
