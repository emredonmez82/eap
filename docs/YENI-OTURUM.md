# Yeni oturumu başlatma

## Durum

- Repo hazır: `emredonmez82/eap`. Dokümanlar `docs/` altında, kurallar kökteki `AGENTS.md` dosyasında.
- Yeni Claude Code oturumunu **eap reposunda** açın ve aşağıdaki metni ilk mesaj olarak yapıştırın.

## Yapıştırılacak ilk mesaj

```
Eapricot projesine başlıyoruz: ABD'deki son kullanıcıya kişiselleştirilebilir ev dekorasyonu
(Wood Décor, Wall Decals, Wallpaper) satan B2C mağaza + admin. Üretim/kargo Ortak Doku'da.

1) AGENTS.md ve docs/ altındaki tüm dokümanları oku: README, MIMARI, VERI-MODELI, API, SITE,
   ADMIN, YOL-HARITASI, ORTAK-DOKUDAN-ALINACAKLAR. Kararlar orada; çelişki görürsen bana sor.
2) Kopyalanacak kod için emredonmez82/ortakdoku reposunu OKUMA erişimiyle ekle (oraya yazma).
3) YOL-HARITASI'ndaki M1 (İskelet) ile başla. Ortak Doku'daki dosyaları ORTAK-DOKUDAN-ALINACAKLAR'daki
   tabloya göre KOPYALAYIP UYARLA; Ortak Doku'ya çalışma zamanında bağımlılık kurma.

Kurallar: main'e push yok; her iş bulut/<kisa-ad> dalında + PR, kendi PR'ını birleştirebilirsin.
Deploy ve sunucu bende; sunucuya bağlanma, komutu bana ver. Para değişikliğine eski kodda başarısız
olan test yaz. "Karar bekliyor" maddelerini PR'da yaz. Türkçe, kısa yaz, tablo kullan.
```

---

## AGENTS.md (kökteki dosyanın kaynağı)

```markdown
# AGENTS.md — Eapricot

> Bu dosya, kod tabanında çalışacak her ajanın ilk okuyacağı yerdir. Kararlar docs/ altında;
> kod ile çelişirse kaynak kod esastır.

## Proje nedir
ABD'deki son kullanıcıya (B2C) kişiselleştirilebilir ev dekorasyonu satan marka sitesi + admin.
Kategoriler: Wood Décor · Wall Decals · Wallpaper. Üretim ve kargo Ortak Doku ağında
(emredonmez82/ortakdoku — ayrı uygulama, ayrı veritabanı; bağlantı yalnız API ile).

## Yapı
Tek Next.js 16 uygulaması: mağaza (app/(site)) + admin (app/admin) + API (app/api).
Prisma + PostgreSQL (DB: eapricot) · Stripe Checkout · R2 + Cloudflare CDN · sharp.
Test: `npm run test` (vitest) + `npm run e2e` (Playwright, Stripe test modu). `npx tsc --noEmit` temiz olmalı.

## ⛔ Bozulmaması gereken kurallar
### Para
- Fiyat otoritesi sunucudadır. İstemci fiyat GÖNDERMEZ; sepet localStorage'da fiyatsız tutulur.
- Ürün kartı, sepet ve Stripe satırı TEK fonksiyondan türer: lib/pricing.ts. Aynı kuralı ikinci yere yazma.
- Para tamsayı kuruş (*Cents). Sipariş anında pricingSnapshot dondurulur.
- Ödeme onayı: webhook (imza + StripeEvent tekilliği + tutar kontrolü) VE success sayfası yedeği.
  Yalnız PENDING_PAYMENT → PAID geçişi (updateMany … where status).
### Yetki
- Admin uçlarının her biri requireAdmin(rol) çağırır. proxy.ts ve menü yalnız UX'tir.
- İade yalnız OWNER. Her admin yazma işlemi AdminAudit'e düşer.
### Ortam / güvenlik
- Kodda sır yazma. Env tek kaynak: lib/env.ts (zod). Yeni değişken önce oraya.
- Görsel yükleme yalnız admin + presigned; MIME içerikten tespit; SVG yüklenmez; alt metin zorunlu.
- Formlarda honeypot + gerçek istemci IP'sine (CF-Connecting-IP) bağlı hız sınırı.
- Public sayfada düz e-posta yok (iletişim formu).
### İçerik / hukuk (ABD)
- Yalnız gerçek ve izinli yorum yayınlanır (FTC). AggregateRating yalnız gerçek yorum varken.
- Bültende çift onay + abonelikten çıkma bağlantısı (CAN-SPAM).
- "Made in USA" gibi iddialar ancak doğruysa.
### Next.js 16 farkları
- Middleware adı proxy.ts. tailwind.config YOK; token'lar app/globals.css @theme içinde.
- Görsellerde çalışma anı optimizasyonu kapalı; CDN boyutları yüklemede üretilir.

## Çalıştırma
npm ci && npx prisma generate && npm run dev   # :3000 (prod'da :3100, staging :3101)
Deploy (sunucuda, ajan ÇALIŞTIRMAZ — komutu kullanıcıya verir): cd /opt/eapricot && ./deploy/deploy.sh

## Yorum yazma kuralı
Yorumlar "ne yapıyor"u değil, "neden böyle ve değiştirirsen ne kırılır"ı anlatır.
Bir kuralı kodlarken onu doğuran hatayı da yaz (Ortak Doku geleneği).
```
