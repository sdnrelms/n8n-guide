# n8n ↔ Azure DevOps: Bağlantı Kurma

Bu belge, n8n ile Azure DevOps arasında kimlik doğrulama ve webhook bağlantısının nasıl kurulacağını anlatır.

## Adım 1 — Azure DevOps'ta PAT (Personal Access Token) oluştur

1. Azure DevOps'a giriş yap → sağ üstte **User Settings** (dişli/profil ikonu) → **Personal Access Tokens**
2. **+ New Token**
3. İsim ver: `n8n-integration`
4. Organization ve expiration (geçerlilik süresi) seç
5. **Scopes** → Custom defined → **Work Items: Read & Write** işaretle
   - Sadece Read yeterli değil, çünkü PATCH ile Feature'a yazacağız
6. **Create** → oluşan token'ı hemen kopyala ve güvenli bir yere kaydet (bir daha gösterilmez)

## Adım 2 — n8n'de credential oluştur

1. n8n → **Credentials → New → Basic Auth**
2. **Username:** boş bırak
3. **Password:** PAT'i yapıştır
4. Kaydet, adını `Azure DevOps PAT` gibi bir şey koy

> Basic Auth, n8n'de encode işlemini otomatik yaptığı için Header Auth'a göre daha az hata payı bırakır.

## Adım 3 — n8n'de Webhook Trigger node kur

1. Yeni workflow aç, **Webhook** node ekle
2. HTTP Method: **POST**
3. Path: örn. `azure-pbi-update`
4. Node'u aç, **Test URL**'i kopyala
5. **"Listen for test event"** butonuna bas — n8n test isteği beklemeye başlar

## Adım 4 — Azure DevOps'ta Service Hook kur

1. **Project Settings → Service Hooks → +**
2. Service: **Web Hooks**
3. Trigger: **Work item updated** seç
4. Filtrelerde **Work item type = Product Backlog Item** seç
   - ⚠️ **Kritik:** Bu filtre olmazsa Feature güncellendiğinde de hook tekrar tetiklenir ve sonsuz döngüye girersiniz
5. Action adımında n8n'den kopyaladığın Test URL'i yapıştır
6. **Test** butonuna bas → n8n tarafında test isteğinin düştüğünü doğrula
7. Başarılıysa **Finish**
8. Aynı adımları **Work item created** event'i için de ayrı bir hook olarak tekrarla (yeni PBI eklendiğinde tetiklenmesi için)

> Test aşamasında Webhook node'un **Test URL**'i kullanılır. Workflow'u **Active** yaptığında **Production URL**'e geçmen ve Service Hook'ları bu yeni URL ile güncellemen gerekir.

## Kontrol Listesi

- [ ] PAT oluşturuldu ve güvenli yere kaydedildi
- [ ] n8n'de Basic Auth credential kuruldu
- [ ] Webhook node test modunda dinliyor
- [ ] Service Hook (Work item updated) kuruldu ve test edildi
- [ ] Service Hook (Work item created) kuruldu ve test edildi
- [ ] Workflow Active yapıldıktan sonra Production URL'e geçildi

Bağlantı tamamlandıktan sonraki adım için bkz. **[02-anlik-guncelleme.md](./02-anlik-guncelleme.md)**
