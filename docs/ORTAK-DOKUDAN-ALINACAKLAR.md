# Ortak Doku'dan alınacaklar

Eapricot ayrı bir uygulama, ama Ortak Doku'da canlıda denenmiş çok parça var. Aşağıdakiler **kopyalanıp uyarlanır**; Eapricot çalışırken Ortak Doku koduna bağımlı olmaz. Yollar `emredonmez82/ortakdoku` reposuna göredir.

## Doğrudan uyarlanacak kod

| Ortak Doku dosyası | Ne alınır | Eapricot'ta |
|---|---|---|
| `backend/src/payments/payment-provider.ts` | Stripe'a SDK'sız `fetch` çağrıları; **Idempotency-Key** deseni ("ölen niyet" dersi dahil); `expireOrderSession` (eski ödeme bağlantısını kapatıp çift tahsilatı önleme); `retrieveCheckoutSession` (webhook yedeği) | `lib/stripe.ts` |
| `backend/src/payments/stripe-keys.util.ts` | Test/canlı anahtar setini **tek yerde** çözme (imza ve tahsilat aynı setten) | `lib/stripe.ts` |
| `backend/src/payments/payments.module.ts` | Webhook imza doğrulama (ham gövde), `markPaid` tekilliği (`updateMany … where status`), tutar doğrulaması, **reconcile** işi (webhook gelmezse Stripe'a sorup tamamlama) | `app/api/webhooks/stripe` + success sayfası |
| `backend/prisma/schema.prisma` → `ProcessedWebhookEvent` | Olay kimliğiyle tekillik tablosu | `StripeEvent` |
| `backend/src/common/crypto.util.ts` | AES-256-GCM (TOTP sırrı, hassas alanlar) | `lib/crypto.ts` |
| `backend/src/common/ip-throttler.guard.ts` | Hız sınırını **gerçek istemci IP'sine** bağlama (Cloudflare arkasında) | `lib/rate-limit.ts` |
| `backend/src/common/file-validation.util.ts` | Uzantı yerine **içerikten** MIME tespiti | medya yükleme |
| `backend/src/files/preview.util.ts` | sharp ile WebP önizleme; `GUVENLI_PIKSEL` (sıkıştırma bombası sınırı) | `lib/images.ts` |
| `backend/src/files/files.service.ts` | R2 presigned yükleme deseni | `lib/r2.ts` |
| `backend/src/mail/mail.service.ts` | SMTP çözümü, admin "test maili" ucu, `250` kabul satırını loglama (teslimat teşhisi için çok işe yaradı) | `lib/mail.ts` |
| `backend/src/config/env.schema.ts` + `app.module.ts` | Env'in tek yerde doğrulanması (Joi) | `lib/env.ts` (zod) |
| `backend/src/common/app-url.util.ts` | Site adresinin tek kaynağı (`frontendBaseUrl`) | `lib/env.ts` → `SITE_URL` |
| `backend/src/public/katalog-vitrini.ts` | Public veride **beyaz liste** (spread yok; admin yeni gizli alan eklese de sızmaz) | ürün/sayfa sorguları |
| `frontend/proxy.ts` | Next 16 middleware adı ve fail-closed yaklaşımı | `proxy.ts` (`/admin`) |
| `frontend/next.config.ts` | Güvenlik başlıkları + CSP temeli | `next.config.ts` |
| `frontend/app/globals.css` | Tailwind v4 `@theme` token düzeni | `app/globals.css` (SITE.md token'ları) |
| `backend/deploy/deploy.sh` | Env gölgesini temizleme, `pm2 restart --update-env`, portu açıkça verme, sağlık kontrolü | `deploy/deploy.sh` |
| `backend/deploy/nginx/` | Sunucu bloğu şablonu | `deploy/nginx.conf` |
| `backend/deploy/yedekle.sh`, `yedek-r2.mjs` | DB yedeğini R2'ye atma | Aynı betiğe `eapricot` DB'si eklenir |

## Faz 2'de işe yarayacaklar

| Ortak Doku | Neden |
|---|---|
| `backend/src/etsy/etsy-import.module.ts`, `gelen-siparisler.util.ts` | Kanal API'si bu "gelen sipariş" modelinin üstüne kurulur (`platform: EAPRICOT`) |
| `backend/src/etsy/etsy-takip.ts` | Takip numarasını dış sisteme geri yazma görevinin deseni (tekillik, deneme sınırı) |
| `backend/src/shippo/` | Gerçek kargo fiyatı ve etiket |
| `backend/src/webhooks/` (`assertSafeWebhookUrl`) | Ortak Doku → Eapricot webhook'unda SSRF süzgeci |
| `backend/src/wall-share`, `backend/src/tryon`, admin `wall-preview` | "Duvarında gör" özelliği (Faz 3) |

## Bilerek FARKLI yapılacaklar (B2B ≠ B2C)

| Ortak Doku | Eapricot | Neden |
|---|---|---|
| Ödemede yalnız kart; Apple Pay, Google Pay ve Link kapalı (bayi kararı) | Stripe Checkout'un cüzdanları **açık** | Tüketicide dönüşüm oranı |
| Rol çarpanları (USER 2×, üye 1×), bakiye ve kredi lotları | Tek perakende fiyat, bakiye yok | B2C |
| `ownerScope`, lider/ekip yetkileri | Tek mağaza, yalnız admin rolleri | Çok kiracılı yapı yok |
| Kendi sayfada Payment Element | Hosted Checkout | Daha az kod, daha az PCI yükü. 3D Secure, ad ve e-postayı Stripe kendisi toplar |
| Staging yok, e2e test yok | **Staging + Playwright e2e baştan** | Ortak Doku'nun bilinen açığı |

## Ortak Doku'da canlıda yaşanan, Eapricot'ta baştan önlenecek hatalar

1. **Gösterilen ≠ tahsil edilen** (ekranda $23, tahsilat $46): Fiyat tek fonksiyondan gelir; istemci fiyat göndermez.
2. **Ödeme yalnız webhook'a bağlıydı** (para çekildi, sipariş beklemede kaldı): Success sayfası ve reconcile yedeği olmalı.
3. **Çift tahsilat** (iki sekme, iki ödeme bağlantısı): Eski oturum kapatılır, siparişte tek etkin oturum tutulur.
4. **`'never'` alanı + değer yok → ödeme düğmesi asılı kaldı:** Hosted Checkout'ta bu risk yok. Payment Element'e geçilirse `frontend/lib/kart-sahibi.ts` deseni kullanılır.
5. **Env okunup şemaya eklenmemişti:** Yeni değişken önce `lib/env.ts`'e eklenir.
6. **Menüde gizli ama adresle açık sayfa:** Admin sayfalarında asıl kapı sunucu tarafındaki rol kontrolüdür.
7. **SMTP şifresi bozuktu, kayıt kodu sessizce gitmedi** (2026-09-22): Gönderim sonucu loglanır, admin "test maili" ucu olur.
8. **World First kartı 3D Secure'da `R` aldı** (2026-10-05): Kartı veren bankanın ret kararı; kodla çözülmez. Admin sipariş ekranında Stripe ödeme bağlantısı bulunsun, teşhis hızlanır.
