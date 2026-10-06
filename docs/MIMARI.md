# Eapricot — Mimari (MVP)

## 1. Bileşenler

```
                 ┌──────────────────────── Cloudflare (DNS · SSL · CDN · WAF) ────────────────────────┐
 Müşteri ──────► │ eapricot.com            → nginx → eapricot-web (Next.js 16, :3100)                 │
 (tarayıcı)      │ media.eapricot.com      → R2 bucket "eapricot-media" (görseller, doğrudan CDN)     │
                 └─────────────────────────────────────────────────────────────────────────────────────┘
                                   │ Prisma                    │ HTTPS
                                   ▼                           ▼
                        Postgres (aynı sunucu,         Stripe Checkout + webhook
                        AYRI veritabanı: eapricot)     SMTP (sipariş e-postaları)
                                   │
                                   │ Faz 2: Kanal API (anahtar + HMAC)
                                   ▼
                        Ortak Doku (api.ortakdoku.com) — üretim + kargo, takip no geri döner
```

**Tek uygulama, üç yüz:**

| Yüz | Yol | Görev |
|---|---|---|
| Mağaza | `/`, `/shop/*`, `/products/*`, `/cart`… | Statik/ISR sayfalar; SEO ve hız |
| Admin | `/admin/*` | Ürün, sipariş, içerik ve medya yönetimi |
| API | `/api/*` | Fiyat hesabı, ödeme, webhook, form, admin işlemleri |

## 2. Teknoloji

| Katman | Seçim | Neden |
|---|---|---|
| Uygulama | **Next.js 16** (App Router) + React 19 + TypeScript | Ekip Ortak Doku'da aynı sürümü kullanıyor. Middleware'in adı **`proxy.ts`**, `tailwind.config` yok (token'lar `globals.css` içinde `@theme`) |
| Stil | Tailwind v4 | Ortak Doku ile aynı |
| Veri | **Prisma + PostgreSQL** (ayrı DB) | Ekip biliyor; migration düzeni hazır |
| Ödeme | **Stripe Checkout (hosted)** | B2C için en az kod ve PCI yükü. Apple Pay/Google Pay hazır gelir (Ortak Doku'da bayi için bilerek kapatıldı; tüketicide **açık** olmalı) |
| Görsel | **R2 + Cloudflare CDN**, yüklemede **sharp** ile boyutlandırma | Sunucu çalışırken görsel işlemez; aynı sunucudaki Ortak Doku'yla CPU yarışı olmaz |
| E-posta | Mevcut SMTP (sonra Postmark/Resend) | Sipariş onayı, kargo bildirimi, bülten onayı |
| Doğrulama | **zod** | Env, form ve API girdileri |
| Test | Vitest (birim) + **Playwright (uçtan uca, Stripe test modu)** | Ortak Doku'nun en büyük açığı e2e testin olmaması; burada baştan kurulur |

## 3. Klasör yapısı (repo: `emredonmez82/eap`)

```
eap/
├─ app/
│  ├─ (site)/                      # mağaza — server component, ISR
│  │  ├─ page.tsx                  # ana sayfa (SITE.md bölümleri)
│  │  ├─ shop/[category]/page.tsx  # Wood Décor / Wall Decals / Wallpaper
│  │  ├─ collections/[slug]/page.tsx  # For your home, For a new home…
│  │  ├─ products/[slug]/page.tsx  # ürün + kişiselleştirici
│  │  ├─ cart/page.tsx
│  │  ├─ checkout/success/page.tsx
│  │  ├─ our-story/page.tsx · faq/page.tsx · contact/page.tsx
│  │  └─ policies/[slug]/page.tsx  # shipping, returns, privacy, terms
│  ├─ admin/                       # admin — dinamik, noindex
│  │  ├─ login/page.tsx
│  │  ├─ page.tsx                  # özet
│  │  └─ orders/ products/ categories/ collections/ content/ media/ faq/ reviews/ subscribers/ messages/ settings/ users/
│  └─ api/
│     ├─ cart/quote/route.ts       # sunucu fiyatı
│     ├─ checkout/route.ts         # sipariş + Stripe Checkout Session
│     ├─ webhooks/stripe/route.ts
│     ├─ newsletter/… · contact/route.ts
│     └─ admin/…                   # admin uçları (her biri oturum + rol kontrolü)
├─ components/ (site/ · admin/ · ui/)
├─ lib/
│  ├─ db.ts · env.ts (zod) · auth.ts · stripe.ts · r2.ts · mail.ts
│  ├─ pricing.ts                   # TEK fiyat kaynağı (gösterilen = tahsil edilen)
│  ├─ personalization/             # alan doğrulama, önizleme şablonu, prova üretimi
│  └─ images.ts                    # sharp boyutlandırma + CDN adresleri
├─ prisma/ (schema.prisma · migrations/)
├─ proxy.ts                        # /admin yönlendirmesi (yalnız UX; asıl kapı handler içinde)
├─ deploy/ (nginx.conf · ecosystem.config.cjs · deploy.sh)
├─ e2e/                            # Playwright
└─ AGENTS.md                       # kurallar (taslak: YENI-OTURUM.md)
```

## 4. Akışlar

### 4.1 Gezinme
Sayfalar server component'tir; veri doğrudan Prisma'dan okunur (ayrı bir okuma API'si yok). Ana sayfa, kategori ve ürün sayfaları **ISR** ile üretilir. Admin kayıt yapınca `revalidateTag('product:<id>')` / `revalidateTag('home')` çağrılır ve sayfa saniyeler içinde güncellenir.

### 4.2 Kişiselleştirme
1. Ürünün **alanları** (ad, tarih, mesaj, seçim) ve **önizleme şablonu** vardır: taban görsel + metin kutuları (konum %, font, renk).
2. Tarayıcı, girilen metni taban görselin üstüne **SVG katmanı** olarak çizer. Önizleme anlıktır, sunucuya gitmez.
3. Sepete eklerken değerler sunucuda doğrulanır: zorunlu alan, uzunluk, izinli karakter, izinli font/renk.
4. Ödeme sonrası sunucu aynı şablondan **prova PNG** üretir (SVG → PNG, sharp). Prova R2'ye yazılır, sipariş kalemine bağlanır; müşteri e-postasında ve üretim föyünde kullanılır.
5. Baskıya/kesime hazır dosyanın otomatik üretimi **Faz 2**'de.

### 4.3 Sepet ve ödeme
1. Sepet tarayıcıda tutulur (localStorage). İçinde fiyat **yoktur**, yalnız ürün, varyant, adet, ölçü ve kişiselleştirme bilgisi vardır.
2. `POST /api/cart/quote` fiyatı **sunucuda** hesaplar (`lib/pricing.ts`, kuruş cinsinden).
3. `POST /api/checkout`:
   - Fiyatı yeniden hesaplar.
   - `Order(PENDING_PAYMENT)` kaydını ve `pricingSnapshot`'ı yazar.
   - Stripe Checkout Session açar (satırlar sunucu fiyatıyla, kargo seçenekleri; Stripe Tax açıksa `automatic_tax`).
   - Ödeme adresini döner.
4. Stripe → `POST /api/webhooks/stripe` (imza doğrulamalı, olay kimliğiyle **tekil**):
   - `checkout.session.completed` gelince `PAID` yazılır, prova üretilir, onay e-postası gider.
5. **Yedek:** Webhook gelmezse `/checkout/success` sayfası oturumu Stripe'tan **okur** ve aynı tekil yolu çağırır. Ortak Doku'da canlıda yaşandı: para çekildi, sipariş PENDING kaldı.

### 4.4 Sipariş → üretim → kargo
- **MVP:** Admin siparişi görür ve **üretim föyü** (PDF: kalemler, ölçü, kişiselleştirme, prova, adres) indirir. Siparişi Ortak Doku panelinde Eapricot bayi hesabından elle açar. Takip numarası gelince admin girer; müşteriye "kargoda" e-postası gider.
- **Faz 2 — Ortak Doku Kanal API'si** (Ortak Doku'da yeni uç gerekir):
  - Eapricot `PAID` siparişi Ortak Doku'ya gönderir (API anahtarı). Sipariş Ortak Doku'nun "Gelen Siparişler" ekranına `platform: EAPRICOT` olarak düşer; bu ekran zaten çok platformlu tasarlandı.
  - Ortak Doku takip numarasını imzalı webhook ile `/api/webhooks/ortakdoku` adresine geri yollar; Eapricot müşteriye bildirir.

## 5. Görsel hattı ("çok resim var")

1. Admin görseli seçer → `POST /api/admin/media/upload-url` → tarayıcı dosyayı **doğrudan R2'ye** yükler (presigned PUT). Dosya sunucudan geçmez.
2. `POST /api/admin/media/complete` → sunucu orijinali okur ve **sharp** ile bir kez boyutlandırır:
   - Genişlikler: 480 / 960 / 1600 / 2400 px.
   - Biçimler: **AVIF + WebP**.
   - Hedef: `media/<id>/<w>.<fmt>`.
3. Sayfalar `<picture>`/`next/image` kullanır ve **custom loader** ile doğrudan CDN adresini verir. Next'in çalışma anı optimizasyonu **kapalı** (`images.unoptimized` ya da loader). Gerekçe: 8 GB'lık ortak sunucuda Ortak Doku'yla CPU yarışmasın.
4. Dosya adları içerik özetli (hash) olur, `Cache-Control: public, max-age=31536000, immutable`.
5. **Alt metin zorunlu** (erişilebilirlik ve SEO). Admin alt metin girmeden kaydedemez.
6. Kurallar:
   - En fazla 25 MB; yalnız JPG, PNG, WebP. SVG yüklenmez (XSS riski).
   - Tür kontrolü dosyanın **içeriğinden** yapılır, uzantıya güvenilmez (Ortak Doku `file-validation.util.ts` deseni).
7. Ana sayfa hero görseli `priority`/preload ile yüklenir. LCP hedefi < 2,5 sn.

## 6. Güvenlik

- **Fiyat otoritesi sunucuda.** İstemci fiyat göndermez; Stripe satırları sunucu fiyatından kurulur. Ortak Doku'da bir kez ekranda $23 görünüp $46 tahsil edildi; aynı ders burada da geçerli.
- **Webhook:** İmza doğrulanır, olay kimliğiyle tekilleştirilir (`StripeEvent` tablosu); tutar ve para birimi siparişle karşılaştırılır.
- **Admin:**
  - Ayrı oturum çerezi (httpOnly, Secure, SameSite=Lax); bcrypt; giriş denemesi sınırı; OWNER için TOTP.
  - `proxy.ts` yalnız yönlendirme yapar. **Her admin uç kendi oturum ve rol kontrolünü yapar** (Ortak Doku dersi: proxy ve menü UX'tir, asıl kapı sunucudadır).
- **Formlar** (bülten, iletişim): honeypot + IP başına hız sınırı. IP, `CF-Connecting-IP` başlığından alınır (Ortak Doku `IpThrottlerGuard` deseni; tek kova = herkesi kilitler).
- **Sırlar** yalnız `.env`'de; kodda anahtar yazılmaz. Env okunduğu yerde değil, `lib/env.ts` içinde **zod** ile tek yerde doğrulanır.
- **Sayfada düz e-posta adresi yok;** iletişim formu kullanılır.
- **CSP:** Stripe Checkout hosted olduğu için dar tutulabilir. Stripe.js yalnız gerekiyorsa açılır.

## 7. Aynı sunucuda kurulum

| Konu | Ortak Doku (mevcut) | Eapricot (yeni) |
|---|---|---|
| Dizin | `/opt/printy` | `/opt/eapricot` |
| Süreç (pm2) | `od-backend` :3001, frontend :3000 | `eapricot-web` :3100 (`next start -p 3100`) |
| nginx | ortakdoku.com, app., api. | `eapricot.com`, `www.` → 127.0.0.1:3100 |
| Veritabanı | printy DB | `CREATE DATABASE eapricot;` + yalnız bu DB'ye yetkili `eapricot_app` kullanıcısı |
| R2 | mevcut bucket | `eapricot-media` + özel alan adı `media.eapricot.com` |
| Yedek | `yedekle.sh` | Aynı betiğe `eapricot` DB'si eklenir |
| Env | `.env.production` | `/opt/eapricot/.env.production` (ayrı) |

**Dikkat:**
- Next build ~1,5–2 GB RAM ister. İki projenin build'i **aynı anda** çalışmamalı; deploy betikleri kilit dosyasıyla sıraya girer.
- Prod'u kırmamak için ucuz bir **staging** kurulur: `staging.eapricot.com`, Stripe test modu, aynı sunucuda :3101. Ortak Doku'da staging yok ve bu bir eksiklik.
- Cloudflare'de e-posta DNS kayıtları (MX/SPF/DKIM) **proxy kapalı** olmalı (Ortak Doku dokümanındaki tuzak).

**Env değişkenleri (lib/env.ts'te zod ile doğrulanır):** `DATABASE_URL`, `SITE_URL`, `SESSION_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `MEDIA_BASE_URL`, `SMTP_HOST/PORT/USER/PASS/FROM`, Faz 2: `ORTAKDOKU_API_URL`, `ORTAKDOKU_API_KEY`, `ORTAKDOKU_WEBHOOK_SECRET`.

## 8. SEO, hız, uyum (ABD B2C)

- **SEO:**
  - `sitemap.xml`, `robots.txt`, kanonik adresler, OpenGraph.
  - Ürün sayfasında `Product` JSON-LD. `AggregateRating` **yalnız gerçek yorum varken** eklenir.
- **Erişilebilirlik (ADA):** Alt metinler, klavyeyle gezinme. Zeytin yeşili/krem renk kontrastı kontrol edilmeli.
- **Yasal sayfalar:** Privacy (CCPA), Terms, Shipping, Returns.
  - Analitik veya reklam pikseli eklenirse çerez onayı gerekir (Faz 2).
- **Pazarlama kuralları:**
  - **FTC:** Sahte ya da uydurma yorum yasak. "Made in USA" iddiası ancak doğruysa kullanılır.
  - **CAN-SPAM:** Bültende abonelikten çıkma bağlantısı ve fiziksel adres zorunlu.
