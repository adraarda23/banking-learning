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

## Review
(iş bitince doldurulacak)
