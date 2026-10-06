# Eapricot — API uçları (MVP)

Okuma uçları **yok**. Mağaza sayfaları server component olarak veriyi doğrudan Prisma'dan okur, ISR ile önbelleğe alır. Aşağıdakiler yalnız **yazma/işlem** uçları.

## Public

| Uç | Ne yapar | Koruma |
|---|---|---|
| `POST /api/cart/quote` | Sepet satırlarını (ürün, varyant, adet, ölçü, kişiselleştirme) alır; **sunucu fiyatını**, kargo ücretini ve satır hatalarını döner | zod, IP hız sınırı |
| `POST /api/checkout` | Fiyatı yeniden hesaplar → `Order(PENDING_PAYMENT)` + `pricingSnapshot` → Stripe Checkout Session → `{ url }` | zod, IP hız sınırı, kişiselleştirme doğrulaması |
| `GET /checkout/success?session_id=` (sayfa) | Oturumu Stripe'tan **okur**; webhook gecikmişse aynı tekil "ödendi" yolunu çalıştırır | Oturum kimliği + tutar karşılaştırması |
| `POST /api/newsletter` | Abone kaydı (`PENDING`) + onay e-postası | Honeypot, IP hız sınırı |
| `GET /api/newsletter/confirm?token=` | `CONFIRMED` | Tek kullanımlık jeton |
| `GET /api/newsletter/unsubscribe?token=` | `UNSUBSCRIBED` (CAN-SPAM) | Jeton |
| `POST /api/contact` | İletişim mesajı + admin'e bildirim | Honeypot, IP hız sınırı |

## Webhook

| Uç | Olaylar | Kural |
|---|---|---|
| `POST /api/webhooks/stripe` | `checkout.session.completed` → `PAID` + prova + onay e-postası; `checkout.session.expired` → `CANCELLED`; `charge.refunded` → `REFUNDED` | Ham gövdeyle **imza doğrulama**; `StripeEvent` ile tekillik; tutar/para birimi siparişle aynı olmalı; yalnız `PENDING_PAYMENT → PAID` geçişi (`updateMany` + `where status`) |
| `POST /api/webhooks/ortakdoku` *(Faz 2)* | Üretime alındı / kargolandı (takip no) | HMAC imza + zaman damgası + olay kimliği tekilliği |

## Admin (`/api/admin/*`)

Her uç **kendi içinde** oturum ve rol kontrolü yapar (`requireAdmin(role)`). `proxy.ts` yalnız yönlendirir. Yazma işlemleri `AdminAudit` kaydı bırakır.

| Alan | Uçlar | Rol |
|---|---|---|
| Oturum | `POST auth/login` · `POST auth/logout` · `POST auth/totp` | — |
| Ürün | `GET/POST products` · `GET/PATCH/DELETE products/:id` · `PUT products/:id/variants` · `PUT products/:id/fields` · `PUT products/:id/images` (sıra) · `PUT products/:id/preview-template` | EDITOR+ |
| Kategori / Koleksiyon | CRUD `categories`, `collections` · `PUT collections/:id/products` | EDITOR+ |
| Medya | `POST media/upload-url` (presigned PUT) · `POST media/complete` (sharp boyutlandırma) · `PATCH media/:id` (alt metin) · `DELETE media/:id` (kullanılmıyorsa) | EDITOR+ |
| Sipariş | `GET orders?status=&q=` · `GET orders/:id` · `PATCH orders/:id/status` · `POST orders/:id/tracking` (+ müşteriye e-posta) · `POST orders/:id/refund` (Stripe) · `GET orders/:id/production-sheet` (PDF) · `GET orders/export.csv` · *Faz 2:* `POST orders/:id/send-to-ortakdoku` | OPERATIONS+ · iade yalnız OWNER |
| İçerik | `GET/PUT content/:key` (zod şemasıyla) → `revalidateTag` | EDITOR+ |
| SSS / Yorum | CRUD `faq` · CRUD `reviews` + `PATCH reviews/:id/approve` | EDITOR+ |
| Bülten / Mesaj | `GET subscribers` · `GET subscribers/export.csv` · `GET messages` · `PATCH messages/:id` | EDITOR+ |
| Ayarlar | `GET/PUT settings/:key` (kargo, vergi, duyuru çubuğu, sosyal linkler) | OWNER |
| Kullanıcılar | CRUD `users` | OWNER |

## Ortak Doku Kanal API'si (Faz 2 — Ortak Doku tarafında YAPILACAK)

Bugün Ortak Doku'da dış mağazadan sipariş **alan** bir uç yok: siparişler panelden ya da Etsy'den çekme yoluyla geliyor. Faz 2 önerisi:

- `POST api.ortakdoku.com/kanal/siparisler`
  - Kimlik: bayi API anahtarı (`x-kanal-anahtar`). Anahtar Ortak Doku'da şifreli saklanır.
  - Kayıt `EtsyImport` benzeri bir "gelen sipariş" olarak düşer (`platform: EAPRICOT`). Ortak Doku'nun "Gelen Siparişler" ekranı ve ürün eşleme akışı aynen kullanılır.
  - **Fiyatı Ortak Doku hesaplar.** Eapricot'un satış fiyatı yalnız bilgi olarak gider; Ortak Doku'daki "gösterilen = tahsil edilen" kuralı bozulmaz.
- **Geri bildirim:** Ortak Doku, sipariş durum ve takip değişikliklerini kayıtlı webhook adresine imzalı gönderir. Ortak Doku'da `assertSafeWebhookUrl` ile SSRF süzgeci zaten var.
