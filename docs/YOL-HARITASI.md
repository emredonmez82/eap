# Eapricot — Yol haritası

## Faz 0 — Hazırlık (kod öncesi, sizin taraf)

| # | İş | Kim |
|---|---|---|
| 1 | Alan adı (`eapricot.com`?) → Cloudflare'e ekle | Emre |
| 2 | ✓ Repo açıldı: `emredonmez82/eap` | Emre |
| 3 | Stripe hesabı (ayrı önerilir), ekstre adı `EAPRICOT`, test anahtarları | Aziz/Emre |
| 4 | R2 bucket `eapricot-media` + `media.eapricot.com` özel alan adı | Emre |
| 5 | Logo: SVG, şeffaf PNG ve favicon | Tasarım |
| 6 | İlk ürün listesi: 3 kategori × birkaç ürün; fiyat, ölçü, kişiselleştirme alanları; ürün fotoğrafları | Aziz |
| 7 | Satış vergisi ve yasal metinler (privacy/terms/returns/shipping) | Aziz + muhasebeci |

## Faz 1 — MVP

Her kilometre taşı ayrı PR'dır ve bir öncekinin üstüne gelir.

### M1 — İskelet
- Next.js 16 + Tailwind v4 + Prisma.
- `lib/env.ts` (zod), `AGENTS.md`, CI (tsc + vitest + eslint).
- Veritabanı şeması ve ilk migration; seed (kategoriler, koleksiyonlar, içerik blokları, SSS — metinler SITE.md'den).
- Admin girişi (OWNER/EDITOR/OPERATIONS) + `requireAdmin`; `proxy.ts`.
- Medya hattı: presigned yükleme → sharp boyutlandırma → CDN adresleri; alt metin zorunlu.
- Deploy betiği + nginx + pm2 (`eapricot-web` :3100) + staging (:3101).

**Bitti:** Admin giriş yapıp görsel yükleyebiliyor; görsel CDN'den AVIF/WebP olarak geliyor.

### M2 — Katalog
- Admin: ürün, varyant, ölçü fiyatı, kategori, koleksiyon, görseller.
- Mağaza: ana sayfa (SITE.md bölümleri; içerik `ContentBlock`'tan), kategori, koleksiyon ve ürün sayfaları. ISR + `revalidateTag`.
- `lib/pricing.ts` + birim testleri (sabit fiyat, ft², sınırlar, yuvarlama).

**Bitti:** Ürün admin'den eklenip yayınlanınca saniyeler içinde sitede görünüyor; fiyat karttaki ve sepetteki hesapla aynı.

### M3 — Kişiselleştirme
- Admin: alanlar + önizleme şablonu düzenleyici.
- Ürün sayfası: form + canlı SVG önizleme; sunucu doğrulaması.

**Bitti:** Müşteri adı yazınca önizleme anında değişiyor; geçersiz girdi (uzun metin, izinsiz karakter) sunucuda reddediliyor.

### M4 — Sepet ve ödeme
- Sepet (localStorage, fiyatsız) → `cart/quote` → `checkout` → Stripe Checkout (kargo seçenekleri, Stripe Tax kararına göre).
- Webhook (imza, tekillik, tutar kontrolü) + success sayfası yedek yolu.
- Ödeme sonrası prova PNG + sipariş onay e-postası.
- **Playwright e2e:** Stripe test kartıyla uçtan uca satın alma, 3D Secure test kartı dahil.

**Bitti:** Test kartıyla ödeme yapılınca sipariş `PAID` oluyor, prova ve e-posta üretiliyor. Webhook kapalıyken de success sayfası siparişi tamamlıyor.

### M5 — Operasyon ve içerik
- Admin sipariş ekranı: durum, takip no + kargo e-postası, üretim föyü PDF, iade, CSV.
- İçerik formları, SSS, yorumlar (izin kutusu zorunlu), bülten (çift onay), iletişim formu, ayarlar.

**Bitti:** Bir sipariş admin'den üretime alınıp takip no girilerek kapatılabiliyor; müşteri her adımda doğru e-postayı alıyor.

### M6 — Yayın
- Yasal sayfalar.
- SEO: sitemap, robots, JSON-LD, OG.
- Erişilebilirlik ve kontrast taraması; Lighthouse (LCP < 2,5 sn).
- Hata izleme; yedeğe DB ekleme; Stripe canlı anahtar.

**MVP bitti tanımı:** ABD'deki bir müşteri ürünü bulur, kişiselleştirir, önizler, öder ve prova içeren onay e-postası alır. Admin siparişi görür, üretim föyünü indirir, takip no girer; müşteri kargo e-postası alır. Bu akış e2e testle korunur.

## Faz 2 — Otomasyon ve büyüme

| İş | Not |
|---|---|
| **Ortak Doku Kanal API'si** | Ortak Doku'da yeni uç: `PAID` sipariş otomatik "Gelen Siparişler"e düşer, takip no imzalı webhook'la geri gelir (API.md) |
| Müşteri hesabı + sipariş sorgulama | E-posta + tek kullanımlık kod ile giriş |
| Kupon / ilk sipariş indirimi | Bülten "%10" kararı verilirse |
| Gerçek kargo fiyatı | Shippo canlı fiyat; Ortak Doku'daki Shippo modülü örnek |
| Yorum toplama e-postası | Teslimden N gün sonra; izin kutulu form |
| Analitik + çerez onayı | GA4/Meta pikseli eklenecekse önce onay |
| Baskı/kesim dosyası otomatik üretimi | Şablondan vektör (SVG/PDF) çıktı |
| Bülten aracına aktarım | Klaviyo / Mailchimp |

## Faz 3 — Fikirler

- **"Duvarında gör":** Ortak Doku'daki duvar önizleme ve sanal deneme (`backend/src/wall-share`, `backend/src/tryon`, admin `wall-preview`) Eapricot'a API ile açılabilir.
- Etsy/Amazon listing'lerinin Eapricot kataloğundan üretilmesi.
- Hediye kartı, istek listesi, çoklu dil.
