# n8n ↔ GitHub: Bağlantı Kurma

Bu belge, n8n üzerinden GitHub'a Personal Access Token (PAT) ile bağlanma kurulumunu anlatır.

## Adım 1 — GitHub'da PAT oluştur

1. GitHub'a giriş yap → sağ üstte profil ikonu → **Settings**
2. Sol menüden en altta **Developer settings**
3. **Personal access tokens** → **Tokens (classic)** ya da **Fine-grained tokens** (ikisi de kullanılabilir, aşağıda ayrım var)

**Classic token ile gidersen (benim kullandığım):**
1. **Generate new token (classic)**
2. Not (isim) ver: `n8n-integration`
3. Expiration seç (süresiz bırakmamak güvenlik açısından daha iyi)
4. **Scopes** kısmında ihtiyacına göre işaretle:
   - `repo` — repo okuma/yazma (private repo'larla çalışacaksan gerekli)
   - `workflow` — GitHub Actions workflow dosyalarını değiştirecekse
5. **Generate token** → çıkan token'ı kopyala, bir daha gösterilmez

**Fine-grained token ile gidersen:**
1. **Generate new token**
2. Hangi repository'lere erişeceğini seç (tümü yerine sadece ilgili repo'yu seçmek daha güvenli)
3. **Permissions** kısmında sadece ihtiyacın olan izinleri ver (örn. Contents: Read & Write, Issues: Read & Write)
4. **Generate token** → kopyala

## Adım 2 — n8n'de credential oluştur

1. n8n → **Credentials → New → GitHub API**
2. **Access Token** alanına oluşturduğun PAT'i yapıştır
3. Kaydet, adını `GitHub PAT` gibi bir şey koy

> GitHub node'u genelde token'ı direkt kabul eder, Gmail/Sheets'teki gibi OAuth2 consent akışı gerekmez — Azure DevOps'a benzer şekilde tek adımlı.

## Adım 3 — Test et

1. Yeni bir workflow'da **GitHub** node ekle
2. Credential olarak oluşturduğun PAT'i seç
3. Resource: **Issue** ya da **Repository** seç, Operation: **Get** / **List** gibi basit bir okuma işlemi dene
4. Owner (kullanıcı/organizasyon adı) ve Repository adını gir
5. Çalıştır, veri döndüğünü kontrol et

## Dikkat Edilecek Noktalar

- **Token kapsamı:** Sadece ihtiyacın olan repo'lara ve izinlere erişim ver — özellikle fine-grained token'da "tüm repository'ler" seçmek yerine spesifik repo seçmek daha güvenli.
- **Expiration:** Token'a süre koyduysan, süre dolduğunda workflow'lar sessizce başarısız olabilir; takvime hatırlatma koymak faydalı olabilir.
- **Rate limit:** GitHub API'de saatlik istek limiti var (authenticated istekler için genelde 5000/saat), çok sık tetiklenen workflow'larda dikkat edilmeli.
- **Organizasyon repo'ları:** Eğer repo bir organizasyona aitse, token'ın o organizasyonun SSO/erişim politikalarına göre yetkilendirilmesi (authorize) gerekebilir.
