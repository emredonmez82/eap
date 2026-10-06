# Eapricot — Veri modeli (Prisma taslağı)

**Kurallar** (Ortak Doku'dan alınan dersler):
- **Para tamsayı kuruş** (`*Cents Int`). Ondalık para birimi hesabı yapılmaz.
- **Sipariş anında fiyat dondurulur** (`pricingSnapshot`). Sonradan ürün fiyatı değişse de siparişin tutarı değişmez.
- **Webhook tekilliği tabloyla yapılır** (`StripeEvent`). Aynı olay iki kez işlenmez.
- Silme yerine çoğunlukla `status`/`active` kullanılır; sipariş geçmişindeki ürün kaybolmaz.

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

// ─── Yönetim ────────────────────────────────────────────────────────────────
enum AdminRole { OWNER EDITOR OPERATIONS }

model AdminUser {
  id           String    @id @default(cuid())
  email        String    @unique
  name         String
  passwordHash String
  role         AdminRole @default(EDITOR)
  totpSecret   String?   // şifreli saklanır (OWNER için zorunlu önerilir)
  active       Boolean   @default(true)
  lastLoginAt  DateTime?
  createdAt    DateTime  @default(now())
}

model AdminAudit {               // kim neyi değiştirdi (fiyat, durum, iade…)
  id        String   @id @default(cuid())
  adminId   String
  action    String   // "product.update", "order.refund"…
  entity    String
  entityId  String
  meta      Json?
  createdAt DateTime @default(now())
  @@index([entity, entityId])
}

// ─── Katalog ────────────────────────────────────────────────────────────────
model Category {                 // WOOD_DECOR · WALL_DECALS · WALLPAPER
  id          String    @id @default(cuid())
  slug        String    @unique // wood-decor, wall-decals, wallpaper
  name        String
  tagline     String?   // "Add warmth, texture and personality to your walls."
  imageId     String?
  sort        Int       @default(0)
  active      Boolean   @default(true)
  products    Product[]
}

enum CollectionKind { SPACE MOMENT }

model Collection {               // For your home · For a new home · For someone special · Just because
  id          String              @id @default(cuid())
  slug        String              @unique
  name        String
  description String?
  kind        CollectionKind      @default(MOMENT)
  imageId     String?
  sort        Int                 @default(0)
  active      Boolean             @default(true)
  products    ProductCollection[]
}

enum ProductStatus { DRAFT ACTIVE ARCHIVED }
enum PricingMode   { FIXED PER_SQFT }   // PER_SQFT: duvar kağıdı / serbest ölçü decal

model Product {
  id               String                 @id @default(cuid())
  slug             String                 @unique
  title            String
  subtitle         String?
  description      String                 // markdown
  categoryId       String
  category         Category               @relation(fields: [categoryId], references: [id])
  status           ProductStatus          @default(DRAFT)
  pricingMode      PricingMode            @default(FIXED)
  sqftPriceCents   Int?                   // PER_SQFT için
  minWidthIn       Int?
  maxWidthIn       Int?
  minHeightIn      Int?
  maxHeightIn      Int?
  leadTimeDays     Int                    @default(5)   // üretim süresi (ekranda tahmini teslim)
  isPersonalizable Boolean                @default(false)
  previewTemplate  Json?                  // { baseMediaId, slots:[{fieldKey,x,y,w,h,font,size,color,align}] }
  seoTitle         String?
  seoDescription   String?
  ortakDokuRef     String?                // Faz 2: Ortak Doku ürün eşlemesi
  variants         Variant[]
  fields           PersonalizationField[]
  images           ProductImage[]
  collections      ProductCollection[]
  reviews          Review[]
  createdAt        DateTime               @default(now())
  updatedAt        DateTime               @updatedAt
  @@index([categoryId, status])
}

model Variant {                  // ölçü / kaplama vb. sabit seçenekler
  id         String  @id @default(cuid())
  productId  String
  product    Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  name       String  // "12 × 18 in · Natural"
  sku        String  @unique
  priceCents Int
  options    Json    // { size: "12x18", finish: "natural" }
  weightOz   Int?
  active     Boolean @default(true)
  sort       Int     @default(0)
}

enum FieldType { TEXT TEXTAREA DATE SELECT FONT COLOR }

model PersonalizationField {
  id          String    @id @default(cuid())
  productId   String
  product     Product   @relation(fields: [productId], references: [id], onDelete: Cascade)
  key         String    // "name", "date", "message"
  label       String    // "Family name"
  type        FieldType
  required    Boolean   @default(true)
  maxLength   Int?
  pattern     String?   // izinli karakterler (regex) — sunucu doğrular
  options     Json?     // SELECT/FONT/COLOR seçenekleri
  placeholder String?
  helpText    String?
  sort        Int       @default(0)
  @@unique([productId, key])
}

model ProductCollection {
  productId    String
  collectionId String
  product      Product    @relation(fields: [productId], references: [id], onDelete: Cascade)
  collection   Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  sort         Int        @default(0)
  @@id([productId, collectionId])
}

// ─── Medya ──────────────────────────────────────────────────────────────────
model Media {
  id         String   @id @default(cuid())
  key        String   @unique // R2 orijinal anahtarı
  alt        String   // ZORUNLU (erişilebilirlik + SEO)
  width      Int
  height     Int
  mime       String
  sizeBytes  Int
  variants   Json     // { avif: {480: url, 960: url…}, webp: {…} }
  createdAt  DateTime @default(now())
}

enum ImageRole { GALLERY MOCKUP_BASE LIFESTYLE }

model ProductImage {
  productId String
  mediaId   String
  role      ImageRole @default(GALLERY)
  sort      Int       @default(0)
  product   Product   @relation(fields: [productId], references: [id], onDelete: Cascade)
  @@id([productId, mediaId])
}

// ─── Müşteri & sipariş ──────────────────────────────────────────────────────
model Customer {                 // MVP: misafir ödeme; hesap girişi Faz 2
  id               String   @id @default(cuid())
  email            String   @unique
  name             String?
  phone            String?
  marketingConsent Boolean  @default(false)
  orders           Order[]
  createdAt        DateTime @default(now())
}

enum OrderStatus { PENDING_PAYMENT PAID IN_PRODUCTION SHIPPED DELIVERED CANCELLED REFUNDED }

model Order {
  id                    String      @id @default(cuid())
  number                String      @unique // "EA-10023" (müşteriye gösterilen)
  customerId            String?
  customer              Customer?   @relation(fields: [customerId], references: [id])
  email                 String
  status                OrderStatus @default(PENDING_PAYMENT)
  subtotalCents         Int
  shippingCents         Int
  taxCents              Int         @default(0)
  discountCents         Int         @default(0)
  totalCents            Int
  currency              String      @default("usd")
  pricingSnapshot       Json        // fiyat kuralları + ürün fiyatları o anki hâliyle
  stripeSessionId       String?     @unique
  stripePaymentIntentId String?     @unique
  shippingAddress       Json?       // Stripe'tan gelen adres
  carrier               String?
  trackingNumber        String?
  ortakDokuOrderRef     String?     // Faz 2
  note                  String?     // müşteri notu
  paidAt                DateTime?
  shippedAt             DateTime?
  items                 OrderItem[]
  events                OrderEvent[]
  createdAt             DateTime    @default(now())
  updatedAt             DateTime    @updatedAt
  @@index([status, createdAt])
}

model OrderItem {
  id              String  @id @default(cuid())
  orderId         String
  order           Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId       String
  variantId       String?
  title           String  // o anki ürün adı (dondurulmuş)
  quantity        Int
  widthIn         Int?
  heightIn        Int?
  personalization Json?   // { name: "The Wilson Family", date: "2026" }
  proofMediaId    String? // ödeme sonrası üretilen prova PNG
  unitPriceCents  Int
  lineTotalCents  Int
}

model OrderEvent {               // durum geçmişi + e-posta kayıtları
  id        String   @id @default(cuid())
  orderId   String
  order     Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)
  type      String   // "paid", "proof.generated", "email.confirmation", "shipped"
  data      Json?
  createdAt DateTime @default(now())
}

model StripeEvent {              // webhook tekilliği
  id          String   @id     // evt_…
  type        String
  processedAt DateTime @default(now())
}

// ─── İçerik ─────────────────────────────────────────────────────────────────
model ContentBlock {             // ana sayfa ve sabit sayfa bölümleri — SITE.md'deki her bölüm bir anahtar
  key       String   @id         // "home.hero", "home.personalization", "about.story"…
  data      Json                 // { title, subtitle, body, ctaLabel, ctaHref, imageId… } — zod şemasıyla doğrulanır
  updatedBy String?
  updatedAt DateTime @updatedAt
}

enum FaqGroup { ORDERING PERSONALIZED SHIPPING RETURNS }

model FaqItem {
  id       String   @id @default(cuid())
  group    FaqGroup
  question String
  answer   String
  sort     Int      @default(0)
  active   Boolean  @default(true)
}

enum ReviewSource { SITE ETSY MANUAL }

model Review {                   // YALNIZ gerçek ve izinli yorum (FTC)
  id         String       @id @default(cuid())
  productId  String?
  product    Product?     @relation(fields: [productId], references: [id])
  authorName String       // "Sarah L." biçimi
  rating     Int          // 1–5
  body       String
  source     ReviewSource
  consent    Boolean      // yayın izni alındı mı
  approved   Boolean      @default(false)
  createdAt  DateTime     @default(now())
}

enum SubscriberStatus { PENDING CONFIRMED UNSUBSCRIBED }

model NewsletterSubscriber {
  id           String           @id @default(cuid())
  email        String           @unique
  status       SubscriberStatus @default(PENDING)
  source       String?          // "footer", "checkout"
  consentAt    DateTime?
  confirmToken String?          @unique
  createdAt    DateTime         @default(now())
}

model ContactMessage {
  id          String   @id @default(cuid())
  name        String
  email       String
  orderNumber String?
  message     String
  status      String   @default("NEW") // NEW · ANSWERED
  createdAt   DateTime @default(now())
}

model Setting {                  // kargo ücretleri, ücretsiz kargo eşiği, vergi açık/kapalı, duyuru çubuğu…
  key       String   @id
  value     Json
  updatedAt DateTime @updatedAt
}
```

## Fiyat kuralı (`lib/pricing.ts`, tek kaynak)

- **FIXED:** `variant.priceCents × adet`.
- **PER_SQFT** (duvar kağıdı / serbest ölçü decal): `ceil(genişlik_in × yükseklik_in / 144 × sqftPriceCents)`.
  - Ölçü ürünün min/max aralığında olmalı.
  - Ölçü inç olarak alınır, gösterim ft/in.
- **Kargo:** `Setting["shipping"]` → `{ flatCents, freeOverCents }`.
- **Vergi:** Stripe Tax açıksa Stripe hesaplar, sonuç webhook'tan `taxCents` alanına yazılır.
- Ürün kartındaki fiyat, sepet fiyatı ve Stripe satırı **aynı fonksiyondan** türer. Ortak Doku'daki `baseUnitRate()` dersi: aynı kuralı ikinci bir yere yazma.
