# Kitaplaştırma Projesi: banking-learning → Web Kitap (mdBook)

## Karar
- Format: **mdBook** (web kitap; sol menü, arama, tema) — kaynak .md dosyaları editlenebilir kalır
- Görsellik: **Tam kitap editörlüğü** (Mermaid diyagramlar, özet/uyarı kutuları, bölüm geçiş sayfaları)
- Strateji: Önce altyapı + tüm içerik kitapta → Faz 1 pilot işçilik → onay → kalan fazlara yayılım

## Yapılacaklar
- [ ] Git init + baseline commit (tüm düzenlemeler geri alınabilir olsun)
- [ ] mdbook + mdbook-mermaid + mdbook-admonish binary kurulumu (~/.local/bin)
- [ ] book.toml + SUMMARY.md (12 faz, tüm topic'ler, mini-project'ler, phase test'ler)
- [ ] Kitap derlenip tarayıcıda açılıyor (tüm 120 dosya)
- [ ] PİLOT — Faz 1 tam kitap işçiliği:
  - [ ] Bölüm geçiş sayfası (faz kapağı)
  - [ ] Mermaid diyagramlar (hexagonal arch, request flow, error handling)
  - [ ] Özet kutuları + dikkat/uyarı kutuları (admonish)
- [ ] Kullanıcı stil onayı → kalan 11 faza yayılım (subagent'larla, faz faz)

## Notlar
- Sistem: brew/cargo/node YOK → GitHub release binary'leri kullanılıyor (arm64)
- Kaynak dosyalar yerinde kalıyor; `mdbook serve` ile canlı önizleme, dosya editlenince otomatik yenilenir

## Revizyon turu 1 (kullanıcı geri bildirimi, 2026-07-13)
- [x] Ana sayfa haritası "Unsupported markdown" → etiketlerde `N. ` kalıbı kaldırıldı, kompakt subgraph düzeni
- [x] Faz haritası ekranı kaplıyor → Hafta 1 / Hafta 2 yatay subgraph + CSS max-height sigortası
- [x] Topic linkleri 404 → tüm içerik linkleri `index.md`'ye çevrildi (127 dosya tarandı, 27'si düzeltildi)
- [x] Mermaid çizgi stili → neutral tema + linear curve (mermaid-init.js)
- [x] Faz 1 metinleri pedagojik yeniden yazım (8 dosya, paralel agent): öğretmen akışı, wall-of-text yok, ~%20-30 kısalma, teknik içerik korundu
- [x] Bonus düzeltmeler: mini-project ve error-handling'de iç içe fence hataları, geçersiz MockMvc assertion, typo

## Revizyon turu 2 (format kararları, 2026-07-13)
- [x] Vurgu katmanı: ilk tanımda kalın terim + bölüm başına ≤4 <mark>
- [x] Mini task'ler → "Kendini Sına" Soru-Cevap (56 soru, katlanır cevaplar)
- [x] >25 satır kod → öğreten parçalar + katlanır tam listing
- [x] Test rehberi + Claude-verify → "Pratik yapmak istersen" katlanır eki
- [x] Mini-project istisnası: proje karakteri korundu, sadece kod düzeni/katlama

## Faz 2 yayılımı (2026-07-13)
- [x] Faz 2 kapak sayfası + PHASE_TEST kitap formatı
- [x] 7 topic + mini proje tek geçişte tam standart (8 paralel agent)
- [x] QA temiz: 27 diyagram, 53 kutu, 56 soru, 89 katlanır kutu, 30 mark
- [x] Commit: fba7574

## Faz 3-4 yayılımı (2026-07-14)
- [x] Faz 3 (Concurrency & JVM): 11 topic + mini proje, 45 diyagram, 66 kutu, 84 soru — commit e69c0d0
- [x] Faz 4 (SQL & Oracle): 6 topic + mini proje, 29 diyagram, 44 kutu, 46 soru — commit 9b843ed
- [ ] Kalan: Faz 5-12 (8 faz)

## Review
- Faz 1 pilot tamam: 34 diyagram, ~70 kutu, 56 soru, 93 katlanır kutu; QA temiz
- Format standardı tasks/lessons.md + hafızada — kalan 11 faza aynı şablonla yayılacak
- Kullanıcı incelemesi bekleniyor → onay sonrası yayılım
- Canlı önizleme: `mdbook serve --port 3000` çalışıyor
