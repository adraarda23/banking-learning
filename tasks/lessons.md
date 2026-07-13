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

## 2026-07-13 — İkinci revizyon turundan çıkan format standardı

7. **Mini task'ler kitapta çalışmıyor → "Kendini Sına" Soru-Cevap.**
   Task'ler dış proje ortamı varsayıyordu, kitap okuruna az bilgiyle yapılamaz göründü.
   Standart: topic sonunda 5-8 mülakat tarzı soru, cevap `<details><summary>Cevabı göster</summary>` içinde.
   details/summary etrafında BOŞ SATIR şart yoksa içerideki markdown render olmaz.

8. **25 satırdan uzun kod blokları okumayı öldürüyor.**
   Standart: öğreten kısım 5-15 satırlık parçalar + aralarında 1-2 cümle açıklama;
   komple listing `<details><summary>Tam kod: X (~N satır)</summary>` içine.

9. **Test rehberi + Claude-verify prompt → "Pratik yapmak istersen" katlanır eki.**
   İçerik korunur ama akışı bölmez. Mini-project dosyaları İSTİSNA: proje karakteri korunur,
   Soru-Cevap'a çevrilmez, sadece kod düzeni + katlama uygulanır.
