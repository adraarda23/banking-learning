# Lessons — banking-learning kitaplaştırma

## 2026-07-13 — İlk pilot revizyonundan çıkan dersler

1. **Mermaid node etiketi asla `N. ` (sayı+nokta+boşluk) ile başlamamalı.**
   Mermaid v10+ etiketi markdown olarak parse'lıyor, "1. Foundation" → "Unsupported markdown: list" hatası.
   Kural: "Faz 1 Foundation" veya "1.1 Mimari" (ondalıklı) formatı kullan.

2. **Uzun dikey TD zincirleri ekranı kaplıyor.**
   7+ node'lu flowchart TD tek sütun devasa oluyor. Kural: subgraph satırları + `direction LR` ile
   kompakt düzen; CSS'te `pre.mermaid svg { max-height }` sigortası.

3. **mdBook, içerik içi `README.md` linklerini `README.html`'e çevirir → 404.**
   Çıktıda dosya `index.html` olduğu için kırılıyor. Kural: içerik linklerinde `index.md` kullan
   (SUMMARY.md gerçek dosya yolu `README.md` ile kalır, o çalışıyor).

4. **Mermaid default teması + curvy çizgiler kitapta kötü duruyor.**
   Kural: `theme: neutral`, `flowchart.curve: linear`, kesikli/kesişen edge'lerden kaçın.

5. **Sadece süsleme yetmez — kullanıcı metinlerin pedagojik yeniden yazımını istiyor.**
   Wall of text kalmayacak; "neden → kavram → örnek → tuzak" akışı; ~%70-80 uzunluk;
   teknik içerik ve kod kaybı yok. Öğretmen tonu + fintech mühendis doğruluğu.

6. **Kullanıcı tekrar tekrar onay istenmesinden rahatsız.**
   Verilen revizyon seti bitene kadar izin sorma; otonom çalış, sonunda topluca raporla.
