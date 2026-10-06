# Eapricot — Site (sayfalar, ana sayfa bölümleri, tasarım)

Kaynaklar: Aziz/Emre'nin metin dokümanı (2026-10-06) ve ana sayfa maketi. **Metin ile maket çelişirse metin esas alınır** (bkz. README "karar bekliyor").
Bölüm metinleri admin'den değiştirilebilir (`ContentBlock` anahtarı yanında yazılı). Aşağıdaki metinler başlangıç verisidir (seed).

## Sayfa haritası

| Yol | Sayfa | Önbellek |
|---|---|---|
| `/` | Ana sayfa (10 bölüm, aşağıda) | ISR, `home` etiketi |
| `/shop` | Tüm ürünler (kategori süzgeciyle) | ISR |
| `/shop/wood-decor` · `/shop/wall-decals` · `/shop/wallpaper` | Kategori | ISR |
| `/collections/[slug]` | For your home · For a new home · For someone special · Just because | ISR |
| `/products/[slug]` | Ürün: galeri, varyant/ölçü, **kişiselleştirici + canlı önizleme**, üretim süresi, kargo bilgisi, SSS kısa | ISR |
| `/cart` | Sepet (sunucu fiyatıyla) | dinamik |
| `/checkout/success` | Teşekkürler + sipariş no + prova görselleri | dinamik |
| `/our-story` | Hikâye (bölüm 6'nın uzun hâli) | ISR |
| `/personalization` | Kişiselleştirme nasıl çalışır | ISR |
| `/faq` | SSS (4 grup) | ISR |
| `/contact` | İletişim formu (düz e-posta yazılmaz) | statik |
| `/policies/shipping` · `/returns` · `/privacy` · `/terms` | Yasal | ISR |

**Üst menü:** Shop ▾ (Wood Décor · Wall Decals · Wallpaper) · Personalized · About · FAQ · 🔍 · 🛒
**Not:** Maketteki kullanıcı ikonu MVP'de yok; hesap girişi Faz 2'de.

---

## Ana sayfa akışı

Hero → Shop by Category → Make It Yours → Shop by Moment → Why Eapricot? → Our Story → Reviews → FAQ → Newsletter → Footer

### 1. Hero — `home.hero`
- **Başlık:** *Thoughtful things.* / *Made for you.* (ikinci satır italik)
- **Metin:** Home décor and personalized pieces designed for your space, your story, and the moments that matter.
- **Düğme:** SHOP NOW → `/shop`
- **Görsel:** Geniş oturma odası. Masaüstü 2400×1100, mobil 1080×1350 ayrı kadraj. `priority` ile yüklenir.

### 2. Shop by Category — `home.categories`
- **Başlık:** *Made for your space.*
- **Metin:** Thoughtfully designed pieces to make your home feel a little more like yours.
- **Kartlar** (veri `Category` tablosundan; görsel 1200×900):
  - **WOOD DÉCOR**: Add warmth, texture and personality to your walls.
  - **WALL DECALS**: Simple details that can completely change a space.
  - **WALLPAPER**: Bring pattern, color and character to your walls.
- **Bağlantı:** EXPLORE ALL → `/shop`

### 3. Personalization — `home.personalization`
- **Üst etiket:** PERSONALIZED
- **Başlık:** *Make it yours.*
- **Alt başlık:** *Personalized pieces, designed by you.*
- **Metin:** Add a name, date, message or meaningful detail to create something that feels truly personal. Choose your product, customize the details, preview your design and make it yours.
- **Adımlar:**
  - **01 — CHOOSE:** Select the product you want to personalize.
  - **02 — CUSTOMIZE:** Add names, dates, messages or available design options.
  - **03 — PREVIEW & ORDER:** Review your details, place your order and we'll create it just for you.
- **Düğme:** START PERSONALIZING → `/collections/personalized` (ya da `/personalization`)
- **Görsel:** Maketteki dizüstü bilgisayar sahnesi. Yayına çıkınca kendi kişiselleştiricimizin gerçek ekran görüntüsü kullanılmalı (sahte arayüz gösterilmez).
- **Not:** Maketteki el yazısı not "Customize in minutes" süs olarak kullanılabilir.

### 4. Shop by Moment — `home.moments`
- **Başlık:** *Made for the moments that matter.*
- **Metin:** From everyday spaces to meaningful occasions, find something designed to fit the moment.
- **Kartlar** (veri `Collection` tablosundan, kind=MOMENT; görsel 1000×750):
  - **FOR YOUR HOME**: Pieces that bring personality to the spaces you live in.
  - **FOR A NEW HOME**: Thoughtful details for a new beginning.
  - **FOR SOMEONE SPECIAL**: Personal pieces made to mean a little more.
  - **JUST BECAUSE**: Because you don't always need a reason.
- **Makette olup metinde olmayanlar:** Ara banttaki *"Beautiful details. For every space."* + EXPLORE WALLPAPERS (`home.banner`) **isteğe bağlı** bölüm olarak tutulur; admin açıp kapatır.

### 5. Why Eapricot? — `home.why`
Dört ikonlu sütun:
- **Thoughtful design:** Every piece starts with an idea worth bringing to life.
- **Personal by nature:** Create something with a name, message, date or detail that matters.
- **Made for real life:** From everyday home décor to gifts for meaningful moments.
- **Designed to feel like yours:** Original designs with personalization options that make each piece more personal.

### 6. Our Story — `home.story` (uzun hâli `/our-story`)
- **Başlık:** *A little more personal.*
- **Metin:**
  > Eapricot was created with a simple idea: the things we live with and give to others should feel personal.
  > We bring together home décor, wall art, personalized pieces and thoughtful gifts designed for everyday spaces and meaningful moments.
  > Some pieces are made to add character to your home. Others are made to celebrate a person, a memory or a moment.
  > Whatever you choose, our goal is simple — to make it feel a little more like yours.
  > *Welcome to Eapricot.*
- **Düğme:** DISCOVER OUR STORY → `/our-story`
- **Alt maddeler:**
  - **DESIGNED WITH INTENTION:** Thoughtful designs made to fit your space and your story.
  - **PERSONAL BY DESIGN:** Personalization makes ordinary pieces feel more meaningful.
  - **MADE TO BRING JOY:** For your home, for someone you love, or simply for yourself.
- **Süs:** El yazısı not "More than just décor" + kalp.

### 7. Reviews — `home.reviews`
- **Başlık:** *Made for real homes.*
- **Metin:** See what our customers are saying about the pieces they've made their own.
- Üç yorum kartı `Review` tablosundan gelir (`approved && consent`).
- **Kural:** Onaylı gerçek yorum sayısı **3'ün altındaysa bölüm hiç görünmez.** Maketteki "Sarah L. / Emily R. / Jessica M." örnek metindir, **yayına çıkmaz** (FTC sahte yorum yasağı).
- **Bağlantı:** READ MORE REVIEWS → (Faz 2 yorum sayfası)

### 8. FAQ — `FaqItem` tablosu (ana sayfada ilk 5 soru, tamamı `/faq`)
- **Başlık:** *FAQ*
- **Alt başlık:** Everything you need to know before you order.

**Ordering**
- *How do I place an order?* Choose your product, select the available options, add it to your cart and complete checkout. For personalized products, make sure all names, dates and other details are entered correctly before placing your order.
- *Can I change or cancel my order?* If you need to make a change or cancel your order, contact us as soon as possible. Because personalized products may enter production quickly, changes may only be possible before production begins.

**Personalized Products**
- *How does personalization work?* Simply enter your name, date, message or other requested details when customizing your product. You'll be able to review your information before placing your order.
- *Can I request a custom design?* Some products may offer custom design options. If you have a specific idea, contact us and we'll let you know what may be possible.
- *What happens if I enter the wrong name or information?* Please carefully review all personalized details before ordering. Once production begins, changes may not be possible.

**Shipping**
- *How long does shipping take?* Production and shipping times vary by product. You'll see the estimated processing and delivery information during the ordering process.
- *Where do you ship?* We currently ship within the United States.
- *How can I track my order?* Once your order ships, you'll receive tracking information when available.

**Returns**
- *Can I return a personalized product?* Because personalized products are made specifically for you, they may not be eligible for return. Please review the product and return policy before ordering.
- *What if my order arrives damaged or has an issue?* We're here to help. Contact us with your order details and photos of the issue, and we'll work with you to find a solution.

**Alt kutu:** *Didn't find what you're looking for? We're here to help.* · CONTACT US → `/contact`

### 9. Newsletter — `home.newsletter`
- **Başlık:** *A little inspiration, delivered.*
- **Metin:** Be the first to discover new designs, personalized pieces, home inspiration and special offers.
- **Form:** EMAIL ADDRESS · SUBSCRIBE →
- Çift onaylı abonelik (onay e-postası). Maketteki "Get 10% off your first order" metni **karar bekliyor**; indirim verilirse kupon sistemi gerekir (Faz 2).

### 10. Footer — `site.footer`
- **Eapricot** · *Thoughtful things. Made for you.*
- **Shop:** Wood Décor · Wall Decals · Wallpaper
- **Help:** FAQ · Shipping · Returns · Contact Us
- **About:** Our Story · Personalization
- Sosyal ikonlar (Instagram · Pinterest · TikTok · YouTube), adresler `Setting["social"]`'den.
- © {içinde bulunulan yıl} Eapricot. All rights reserved. (Makette sabit "2025" yazıyor; yıl otomatik olmalı.)
- Etsy'den söz edilirse Etsy marka notu gerekir; şu an site Etsy'yi anmıyor.

---

## Tasarım işaretleri (maket + logo; tasarımcı onaylasın)

| Token | Değer (yaklaşık) | Kullanım |
|---|---|---|
| `--bg` | `#F8F4EE` krem | Sayfa zemini |
| `--surface` | `#FFFFFF` / `#F3EEE6` | Kartlar, bantlar |
| `--ink` | `#2E2A26` | Metin |
| `--muted` | `#7A7067` | İkincil metin |
| `--olive` | `#66783C` | Logo yazısı, vurgu |
| `--sage-btn` | `#4E5B47` | Ana düğmeler |
| `--apricot` | `#F28C28` | Rozet, sepet sayacı, küçük vurgular |
| `--line` | `#E6DED3` | Ayraçlar |

- **Fontlar:**
  - Başlıklar: serif (ör. *Cormorant Garamond* / *Libre Caslon*). İkinci satırlarda italik.
  - Gövde: *Inter* / *DM Sans*.
  - El yazısı süs: *Caveat*.
  - Google Fonts, `next/font` ile kendi sunucumuzdan yüklenir.
- **Düğmeler:** Büyük harf, harf aralığı geniş, küçük köşe yarıçapı, sonda "→".
- **Bölüm başlıkları:** Sol başlık + ince çizgi + sağda "VIEW ALL →".
- **Mobil:** Kartlar yatay kaydırma (kategori, moment, yorum); FAQ akordeon.
- **Logo:** Gelen raster logo (mozaik kayısı + "eapricot"). SVG, şeffaf PNG, tek renk sürüm ve favicon (32/180/512) üretilmeli.

## Gereken görseller (MVP)

| Yer | Adet | Ölçü (en az) |
|---|---|---|
| Hero | 1 (+ mobil kadraj) | 2400×1100 · 1080×1350 |
| Kategori kartları | 3 | 1200×900 |
| Personalization sahnesi | 1 | 1600×1100 |
| Moment kartları | 4 | 1000×750 |
| Ara bant (isteğe bağlı) | 1 | 2000×900 |
| Our Story | 1–2 | 1600×1000 |
| Ürün galerisi | ürün başına 3–6 | 2000×2000 |
| Kişiselleştirme taban görseli (mockup) | kişiselleştirilebilir ürün başına 1 | 2000×2000, metin alanı boş |
| OG paylaşım görseli | 1 | 1200×630 |

Maketteki görseller üretilmiş (AI) görsellerse bunu ürünün gerçek fotoğrafıymış gibi sunmamak gerekir. Ürün sayfalarında **gerçek ürün fotoğrafı** kullanılmalı.
