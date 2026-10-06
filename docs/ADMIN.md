# Eapricot — Admin paneli (MVP)

`/admin` altında, aynı Next.js uygulamasının içinde çalışır. Arama motoruna kapalıdır (`noindex`). Tüketiciden tamamen ayrı bir oturum kullanır.

## Roller

| Rol | Yetki |
|---|---|
| **OWNER** | Her şey: ayarlar, kullanıcılar, **iade** |
| **EDITOR** | Ürün, kategori, koleksiyon, içerik, medya, SSS, yorum, bülten, mesajlar. Siparişleri yalnız okur |
| **OPERATIONS** | Siparişler: durum, takip no, üretim föyü, CSV. İade yapamaz |

Her yazma işlemi `AdminAudit`'e düşer: kim, ne zaman, ne değiştirdi.

## Ekranlar

### Özet
- Bugün / 7 gün için sipariş sayısı ve ciro.
- Bekleyen işler: **ödenmiş ama üretime alınmamış**, üretimde, kargo bekleyen.
- Son mesajlar, son aboneler.

### Siparişler
- **Liste:** Durum süzgeci (Ödendi / Üretimde / Kargoda / Teslim / İptal / İade), arama (sipariş no, e-posta, ad), tarih.
- **Detay:**
  - Kalemler: ürün, varyant/ölçü, adet, fiyat. **Kişiselleştirme değerleri kopyalanabilir metin olarak** + prova görseli.
  - Müşteri, adres, ödeme (Stripe bağlantısı), vergi/kargo dökümü.
  - Durum geçmişi (`OrderEvent`) ve gönderilen e-postalar.
- **Eylemler:**
  - **Üretime al:** Durumu `IN_PRODUCTION` yapar. MVP'de sipariş Ortak Doku'da elle açılır.
  - **Üretim föyü (PDF):** Kalemler, ölçüler, kişiselleştirme, prova, adres ve sipariş no barkodu.
  - **Takip no ekle:** Taşıyıcı + numara → `SHIPPED` + müşteriye kargo e-postası.
  - **İade (OWNER):** Tam ya da kısmi. Stripe'ta iade açar; durum webhook ile güncellenir.
  - **Not ekle**, **CSV dışa aktar**.
  - *Faz 2:* **Ortak Doku'ya gönder** (Kanal API), Ortak Doku'daki durum ve takip no otomatik gelir.

### Ürünler
- **Liste:** Durum (Taslak / Yayında / Arşiv), kategori, arama.
- **Düzenleme sekmeleri:**
  1. **Bilgi:** Başlık, alt başlık, açıklama (markdown), kategori, koleksiyonlar, üretim süresi.
  2. **Fiyat:**
     - Sabit fiyatlı ürün → **Varyantlar** tablosu (ad, SKU, fiyat, seçenekler, aktif).
     - Ölçüye göre fiyatlı ürün → ft² fiyatı + min/max genişlik/yükseklik.
     - **Fiyat hesabını gösteren önizleme:** "48×96 in = 32 ft² → $X". Ekrandaki hesap `lib/pricing.ts`'ten gelir; aynı kural ikinci kez yazılmaz.
  3. **Kişiselleştirme:**
     - **Alanlar:** Anahtar, etiket, tür, zorunlu, en fazla uzunluk, izinli karakter, seçenekler.
     - **Önizleme şablonu:** Taban görsel seçilir, her alan için metin kutusu yerleştirilir (sürükle ya da x/y/genişlik/yükseklik), font, boyut, renk ve hizalama ayarlanır.
     - **Canlı deneme:** Örnek değerlerle önizleme.
  4. **Görseller:** Medya kütüphanesinden seç, sürükle-sırala, rol (galeri / mockup tabanı / yaşam alanı).
  5. **SEO:** Başlık, açıklama, URL (slug). Slug değişirse eski adres yönlendirilir.
- **Yayın kontrolü:** "Yayınla" denince en az 1 görsel, alt metinler ve fiyat zorunlu. Kişiselleştirilebilir ürünse şablon zorunlu.

### Kategoriler & Koleksiyonlar
Ad, slug, kısa metin, görsel, sıra, aktif. Koleksiyona ürün ekleme/sıralama.

### İçerik (ana sayfa ve sabit sayfalar)
- `SITE.md`'deki her bölüm (`home.hero`, `home.personalization`, …) bir **form**:
  - Başlık, metin, düğme etiketi/adresi, görsel.
  - Bölümü göster/gizle.
- Kaydedince ilgili sayfa yeniden üretilir (`revalidateTag`). Önizleme düğmesi sayfayı yeni sekmede açar.
- Duyuru çubuğu (ör. "Free US shipping over $75") → `Setting`.

### Medya kütüphanesi
- Çoklu yükleme: tarayıcıdan doğrudan R2'ye; ilerleme çubuğu.
- Yükleme sonrası boyutlandırma durumu görünür.
- Izgara görünüm, arama (alt metin), **alt metin düzenleme (zorunlu)**, kullanıldığı yerler.
- Kullanılan görsel silinemez.

### SSS
4 grup (Ordering / Personalized / Shipping / Returns). Soru, cevap, sıra, aktif.

### Yorumlar
- Ekle (kaynak: site / Etsy / elle), puan, metin, yazar adı biçimi ("Sarah L.").
- **Yayın izni alındı** kutusu **zorunlu**; onayla/gizle.
- Ürüne bağlama isteğe bağlı.

### Bülten aboneleri
Liste (durum, kaynak, tarih), CSV dışa aktar. Silme talebinde kayıt silinir (CCPA).

### Mesajlar
İletişim formu kayıtları; "yanıtlandı" işareti; sipariş no varsa siparişe bağlantı.

### Ayarlar (OWNER)
- **Kargo:** Sabit ücret, ücretsiz kargo eşiği.
- **Vergi:** Stripe Tax açık/kapalı.
- İletişim bildirim e-postası, sosyal linkler, duyuru çubuğu.
- Bakım modu: mağaza "yakında" sayfası gösterir, admin çalışmaya devam eder.

### Kullanıcılar (OWNER)
Admin ekle/pasifleştir, rol değiştir, TOTP sıfırla.

## Bilerek MVP dışında bırakılanlar
- Stok takibi (ürünler siparişe göre üretiliyor).
- Kupon/indirim.
- Müşteri hesabı.
- Çoklu dil ve para birimi.
- Gelişmiş raporlar.
- Etsy/Amazon listing senkronu.

Ayrıntı: YOL-HARITASI.md.
