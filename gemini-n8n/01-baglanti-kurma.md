# n8n ↔ Gemini: Bağlantı Kurma

Bu belge, Google AI Studio üzerinden Gemini API key alıp n8n'de kullanmayı anlatır.

## Adım 1 — Google AI Studio'dan API Key al

1. [Google AI Studio](https://aistudio.google.com/) adresine git, Google hesabınla giriş yap
2. Sol menüden ya da ana sayfadan **Get API Key** butonuna tıkla
3. **Create API Key** de
4. Bir Google Cloud projesi seçmeni isteyebilir:
   - Var olan bir proje seçebilirsin (Gmail için oluşturduğun projeyi de kullanabilirsin)
   - Ya da "Create API key in new project" ile otomatik yeni proje açtırabilirsin
5. Oluşan API key'i kopyala ve güvenli bir yere kaydet

> Google Cloud Console'daki gibi OAuth2 consent screen, redirect URI gibi karmaşık bir süreç yok — Gemini API key doğrudan kullanılabilir bir "API key" tipi, PAT'e benzer basitlikte.

## Adım 2 — n8n'de credential oluştur

1. n8n → **Credentials → New**
2. Arama kutusuna `Gemini` yaz → **Google Gemini(PaLM) Api** credential tipini seç
3. **API Key** alanına Adım 1'de aldığın key'i yapıştır
4. Kaydet, adını `Gemini API` gibi bir şey koy

## Adım 3 — Test et

1. Yeni bir workflow'da **Google Gemini** node'unu ekle (Model / Chat node olarak da geçebilir, n8n sürümüne göre isim değişebilir)
2. Credential olarak oluşturduğun Gemini API'yi seç
3. **Model** seç (örn. `gemini-1.5-flash`, `gemini-1.5-pro` — hesabında hangi modeller aktifse)
4. Basit bir prompt yaz (örn. "Merhaba, bağlantı testi") ve node'u çalıştır
5. Dönen cevabı kontrol et

## Dikkat Edilecek Noktalar

- **Ücretsiz kota:** AI Studio üzerinden alınan key'lerde günlük/dakikalık istek limitleri var (free tier), sık tetiklenen otomasyonlarda rate limit'e takılabilirsin.
- **Model isimleri değişebilir:** Google zaman zaman yeni model versiyonları çıkarıyor (`gemini-2.0-...` gibi), n8n node'undaki model listesini güncel tutmak için ara sıra kontrol etmek iyi olur.
