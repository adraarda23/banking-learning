# Faz 5 — Spring Batch: Banking EOD ve Toplu İşleme

## Hedef

Banking core sistemlerinin **gece yarısı ne yaptığını** anlamak ve kendi `core-banking` projende **Spring Batch ile EOD (End-of-Day) job'ları** yazabilmek.

TR bankalarında her gece 23:00 sonrasında **onlarca batch job** çalışır: günsonu mutabakat, faiz tahakkuk, kredi taksit oluşturma, kart hesap kesimi, MASAK raporu, BDDK raporu, fraud taraması, dosya alış/veriş (clearing house, EFT, Swift). Bu işlerin **deterministik, restartable, izlenebilir, idempotent** olması zorunludur.

Bu faz sonunda:

- Spring Batch'in `Job` / `Step` / `JobExecution` modelini içselleştirmiş olacaksın
- Chunk-oriented vs Tasklet step farkını kavrayacaksın
- 1 milyon satırlık tabloyu **memory'ye tüketmeden** işleyebileceksin
- Skip / Retry / Restart semantiğini banking örneklerinde uygulayacaksın
- Listener'larla cross-cutting kaygıları (audit, DLQ, notification) çözeceksin
- Partitioning ile saatlerce süren job'u dakikalara indireceksin
- `@Scheduled` + Quartz + ShedLock ile cluster ortamında **tek instance** çalıştıracaksın

## Süre

Yaklaşık 2-2.5 hafta. Topic başına 1-1.5 gün okuma + task, mini-project 3-4 gün.

## Önbilgi

- Faz 1 tamamlandı (`core-banking` çalışıyor)
- Faz 2 (JPA, transaction) tamamlandı — chunk transaction boundary'sini anlaman için kritik
- Faz 3 (concurrency) tamamlandı — multi-threaded step ve partitioning için
- Faz 4 (SQL/Oracle) tamamlandı — paging reader'ın altında SQL fetch mantığı var

---

## Topic'ler

| # | Topic | Süre | Kritik kavram |
|---|-------|------|---------------|
| 1 | [01-batch-architecture](./01-batch-architecture/index.md) | 1 gün | Job, Step, JobInstance vs JobExecution, JobRepository |
| 2 | [02-chunk-oriented](./02-chunk-oriented/index.md) | 1.5 gün | ItemReader/Processor/Writer, chunk size, paging reader |
| 3 | [03-skip-retry-restart](./03-skip-retry-restart/index.md) | 1.5 gün | SkipPolicy, RetryPolicy, ExecutionContext, restart semantics |
| 4 | [04-listeners-flow](./04-listeners-flow/index.md) | 1 gün | Listener'lar, conditional flow, decider, split |
| 5 | [05-partitioning-parallel](./05-partitioning-parallel/index.md) | 1.5 gün | Partitioner, multi-threaded step, throttle limit |
| 6 | [06-scheduling](./06-scheduling/index.md) | 1 gün | @Scheduled, Quartz, ShedLock |
| M | [mini-project](./mini-project/index.md) | 3-4 gün | EOD Reconciliation + Interest Accrual + Fraud Report |
| T | [PHASE_TEST.md](./PHASE_TEST.md) | 0.5 gün | Self-assessment |

---

## Banking domain bağlamı — neden bu faz kritik

Bir TR bankasının **gerçek gecesi**:

```
23:00  EOD penceresi açılır (online işlemler "value date next day" olur)
23:05  Receiver job — TC Merkez Bankası'ndan EFT mutabakat dosyası gelir
23:15  Posting job — gün içi pending transactions account postings'e dönüşür
23:30  Reconciliation job — journal_entries.sum() == accounts.balance.sum() kontrolü
23:45  Interest accrual job — mevduat/kredi faizleri günlük tahakkuk
00:15  Card cycle job — kredi kartı hesap kesim tarihi gelmiş müşteriler
00:30  Fee charge job — havale ücreti, kart aidat, EFT komisyonu
01:00  Reporting job — BDDK günlük likidite raporu CSV
01:30  Fraud scan job — şüpheli pattern'ler (split deposit, round-trip)
02:00  MASAK report — STR / KYC anomalileri
03:00  File send job — clearing house'a dosya gönder (SFTP)
03:30  Statement generation — müşteri ekstreleri PDF
04:30  Cleanup job — geçici tablolardaki gün içi verileri arşive taşı
05:00  EOD penceresi kapanır, sistem online'a döner
```

Her bir kutucuk **bir Spring Batch Job'u**. Bunların:

- **Deterministik** olması: aynı input → aynı output (idempotent)
- **Restartable** olması: gece 02:14'te crash olursa, sabah başa dönmek yok — kaldığı yerden devam
- **Auditable** olması: hangi job ne zaman başladı, kaç satır işledi, ne zaman bitti
- **Parallelizable** olması: 10M müşteri için faiz hesabı saatler değil dakikalar sürmeli
- **Skip/Retry** capable: 1 bozuk satır job'u durdurmaz, transient hata retry edilir

Bu faz tam olarak bu özellikleri öğretiyor.

---

## Spring Batch'in mimari resmi (high-level)

```
┌─────────────────────────────────────────────────────────────┐
│  Spring Batch Framework                                     │
│                                                             │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ JobLauncher │───→│     Job      │───→│    Step 1    │    │
│  └─────────────┘    │              │    └──────────────┘    │
│                     │              │    ┌──────────────┐    │
│                     │              │───→│    Step 2    │    │
│                     │              │    └──────────────┘    │
│                     └──────┬───────┘                        │
│                            │ persists state                 │
│                            ↓                                │
│                     ┌──────────────────┐                    │
│                     │  JobRepository   │                    │
│                     │  (BATCH_* tables)│                    │
│                     └──────────────────┘                    │
│                                                             │
│   Step (chunk-oriented):                                    │
│   ┌────────────┐  ┌──────────────┐  ┌────────────┐          │
│   │ ItemReader │→ │ItemProcessor │→ │ ItemWriter │          │
│   └────────────┘  └──────────────┘  └────────────┘          │
│   read N items → process each → write batch (1 TX)          │
└─────────────────────────────────────────────────────────────┘
```

`JobRepository` PostgreSQL'de `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION`, `BATCH_JOB_EXECUTION_CONTEXT`, `BATCH_STEP_EXECUTION_CONTEXT`, `BATCH_JOB_EXECUTION_PARAMS` tablolarında state tutar. Bu tablolar **restart'ın anahtarıdır**.

---

## Tamamlama kriterleri

- [ ] 6 topic README'sini bitirdim, her topic'in mini task'lerini yaptım
- [ ] `core-banking` projende `eod-reconciliation`, `interest-accrual`, `fraud-report` job'ları çalışıyor
- [ ] 1M satırlık fake data ile chunk size 1 vs 1000 benchmark'ını yaptım
- [ ] Bir job'u **kasten kırdım** (DB connection drop, OOM simulation) ve restart ile bitirdim
- [ ] `BATCH_JOB_EXECUTION` tablosunda kendi job'larımın history'sini gördüm
- [ ] ShedLock'lu bir scheduled job'u 2 instance'ta çalıştırdım, yalnızca biri tetikledi
- [ ] PHASE_TEST'in tüm sorularına cevap verebiliyorum

Sonraki adım: Faz 6 → [06-messaging/](../06-messaging/)

---

## Faza özel notlar (defterine yaz)

1. "Spring Batch'i ne zaman seçerim, ne zaman seçmem?" — junior developer'a 30 saniyede anlat.
2. "EOD batch job'u neden online API'den farklı yazılır?" — TX boundary, retry semantiği, observability açısından.
3. "Bir job restart edilebilir olmak için hangi koşulları sağlamalı?" — checkpoint, idempotency, no random side effect.
4. "Partitioning vs multi-threaded step ne zaman hangisi?" — order, reader state, throughput perspektifinden.
