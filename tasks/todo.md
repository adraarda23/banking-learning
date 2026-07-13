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

## Review
- Faz 1 pilot tamam: 34 diyagram, 70 kutu, tüm QA kontrolleri temiz (fence dengesi, mermaid etiket kuralları, link taraması)
- Kullanıcı incelemesi bekleniyor → onay sonrası kalan 11 faza yayılım
- Canlı önizleme: `mdbook serve --port 3000` çalışıyor
