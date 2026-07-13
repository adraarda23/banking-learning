# Faz 4 — SQL & Oracle

## Bu fazın hedefi

Türkiye'deki büyük bankaların **veri tabanı dünyası Oracle hâkim**. Garanti, İş Bankası, Akbank, Yapı Kredi, Halkbank, Vakıfbank, Ziraat — hepsinin core banking sistemleri Oracle (Exadata) üzerinde çalışır. PostgreSQL son yıllarda yan sistemlerde (raporlama, microservice'lerin local store'ları, internal tooling) yaygınlaşıyor ama **çekirdek hesap, mevduat, kart, kredi ledger'ı Oracle**.

Bu yüzden TR bank backend developer'ı için **Oracle bilgisi pazarlık konusu değil**. SQL'in temelini biliyor olabilirsin, ama bu fazda:

- **Index internals** — B-tree, bitmap, function-based, composite — neden bazen index hızlandırır, bazen yavaşlatır
- **Execution plan tuning** — `EXPLAIN PLAN`, join algoritmaları, statistics, histograms
- **Advanced SQL** — window function'lar, recursive CTE, MERGE, hierarchical query (CONNECT BY)
- **PL/SQL** — Oracle'ın stored procedure dili; package, cursor, BULK COLLECT, FORALL, autonomous transaction
- **Oracle-specific** — sequence, partitioning, materialized view, MVCC, ORA-01555
- **DB concurrency** — `SELECT FOR UPDATE`, `SKIP LOCKED`, deadlock, lock türleri

Mini-project: **core-banking projesini PostgreSQL'den Oracle'a migrate** edeceksin. Flyway dialect değiştirecek, 3 PL/SQL package yazacaksın (interest, EOD reconciliation, fraud check), 1M-row sorgusunda index'in farkını göreceksin, ORA-01555 hatasını **kasten** üreteceksin.

## Önbilgi

- Faz 1-3 tamamlandı (Foundation + JPA + Concurrency)
- SQL temel: SELECT, JOIN (INNER/LEFT/RIGHT), GROUP BY, HAVING, subquery
- DDL temel: CREATE TABLE, ALTER TABLE, CREATE INDEX
- Faz 2'den JPA'nın altında SQL'in nasıl üretildiğini gördün
- Faz 3'ten optimistic vs pessimistic locking biliyorsun

## Topic listesi

| # | Topic | Süre tahmini | Konu |
|---|---|---|---|
| 4.1 | [Index Internals](01-index-internals/README.md) | ~5 saat | B-tree derinlik, bitmap, composite, function-based, selectivity |
| 4.2 | [Execution Plan & Tuning](02-execution-plan-tuning/README.md) | ~5 saat | `EXPLAIN PLAN`, `EXPLAIN ANALYZE`, join algoritmaları, CBO, histogram, hints |
| 4.3 | [Window Functions & Advanced SQL](03-window-functions-advanced-sql/README.md) | ~5 saat | ROW_NUMBER, LAG, LEAD, frame clause, CTE, recursive CTE, MERGE, CONNECT BY |
| 4.4 | [PL/SQL](04-plsql/README.md) | ~7 saat | Block yapısı, cursor, exception, BULK COLLECT, FORALL, package, autonomous transaction |
| 4.5 | [Oracle-Specific Features](05-oracle-specific/README.md) | ~5 saat | Sequence, partitioning, materialized view, MVCC, ORA-01555, redo/undo |
| 4.6 | [DB Concurrency & Locking](06-db-concurrency-locking/README.md) | ~5 saat | Isolation level, `FOR UPDATE`, `SKIP LOCKED`, deadlock, enqueue, DBMS_LOCK |
| 4.7 | [Mini-Project: PostgreSQL → Oracle Migration](mini-project/README.md) | ~7 gün | core-banking'in Oracle versiyonu + 3 PL/SQL package + index analizi + worker queue |
| | [PHASE_TEST.md](PHASE_TEST.md) | 1 saat | Self-assessment |

Toplam ~30 saat öğrenme + 7 gün proje. Acele etme. Oracle, sabırla derinleşilen bir dünya.

## Oracle'ın TR banking'deki yerini anlama (kısa kültür)

**Neden Oracle?**
1. **30+ yıl tarihsel mevcudiyet** — TR bankaları 1990'larda Oracle aldı, üzerine kuruldular
2. **Exadata** — Oracle'ın özel donanım+yazılım kombinasyonu, OLTP+OLAP hibrit yük için optimize
3. **Real Application Clusters (RAC)** — multiple node tek DB cluster, banking için critical
4. **Data Guard** — standby replication, disaster recovery için BDDK standardı
5. **PL/SQL ekosistemi** — banka iş kurallarının önemli kısmı stored procedure'larda yaşıyor (10-20 yıl önce yazılmış, hâlâ production'da)
6. **DBA kadrosu** — Türkiye'de Oracle DBA sayısı PostgreSQL DBA'dan kat kat fazla

**PostgreSQL'in yan rolü:**
- Microservice'lerin local store'u
- Raporlama / analytics
- Internal tooling (CRM, ticketing)
- Yeni greenfield projeler (özellikle açık bankacılık / yeni nesil ürünler)

**Bu fazın TR bank gerçeği:** Mülakatta "Oracle deneyimin var mı?" sorusu **kritik**. "Hayır ama öğrenirim" cevabı kabul edilir ama "Evet, şu konularda dokundum: PL/SQL package, index tuning, partitioning, ORA-01555 ile karşılaştım" cevabı seni **mid-level'a fırlatır**.

## Bu fazda kullanacağın araçlar

- **Oracle Database XE 21c** (Express Edition) — ücretsiz, lokalde Docker ile çalıştıracağız
- **SQL*Plus** — komut satırı client
- **SQL Developer** — GUI client (Oracle'ın resmi)
- **DBeaver** — alternatif çok-DB GUI client (PostgreSQL + Oracle aynı arayüzde)
- **DBMS_XPLAN** — Oracle'ın execution plan analyzer paketi
- **DBMS_STATS** — istatistik toplama paketi
- **Flyway** — Oracle dialect'i destekler, migration'ları aynı tool'la yöneteceğiz

## Faz çıkışında kazanacakların

1. **`EXPLAIN PLAN` okuyabileceksin** ve "bu sorgu yavaş çünkü full table scan yapıyor" diyebileceksin
2. **Index seçimi konusunda karar verebileceksin** — kompozit kolonun sırası, partial vs full, function-based ne zaman
3. **PL/SQL package yazabileceksin** — interest hesaplama, EOD recon, fraud rule engine
4. **Window function'larla raporlama sorgusu yazabileceksin** — running balance, top earners, monthly aggregation
5. **Partitioning stratejisi tasarlayabileceksin** — `transactions` tablosu 100M satır olduğunda ne yaparsın
6. **MVCC'yi anlayacaksın** — Oracle'ın undo segment'leri, read consistency, ORA-01555
7. **`SKIP LOCKED` ile worker queue yazabileceksin** — async job processing pattern
8. **PostgreSQL ve Oracle arasındaki major farkları** bileceksin

## Bu fazın anti-pattern uyarısı

Bu fazda kolayca yapılan yanlışlar:

1. **"Index ekleyince hızlanır" kafası** — write-heavy tabloya 10 index → INSERT yavaşlar, lock contention artar
2. **`SELECT *`** — yarın kolon eklenirse ne olur? Network bandwidth, memory waste, plan stability
3. **PL/SQL'de business logic toplama** — 20 yıl önce yapıldı, bugün hâlâ acı çekiliyor. Stored procedure infrastructure code için (audit, EOD, partitioning) ama core business logic Java'da kalmalı
4. **`WHEN OTHERS THEN NULL`** — PL/SQL'in en zararlı pattern'i. Hata yutuluyor, kimse fark etmiyor
5. **Statistics toplamayı unutmak** — production'da statistics eski → CBO yanlış plan seçer → query 10x yavaşlar
6. **`FOR UPDATE` lock'unu uzun tutmak** — banking'de transfer 50ms olmalı, 5 saniye değil
7. **Materialized view'i hep `ON COMMIT` yapmak** — write performance ölür, batch refresh düşün

## İlerleme stratejisi

Bu fazda iki paralel okuma yap:
1. **Topic'leri sırayla oku ve mini task'leri yap** — kavramları öğren
2. **Mini-project'i adım adım inşa et** — Oracle XE'yi Docker'da çalıştır, PostgreSQL'den migrate et

Topic 4.1 ve 4.2 (index + execution plan) **tüm fazın temeli**. Onları sıkı çalış. PL/SQL (4.4) **en uzun** topic, sabırlı ol — bankada bu tek başına bir lisans değeri taşıyor.

→ Başla: [01-index-internals/README.md](01-index-internals/README.md)
