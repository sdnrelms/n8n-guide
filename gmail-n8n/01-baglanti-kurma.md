# n8n ↔ Gmail: Bağlantı Kurma

Bu belge, n8n üzerinden Gmail node'unu kullanabilmek için Google Cloud üzerinde OAuth2 kimlik doğrulama kurulumunu anlatır.

## Neden OAuth2?

Gmail, PAT gibi basit bir token sistemi kullanmaz. Google, güvenlik gereği **OAuth2** ister — yani n8n'in senin adına Gmail'e erişebilmesi için Google'ın kendi yetkilendirme akışından geçmen gerekir.

## Adım 1 — Google Cloud'da proje oluştur

1. [Google Cloud Console](https://console.cloud.google.com/) adresine git
2. Üstteki proje seçiciden **New Project** ile yeni bir proje oluştur (örn. `n8n-integration`)
3. Proje oluşunca onu seçili hale getir

## Adım 2 — Gmail API'yi etkinleştir

1. Sol menüden **APIs & Services → Library**
2. Arama kutusuna `Gmail API` yaz
3. Gmail API'ye tıkla → **Enable**

## Adım 3 — OAuth consent screen (izin ekranı) ayarla

1. **APIs & Services → OAuth consent screen**
2. User Type: **External** seç (kişisel Gmail hesabı kullanıyorsan)
3. Uygulama adı, destek e-postası gibi zorunlu alanları doldur
4. **Scopes** adımında Gmail ile ilgili scope'ları ekle (örn. `.../auth/gmail.send`, `.../auth/gmail.readonly` — ne yapacaksan ona göre)
5. **Test users** adımında kendi Gmail adresini ekle (uygulama henüz "yayında" olmadığı için sadece test kullanıcıları erişebilir)

## Adım 4 — OAuth2 Client ID oluştur

1. **APIs & Services → Credentials → + Create Credentials → OAuth client ID**
2. Application type: **Web application**
3. **Authorized redirect URIs** kısmına n8n'in verdiği callback URL'i ekle
   - n8n'de Gmail credential oluştururken bu URL otomatik gösterilir, oradan kopyala
   - Genelde şu formattadır: `https://{n8n-domain}/rest/oauth2-credential/callback`
4. **Create** → çıkan **Client ID** ve **Client Secret**'i kopyala

## Adım 5 — n8n'de Gmail credential oluştur

1. n8n → **Credentials → New → Gmail OAuth2 API**
2. **Client ID** ve **Client Secret**'i yapıştır
3. **Connect my account** butonuna tıkla
4. Açılan Google ekranından hesabını seç, izinleri onayla
5. Bağlantı başarılıysa credential yeşil/aktif görünür

## Adım 6 — Test et

1. Yeni bir workflow'da **Gmail** node ekle
2. Operation: **Send Email** seç (ya da ihtiyacın neyse)
3. Credential olarak az önce oluşturduğun Gmail OAuth2 hesabını seç
4. Test amaçlı kendine bir e-posta gönder, gelen kutusunu kontrol et

## Dikkat Edilecek Noktalar

- **Test mode kısıtı:** Consent screen "Testing" modundaysa sadece eklediğin test kullanıcıları bağlanabilir. Ekip içinde başka biri de kullanacaksa onun e-postasını da test users listesine eklemen gerekir, ya da uygulamayı "Production"a almak için Google'ın doğrulama sürecine girmen gerekebilir.
- **Scope seçimi:** Sadece ihtiyacın olan scope'u seç (örn. sadece gönderim yapacaksan `gmail.send` yeterli, `gmail.readonly` gibi fazladan izin istemene gerek yok).
- **Refresh token süresi:** Test modundaki OAuth uygulamalarında refresh token belirli bir süre sonra (genelde 7 gün) geçersiz olabilir, bağlantı kopukmuş gibi görünürse yeniden "Connect my account" yapman gerekebilir.

