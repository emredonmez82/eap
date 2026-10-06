# Eapricot — B2C mağaza (MVP) mimari paketi

> **Eapricot:** ABD'deki son kullanıcıya kişiselleştirilebilir ev dekorasyonu (Wood Décor, Wall Decals, Wallpaper) satan marka sitesi. Üretim ve kargo Ortak Doku ağında yapılır.
> **Durum:** TASLAK · 2026-10-06 · Kod henüz yok. Repo: `emredonmez82/eap`. Kodlama yeni bir oturumda başlayacak (bkz. YENI-OTURUM.md).
> Ortak Doku **B2B** (bayi paneli), Eapricot **B2C** (tüketici mağazası). Bu fark kararların çoğunu belirliyor.

## Kısa karar

Eapricot **ayrı bir uygulama** olur: ayrı repo, ayrı veritabanı, ayrı alan adı. **Aynı sunucuda** çalışır ve Ortak Doku'ya yalnız API ile bağlanır.

**"Çok resim var, backend lazım mı?"** Evet, ama ince bir backend yeter, ayrı bir sunucu uygulaması gerekmez:

- Next.js içinde route handler + Prisma kullanılır.
- **Görseller backend'den geçmez.** Yükleme anında boyutlandırılıp R2'ye konur ve Cloudflare CDN'den servis edilir.
- Backend'in işi şunlar: sepet fiyatı (sunucu hesaplar), ödeme (Stripe), sipariş, kişiselleştirme doğrulaması, admin, bülten ve iletişim formu.

## Neden Ortak Doku'nun içine yazılmıyor

| Konu | Ortak Doku içinde olsaydı | Ayrı uygulama (seçilen) |
|---|---|---|
| Kitle | Bayi rolleri, çarpanları ve onay kapıları tüketiciye uymuyor | Tüketiciye özel, basit akış |
| Risk | Ortak Doku'da staging yok, 56 modül var; mağaza değişikliği portalı bozabilir | Ayrı deploy; biri düşerse öteki çalışır |
| Güvenlik | Tüketici hesapları bayi/personel tablosuna girer, `ownerScope` ve roller tüketiciyi bilmiyor | Ayrı veritabanı, ayrı oturum |
| Hız | Kampanya ve içerik değişikliği portalın deploy ritmine bağlanır | Mağaza istediği sıklıkta yayın yapar |
| Ortak kullanılanlar | — | Sunucu, Postgres sunucusu (ayrı DB), Cloudflare, R2 hesabı (ayrı bucket), SMTP |

**Değerlendirilip seçilmeyenler** (ileride yeniden bakılabilir):
- **Shopify:** Hızlı kurulur, ama kişiselleştirme önizlemesi ve Ortak Doku'ya otomatik üretim aktarımı için eklentiye bağımlı kalınır; aylık ücret ve işlem payı da var.
- **Medusa:** Hazır ticaret motoru, ama ayrı süreç ve Redis istiyor; MVP için ağır.
- **Payload CMS:** Hazır admin paneli sunuyor, ama ek öğrenme maliyeti var ve Next 16 uyumu doğrulanmalı.

## Dokümanlar

| Dosya | İçerik |
|---|---|
| [MIMARI.md](MIMARI.md) | Bileşenler, teknoloji, klasör yapısı, akışlar, görsel hattı, güvenlik, aynı sunucuda kurulum |
| [VERI-MODELI.md](VERI-MODELI.md) | Prisma şema taslağı (para kuruş cinsinden, fiyat anlık görüntüsü, webhook tekilliği) |
| [API.md](API.md) | Public, admin ve webhook uçları |
| [SITE.md](SITE.md) | Sayfalar, ana sayfa bölümleri (metinler dahil), tasarım işaretleri, gereken görseller |
| [ADMIN.md](ADMIN.md) | Admin ekranları ve roller |
| [YOL-HARITASI.md](YOL-HARITASI.md) | MVP kapsamı, kilometre taşları, Faz 2–3 |
| [ORTAK-DOKUDAN-ALINACAKLAR.md](ORTAK-DOKUDAN-ALINACAKLAR.md) | Ortak Doku'dan kopyalanacak/uyarlanacak kod ve dersler (dosya yollarıyla) |
| [YENI-OTURUM.md](YENI-OTURUM.md) | Yeni oturuma yapıştırılacak başlangıç talimatı + yeni repo için AGENTS.md taslağı |

## Karar bekliyor

1. **Alan adı:** `eapricot.com` mu? (Metinlerde yer tutucu olarak bu kullanıldı.)
2. ~~Repo~~ — **karar verildi:** `emredonmez82/eap` (ayrı repo).
3. **Stripe:** Ayrı hesap mı, aynı hesapta farklı ekstre adı (`EAPRICOT`) mı? Ayrı hesap öneriliyor (raporlama ve itiraz geçmişi ayrı kalır).
4. **ABD satış vergisi:** Stripe Tax açılacak mı, hangi eyaletlerde kayıt var? Muhasebeciyle netleşmeli. Kendi sitede vergiyi pazaryeri değil **satıcı** toplar.
5. **Kargo ücreti:** Sabit ücret mi, belli tutar üstü ücretsiz mi, gerçek fiyat (Shippo) mı? MVP için sabit ücret + ücretsiz eşik öneriliyor.
6. **Üretime aktarım:** MVP'de elle mi (admin'den üretim föyü), Faz 2'de Ortak Doku Kanal API'si mi?
7. **Yorumlar:** Yalnız gerçek ve izinli yorumlar yayınlanabilir. Maketteki örnek yorumlar yayına çıkamaz (FTC kuralı).
8. **Bülten indirimi:** Makette "%10 indirim" var, metinde yok. Hangisi geçerli?
9. **"Shop by space" bölümü:** Makette Home/Kids/Office/Entryway var, metinde "moment" kartları (For your home / For a new home / For someone special / Just because). Metin esas alındı.
10. **Logo:** Gelen logo raster. SVG ve şeffaf PNG sürümleri ile favicon gerekiyor.
