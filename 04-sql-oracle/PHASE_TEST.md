# Faz 4 — PHASE TEST

```admonish question title="Bu test ne işe yarar?"
Faz 5'e geçmeden önce kendini sına. Bu bir sınav değil, **dürüstlük kontrolü**:
işaretleyemediğin her madde, hangi topic'e geri döneceğini gösterir.
Hepsine "evet" diyebiliyorsan hazırsın.
```

## Pratik test

- [ ] Oracle XE Docker'da çalışıyor
- [ ] `core-banking` Oracle'a migrate edildi, Flyway migration'ları geçiyor
- [ ] interest_pkg + eod_reconciliation_pkg + fraud_check_pkg implementasyonu
- [ ] Java'dan PL/SQL package çağırma çalışıyor
- [ ] 1M-row sorgu öncesi/sonrası timing tablomda
- [ ] SKIP LOCKED ile 10-worker job queue çalışıyor
- [ ] Deadlock reproduction + fix denendi
- [ ] ORA-01555 hatasını canlı gördüm

## Konsept testi

### Index
- [ ] B-tree, bitmap, function-based index farkları
- [ ] Composite index leftmost prefix rule
- [ ] Selectivity ve cardinality'nin index seçimine etkisi
- [ ] Partial index (PostgreSQL) / function-based (Oracle)

### Execution plan
- [ ] EXPLAIN ANALYZE / DBMS_XPLAN.DISPLAY okuyabilirim
- [ ] Index Scan vs Seq Scan ayırt edebilirim
- [ ] Join algoritmaları (nested loop, hash, merge) ne zaman hangisi
- [ ] Stats güncelleme (ANALYZE / DBMS_STATS) önemi
- [ ] Histogram'ın skewed kolonda neden gerekli

### Window functions
- [ ] ROW_NUMBER, RANK, DENSE_RANK farkı
- [ ] LEAD, LAG ile temporal analiz
- [ ] PARTITION BY, ORDER BY, frame (ROWS/RANGE)
- [ ] Recursive CTE termination
- [ ] MERGE / ON CONFLICT atomik upsert

### PL/SQL
- [ ] Package spec vs body
- [ ] BULK COLLECT + FORALL ile loop+DML performans farkı
- [ ] Autonomous transaction audit kullanımı
- [ ] WHEN OTHERS THEN NULL anti-pattern'i
- [ ] %TYPE / %ROWTYPE kullanımı

### Oracle-specific
- [ ] Sequence CACHE değerinin performans etkisi
- [ ] RANGE INTERVAL partitioning otomatik partition
- [ ] Materialized view refresh stratejileri
- [ ] MVCC + undo segment ilişkisi (ORA-01555)
- [ ] ROWNUM `> n` ile yanlış pagination tuzağı

### Concurrency
- [ ] SELECT FOR UPDATE NOWAIT / WAIT n / SKIP LOCKED ne zaman hangisi
- [ ] SKIP LOCKED job queue pattern'i
- [ ] Lock ordering ile deadlock prevention
- [ ] DBMS_LOCK / pg_advisory_lock distributed singleton
- [ ] Oracle SERIALIZABLE vs PostgreSQL SSI

## Banking domain anlama

- [ ] Index'siz büyük tabloda sorgu felaketinin sebebini biliyorum
- [ ] Partition'sız transaction tablosunun yönetilmez olduğunu biliyorum
- [ ] Materialized view ile reporting cache pattern'i
- [ ] EOD reconciliation ne yapar, neden önemli
- [ ] Compound interest formülünü PL/SQL'de yazabilirim
- [ ] Job queue worker pattern'i ile EOD job distribution

---

Hepsine "evet" → **Faz 5'e geç → 05-batch/**

---

## Bonus — mid-junior'a geçtiğinin işareti

Phase 4'ü bitirdiğinde:

- "Yavaş SQL sorgusunu 5 dakikada teşhis edebilirim."
- "PL/SQL package yazıp Java tarafından entegre edebilirim."
- "Banking için Oracle partitioning + materialized view stratejisi kurabilirim."
- "Deadlock'u yakalayıp lock ordering ile çözebilirim."
- "DBA ile aynı dili konuşabilirim (AWR, v$session, undo segments, redo log)."

```admonish success title="Sonraki durak: Faz 5"
TR bank mülakatında Oracle/SQL detayları kritik. Phase 4'ten sonra bu kısımda **rahatsın**.
→ [Faz 5 — Spring Batch](../05-batch/index.md)
```
