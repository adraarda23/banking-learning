# Faz 5 — PHASE TEST

```admonish question title="Bu test ne işe yarar?"
Faz 6'ya geçmeden önce kendini sına. Bu bir sınav değil, **dürüstlük kontrolü**:
işaretleyemediğin her madde, hangi topic'e geri döneceğini gösterir.
Hepsine "evet" diyebiliyorsan hazırsın.
```

## Pratik test

- [ ] `core-banking` projesinde 3 banking job (EOD reconciliation, monthly interest, daily fraud) + master orchestration çalışıyor
- [ ] Multi-instance simulation: 3 app instance lokal, sadece 1'i job çalıştırıyor (ShedLock test)
- [ ] Partitioned interest job sequential'a göre 5x+ hızlı (defterimde tablo)
- [ ] Job kesip restart yaptım, kaldığı yerden devam etti (canlı denedim)
- [ ] Idempotency UNIQUE constraint ile garantili (duplicate run reject)
- [ ] Skip + Retry + DLQ pattern uygulanmış (kasten invalid data ile test)
- [ ] PL/SQL package'lar (Phase 4) Java'dan çağrılıyor

## Konsept testi — kendine sor

### Architecture
- [ ] Job, JobInstance, JobExecution, Step, StepExecution kavramları net
- [ ] JobRepository neden DB-backed (restart için)
- [ ] JobLauncher sync vs async ne zaman hangisi
- [ ] JobParameters identifying vs non-identifying — restart impact

### Chunk-oriented
- [ ] Reader/Processor/Writer lifecycle her chunk için
- [ ] Chunk transaction boundary — bir chunk içinde 999 başarılı + 1 fail = ?
- [ ] Chunk size küçük/büyük trade-off (memory vs throughput)
- [ ] JpaPagingItemReader vs JdbcPagingItemReader performans farkı
- [ ] @StepScope + late binding ile job parameter erişimi
- [ ] @JobScope ne zaman gerekli

### Skip / Retry / Restart
- [ ] Skip = atla, Retry = yeniden dene, Restart = kaldığı yerden başla
- [ ] SkipListener ile DLQ pattern (failed_*.csv tablosu)
- [ ] RetryListener ile retry visibility
- [ ] Idempotent writer pattern — UNIQUE constraint ile garanti
- [ ] Non-idempotent writer + restart = banking felaketi (neden)

### Listeners & Flow
- [ ] JobExecutionListener (before/after job)
- [ ] StepExecutionListener (before/after step + ExitStatus override)
- [ ] ChunkListener / ItemListener — production'da TRACE level
- [ ] Conditional flow `.on(...).to(...)` ile dallanma
- [ ] JobExecutionDecider — explicit karar
- [ ] Parallel flow `.split(executor)` — bağımsız step'ler

### Partitioning
- [ ] Multi-threaded step vs partitioning farkı
- [ ] Reader thread-safety concern (SynchronizedItemStreamReader)
- [ ] gridSize ile TaskExecutor pool size ilişkisi
- [ ] gridSize ile DB connection pool buffer
- [ ] ID range, modulo, date, category partitioning stratejileri

### Scheduling
- [ ] Spring @Scheduled cron format (6 segment) + timezone
- [ ] fixedRate vs fixedDelay farkı
- [ ] Multi-instance + ShedLock zorunluluğu (neden)
- [ ] lockAtMostFor vs lockAtLeastFor parametreleri
- [ ] Quartz vs ShedLock — ne zaman hangisi
- [ ] Misfire handling (fireAndProceed vs doNothing)
- [ ] RunIdIncrementer vs TimestampJobParametersIncrementer

## Banking domain anlama

- [ ] EOD reconciliation double-entry invariant'ı doğruluyor (sum debit = sum credit)
- [ ] Compound interest formülünü Phase 4 PL/SQL ile Phase 5 Java arasında **aynı sonuç** verdiğini doğruladım
- [ ] Fraud sliding window pattern (1 dakika içinde 5+ tx)
- [ ] Banking EOD penceresi (23:55-02:00) 3 job'u nasıl yerleştirdiğimi söyleyebilirim
- [ ] Master orchestration mismatch branch'inde DBA notify + halt mantığı

## Soft skills

- [ ] 3 job + orchestration kodu GitHub'da, commit mesajları anlamlı
- [ ] 15 defter notu mini-project'in sonunda dolduruldu
- [ ] Hata aldığımda log → MAT → jstack akışını uyguladım
- [ ] PL/SQL + Spring Batch entegrasyon kararını gerekçeli verdim

## Kaç gün?

Tahmin: ~14-18 gün (günde 2-3 saat). Daha az/çok olabilir, normal.

Kendine sor: "Bir mülakatta 'EOD job mimarisi tasarla' deseler, beyaz tahtaya 30 dakika içinde kapsamlı bir diagram çizebilir miyim?"

Evet → Faz 6'ya geç → `../06-messaging/`

Hayır → İlgili topic'in kavramlar bölümünü tekrar oku, mini task'i yeniden dene.

---

## Bonus — mid-junior'a geçtiğinin işareti

Phase 5'i bitirdiğinde şu cümleleri rahatça söyleyebilirsen banking batch tarafında **mid-junior+ seviyede**sin:

- "EOD job mimarisi kurabilirim, multi-instance safe + restartable + partitioned."
- "1M müşteri için partitioned interest accrual'ı tasarlayıp 10 worker thread ile paralel çalıştırabilirim."
- "Failed item'lar için DLQ pattern uygulayabilirim ve operator review workflow tasarlayabilirim."
- "Spring Batch + ShedLock + Quartz arasında **gerekçeli karar** verebilirim."
- "PL/SQL package'larını Spring Batch worker'larından çağırabilirim."
- "Master orchestration conditional flow tasarlayabilirim."

Bunlar TR bankalarında mid-level Java backend developer'ın bilmesi gereken minimum'lar. Phase 5'i ciddi yaptıysan bu seviyedesin.

---

## Banking'de batch tarafı ne kadar önemli?

TR bankalarının "core banking" sistemlerinin **gece 8 saati** EOD batch'lerle geçer:
- Faiz tahakkuku (mevduat, kredi)
- Karşılıklı mutabakat (intra-bank, inter-bank)
- Risk reporting
- Regulatory reporting (BDDK)
- Card scheme settlement
- Fraud retrospective scan

**Mid-level developer için en çok zaman geçen alan.** Phase 5'i sağlam bilirsen TR bank tech ekibinin **iş gücüne hazır** olursun.

```admonish success title="Sonraki durak: Faz 6"
Phase 6 mesajlaşma fazı — TR bankaları için modern uçtan biraz uzak (eski IBM MQ hâlâ var),
ama yeni servislerin standardı.
→ [Faz 6 — Messaging & Events](../06-messaging/index.md)
```
