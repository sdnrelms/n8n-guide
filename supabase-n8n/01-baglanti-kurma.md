# n8n ↔ Supabase: Bağlantı Kurma

Bu belge, n8n üzerinden Supabase'e bağlanmak için gerekli kimlik doğrulama kurulumunu anlatır.

## Adım 1 — Supabase'de proje oluştur (yoksa)

1. [Supabase](https://supabase.com/) hesabına giriş yap
2. **New Project** ile bir proje oluştur (org, isim, database şifresi, bölge seç)
3. Proje kurulumu tamamlanana kadar bekle (birkaç dakika sürebilir)

## Adım 2 — Bağlantı bilgilerini bul

Supabase'de n8n bağlantısı için genelde iki yöntem var; hangisini kullanacağın ihtiyacına göre değişir.

**Yöntem A — Supabase API (REST/Data API) ile:**

1. Proje içinde sol menüden **Project Settings → API**
2. Şu iki bilgiyi kopyala:
   - **Project URL** (örn. `https://xxxxx.supabase.co`)
   - **anon public key** veya **service_role key** (service_role, RLS'i (Row Level Security) bypass eder — n8n gibi backend otomasyonları için genelde service_role kullanılır, ama dikkatli saklanmalı)

**Yöntem B — Doğrudan Postgres bağlantısı ile:**

1. **Project Settings → Database**
2. **Connection string** kısmından bağlantı bilgilerini al (host, port, database, user, password)
3. Bu yöntem n8n'in **Postgres** node'uyla kullanılır, Supabase'e özel bir node değildir — Supabase sadece yönetilen bir Postgres olduğu için doğrudan çalışır

## Adım 3 — n8n'de credential oluştur

**Supabase node kullanıyorsan:**
1. n8n → **Credentials → New → Supabase API**
2. **Host** alanına Project URL'i yapıştır
3. **Service Role Secret** alanına service_role key'i yapıştır
4. Kaydet

**Postgres node kullanıyorsan:**
1. n8n → **Credentials → New → Postgres**
2. Host, database, user, password, port bilgilerini Adım 2'deki connection string'den doldur
3. SSL genelde **Enable** olmalı (Supabase SSL zorunlu tutuyor)
4. Kaydet

## Adım 4 — Test et

1. Yeni bir workflow'da **Supabase** (ya da **Postgres**) node ekle
2. Credential olarak oluşturduğun hesabı seç
3. Operation: **Get Rows** / **Select** gibi basit bir okuma işlemi seç
4. Bir tablo adı belirt, çalıştır
5. Veri döndüğünü kontrol et

## Dikkat Edilecek Noktalar

- **service_role key riski:** Bu key tüm RLS kurallarını bypass eder, tam yetkilidir. Repoya kesinlikle commit edilmemeli, sadece n8n credential içinde saklanmalı.
- **anon key ile sınırlı erişim:** Eğer RLS politikalarına uygun, kısıtlı erişim istiyorsan anon key + RLS kombinasyonu daha güvenlidir ama bu durumda tablo politikalarını (policies) doğru tanımlamış olman gerekir.
- **Hangi node'u seçmeli:** Basit CRUD işlemleri için Supabase node yeterli ve daha az konfigürasyon ister. Karmaşık SQL sorguları (join, aggregate vs.) gerekiyorsa Postgres node ile ham SQL yazmak daha esnek olur.
- **IP kısıtlaması:** Supabase projende bir "Network Restrictions" ayarı varsa, n8n'in çalıştığı sunucunun IP'sinin izinli listede olduğundan emin ol.
