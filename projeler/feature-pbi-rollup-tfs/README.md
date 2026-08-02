# Feature-PBI Rollup (TFS/Azure DevOps Server)

Bir Feature'ın **Effort** ve **Remaining** alanlarını, altındaki PBI'ların (Product Backlog Item) Total SP değerlerinden otomatik hesaplayan bir n8n workflow'u. On-premises TFS/Azure DevOps Server üzerinde çalışacak şekilde kuruldu.

## Hedef

- **Effort** = Feature'a bağlı **tüm** PBI'ların Total SP toplamı (State'e bakılmaksızın) — sabit, tamamlanma durumundan etkilenmeyen toplam kapsam
- **Remaining** = Feature'a bağlı, **Done olmayan** PBI'ların Total SP toplamı — bir PBI Done olunca bu değerden düşer
- **Removed** durumundaki PBI'lar hem Effort'a hem Remaining'e hiç dahil edilmez
- Bir PBI Feature'dan **unlink edildiğinde** (bağlantısı koparıldığında, silinmese bile), eski Feature'ın Effort/Remaining'i güncellenir
- Yeni bir PBI eklendiğinde toplamlar otomatik güncellenir

**Temel mantık:** Her tetiklenmede tüm kardeş PBI'lar sıfırdan yeniden toplanır ve Feature'a yazılır. Ayrı "ekle/çıkar" state'i tutulmaz — bu hem daha güvenli hem daha az hataya açık.

## Kullanılan Alanlar (Reference Name'ler)

| Alan | Reference Name | Bulunduğu Yer |
|---|---|---|
| Total SP | `TAKIM_ADI.TOTAL_SP_ORIG` | PBI |
| Effort | `Microsoft.VSTS.Scheduling.Effort` | Feature |
| Remaining | `TAKIM_ADI.REMAINING_SP` | Feature |

> Reference name'ler şu endpoint ile doğrulanabilir/teyit edilebilir:
> ```
> GET https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/fields?api-version=7.1
> ```
> **Not:** Görünen isim (`Total SP`, `Effort`) ile reference name her zaman birebir örtüşmeyebilir; process customization'a göre farklı olabilir. Alan bir PBI'da boşsa, API cevabında o alan hiç görünmez (undefined döner).
## Ön Koşul: TFS Bağlantı Kurulumu

Genel PAT/credential kurulumu için [Azure DevOps bağlantı belgesine](../../azure-devops-n8n/01-baglanti-kurma.md) bakılabilir.

**Fark:** Bu proje **cloud Azure DevOps değil, on-prem TFS/Azure DevOps Server** kullanıyor. URL formatı farklı:

| | Cloud | On-prem TFS |
|---|---|---|
| Format | `https://dev.azure.com/{org}/{project}/_apis/...` | `https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/...` |

Sunucu adresi, collection ve proje adı, TFS'te herhangi bir proje sayfasının tarayıcı URL'inden okunabilir.

## Workflow Akışı

```
Webhook (PBI created / updated — "deleted" event'i pratikte kullanılmıyor, bkz. not)
  → Code: workItemId çıkar
  → HTTP Request: work item detayını relations dahil çek
  → Code (extract-parent): önce webhook'taki "removed" (unlink) ilişkisine bak,
                             yoksa HTTP Request cevabındaki güncel relations'a bak,
                             featureId'yi belirle
  → IF: sadece hedef Feature mı? (test aşamasında güvenlik amaçlı)
  → HTTP Request (WIQL): Feature'a bağlı tüm kardeş PBI ID'lerini çek
  → Code: ID'leri virgüllü stringe birleştir
  → HTTP Request: PBI'ların State + Total SP bilgilerini toplu çek
  → Code: Effort (tüm PBI'lar) ve Remaining (Done olmayanlar) toplamlarını hesapla, Removed'ları hariç tut
  → HTTP Request (PATCH): Feature'ın Effort ve Remaining alanlarını tek istekte güncelle
```

## Adım Adım Kurulum

### 1. Service Hook'ları kur

**Project Settings → Service Hooks** üzerinden iki hook oluşturuldu:
- **Work item updated** → filtre: `Work item type = Product Backlog Item`
- **Work item created** → filtre: `Work item type = Product Backlog Item`

İkisi de aynı n8n Webhook URL'ine yönlendirilir.

> ⚠️ **Kritik:** Filtre kesinlikle PBI ile sınırlı tutulmalı. Feature dahil edilirse, Feature güncellendiğinde hook tekrar tetiklenip sonsuz döngüye girilir.

> ⚠️ **Not — "Deleted" event'i gerekmedi:** Başta bir PBI'ın Feature'dan çıkarılması (Feature ekranındaki listeden "X" ile kaldırma) "delete" işlemiymiş gibi düşünülmüştü. Test edildiğinde bu işlemin aslında **`workitem.deleted` değil, `workitem.updated`** event'i fırlattığı görüldü — çünkü work item silinmiyor, sadece parent bağlantısı kopuyor (unlink). Bu yüzden ayrı bir "Work item deleted" hook'una ihtiyaç kalmadı, mevcut "updated" akışı zaten bu senaryoyu da kapsıyor.


### 2. Webhook node — Production URL

Geliştirme sırasında "Listen for Test Event" ile Test URL kullanılır. Workflow **Active** yapıldıktan sonra Webhook node'un **Production URL**'i kopyalanıp Service Hook'lardaki Action URL'i güncellenmelidir — aksi halde gerçek event'ler yakalanmaz.

### 3. Code — workItemId çıkarma

Webhook payload'ı n8n'de `body` altına sarılır.

```javascript
const workItemId = $json.body?.resource?.workItemId
  || $json.body?.resource?.id
  || $json.resource?.workItemId
  || $json.resource?.id;

return { json: { workItemId } };
```

> **Mod:** Run Once for Each Item (`$json` tek item'ı temsil eder, dönüş `return { json: {...} }` — dizi değil).

### 4. HTTP Request — work item detayını relations dahil çek

```
GET https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems/{{ $json.workItemId }}?api-version=7.1&$expand=relations
```

- Authentication: Generic Credential Type → Basic Auth (username boş, password = PAT)
- Tüm parametreler doğrudan URL'e yazıldı; ayrı "Send Query" parametre bloğu kullanımı sorun çıkardığı için tercih edilmedi.

### 5. Code (extract-parent) — featureId çıkarma (unlink dahil)

Bu node iki senaryoyu birlikte ele alır:

1. **Unlink (child, Feature'dan çıkarıldı):** Bu bilgi sadece webhook'un **o anki** payload'ında bulunur (`resource.relations.removed`), çünkü work item artık o Feature'a bağlı olmadığı için API'den tekrar sorgulandığında bu bilgi görünmez.
2. **Normal durum (state/alan değişikliği, yeni ekleme):** Bir önceki HTTP Request'in (detay çekme) güncel `relations` verisine bakılır.

```javascript
// Webhook'un orijinal payload'ına node referansıyla eriş
const webhookJson = $('Webhook').first().json; // 'Webhook' kısmını canvas'taki gerçek node adıyla değiştir

// 1. Önce webhook'ta "removed" (unlink) var mı kontrol et
const removedRels = webhookJson.body?.resource?.relations?.removed || [];
const removedParent = removedRels.find(r => r.rel === 'System.LinkTypes.Hierarchy-Reverse');

if (removedParent) {
  const parentId = removedParent.url.split('/').pop();
  return { json: { featureId: parentId, isUnlink: true } };
}

// 2. Unlink değilse, HTTP Request'in (şu an $json) getirdiği güncel relations'a bak
const rels = $json.relations || [];
const parentRel = rels.find(r => r.rel === 'System.LinkTypes.Hierarchy-Reverse');

if (parentRel) {
  const parentId = parentRel.url.split('/').pop();
  return { json: { featureId: parentId, isUnlink: false } };
}

return null; // parent hiç bulunamadı
```

> `relations.removed` içindeki her kayıtta `rel`, `url`, `attributes.isLocked`, `attributes.name` alanları bulunur — parent tespiti için `rel === 'System.LinkTypes.Hierarchy-Reverse'` yeterlidir.

### 6. IF — test güvenliği

Ekip verisini bozmamak için, test aşamasında sadece belirli bir Feature ID'sinde devam edecek şekilde bir IF node eklendi:

```
{{ $json.featureId }}  Equals  <TEST_FEATURE_ID>
```

True → workflow devam eder. False → bağlantı yok, işlem durur.

> Ekibe açmadan önce bu node kaldırılmalı veya koşulu genişletilmeli.

### 7. HTTP Request (WIQL) — kardeş PBI ID'lerini çek

```
POST https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/wiql?api-version=7.1
```

Body (expression modunda, `JSON.stringify` ile — WIQL sorgusundaki süslü parantezler n8n'in JSON body parser'ıyla çakıştığı için düz JSON yazımı hataya sebep olmuştu):

```javascript
{{ JSON.stringify({
  query: "SELECT [System.Id] FROM WorkItems WHERE [System.Parent] = " + $json.featureId + " AND [System.WorkItemType] = 'Product Backlog Item'"
}) }}
```

Dönüş yapısı: `{ workItems: [{ id, url }, ...] }`

> Unlink edilen PBI, artık WIQL sorgusunda `Parent = X` koşuluna uymadığı için sonuçlara dahil olmaz — bu da hesaplamadan otomatik olarak düşmesini sağlar, ekstra bir "çıkarma" mantığı gerekmez.

### 8. Code — ID'leri birleştir

```javascript
const items = $input.all();
const workItems = items[0].json.workItems;
const ids = workItems.map(wi => wi.id).join(',');
return [{ json: { idList: ids } }];
```

> **Mod:** Run Once for All Items.

### 9. HTTP Request — State + Total SP bilgilerini toplu çek

```
GET https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems
```

Query Parameters:
| Name | Value |
|---|---|
| `ids` | `{{ $json.idList }}` |
| `fields` | `System.State,TAKIM_ADI.TOTAL_SP_ORIG` |
| `api-version` | `7.1` |

### 10. Code — Effort ve Remaining hesaplama

```javascript
const items = $input.all();
const workItems = items[0].json.value;

let toplamEffort = 0;
let toplamRemaining = 0;

for (const item of workItems) {
  const state = item.fields['System.State'];
  const totalSP = item.fields['TAKIM_ADI.TOTAL_SP_ORIG'] || 0;

  if (state === 'Removed') {
    continue; // Removed olanlar hem Effort hem Remaining'e hiç dahil edilmez
  }

  toplamEffort += totalSP;
  if (state !== 'Done') {
    toplamRemaining += totalSP;
  }
}

return [{ json: { toplamEffort, toplamRemaining } }];
```

### 11. HTTP Request (PATCH) — Feature'ı güncelle

```
PATCH https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems/{{ $('extract-parent').first().json.featureId }}?api-version=7.1
```

- Body Content Type: **Raw**, Content-Type header: `application/json-patch+json` (Body Content Type "JSON" ile elle eklenen Content-Type header'ı birlikte kullanmak çakışmaya sebep olmuştu)
- Body (expression modunda):

```javascript
{{ JSON.stringify([
  {
    "op": "add",
    "path": "/fields/Microsoft.VSTS.Scheduling.Effort",
    "value": Number($json.toplamEffort)
  },
  {
    "op": "add",
    "path": "/fields/TAKIM_ADI.REMAINING_SP",
    "value": Number($json.toplamRemaining)
  }
]) }}
```

> **`replace` yerine `add` kullanıldı:** Alan boşken `replace` bazı durumlarda sessizce başarısız olabiliyor (200 dönüp değişiklik uygulanmıyor); `add` daha güvenilir çalıştı.

## Karşılaşılan Sorunlar ve Çözümleri

| Sorun | Sebep | Çözüm |
|---|---|---|
| `Cannot read properties of undefined (reading relations)` | Webhook payload'ı `body` altında geliyor, doğrudan `resource` değil | `$json.body?.resource?.relations` ile erişildi |
| `$json` altı çizili, "Run Once for Each Item" uyarısı | Node varsayılan "Run Once for All Items" modundaydı | Mod değiştirildi, dönüş formatı `return { json: {...} }` olarak güncellendi |
| `relations` boş geliyor | Webhook payload'ı her zaman ilişki verisini taşımıyor | Payload yerine work item detay API'si (`$expand=relations`) ile ayrı çekildi |
| HTTP Request'te "incorrect host domain" | Cloud formatı (`dev.azure.com`) kullanılmış, gerçekte on-prem TFS | Gerçek tarayıcı URL'inden `{sunucu}:{port}/tfs/{collection}/{project}` formatı çıkarıldı |
| WIQL isteğinde "Bad Request... caused by {" | JSON body içine `{{ }}` expression'ı doğrudan gömmek JSON syntax'ını bozuyor | Body tamamen `JSON.stringify(...)` ile expression modunda inşa edildi |
| `items.map is not a function` | WIQL sonucu ayrı item'lara bölünmüyor, tek item içinde `workItems` dizisi olarak geliyor | `items[0].json.workItems` üzerinden erişildi |
| PATCH'te "no fields" / referans hatası | `$('node-adı')` içindeki isim, canvas'taki gerçek node adıyla eşleşmiyordu | Node adı canvas'tan birebir kopyalanıp referans düzeltildi; `.item` yerine `.first()` kullanıldı |
| PATCH'te "you must pass a valid patch document" | Body Content Type ile elle eklenen Content-Type header çakışıyordu | Body Content Type **Raw** yapıldı, tek bir Content-Type header bırakıldı |
| PATCH'te "value cannot be null" | Araya eklenen geçici bir debug node, gerekli alanı taşımıyordu | Debug node kaldırılıp hesaplama node'u doğrudan PATCH'e bağlandı |
| PATCH başarılı (yeşil) ama Feature değişmemiş gibi görünüyor | TFS arayüzü gerçek zamanlı güncellenmiyor | Sayfa manuel yenilenince (F5) doğru değer görüldü |
| Effort/Remaining hesaplaması hep 0 çıkıyor | `fields` parametresinde yanlış reference name kullanılmış (`TAKIM_ADI.TOTAL_SP` yerine gerçek ad `TAKIM_ADI.TOTAL_SP_ORIG`), ayrıca alan boş olan PBI'larda API o alanı response'a hiç dahil etmiyor | Doğru reference name `fields` endpoint'inden teyit edildi |
| Remaining alanı hep sabit bir değerde kalıyor | PATCH'te yanlış `path` kullanılmış olabilir | Feature'ı tekil sorgulayıp gerçek Remaining reference name'i (`TAKIM_ADI.REMAINING_SP`) doğrulandı |
| "Silinen" PBI'lar Effort'tan düşmüyor sanılıyordu | Feature ekranından PBI'ı "X" ile kaldırmak aslında **silme değil, unlink** işlemi; `workitem.deleted` değil `workitem.updated` event'i tetikliyor | Ayrı bir "deleted" hook/kol kurmaya gerek kalmadı; `extract-parent` node'u `relations.removed` alanına bakacak şekilde genişletildi |
| Unlink sonrası parent bulunamıyor | HTTP Request (detay çekme) o anda **güncel** veriyi döndürüyor, unlink olmuş ilişki orada artık görünmüyor | Unlink bilgisi, webhook'un o anki payload'ındaki `resource.relations.removed` alanından okundu (node referansıyla `$('Webhook').first().json`) |

## Güvenlik / Test Notları

- Geliştirme sürecinde IF node ile sadece tek bir test Feature'ı üzerinde çalışıldı, ekibin diğer Feature'ları etkilenmedi
- Ekibe açmadan önce: IF node kaldırılmalı veya koşulu genişletilmeli
- n8n lokal (Node.js ile) çalıştığı için, bilgisayar kapandığında webhook'lar başarısız olup disable olabiliyor — üretime alınmadan önce n8n'in sürekli açık bir ortamda (örn. sunucu, Docker + `--restart unless-stopped`) çalıştırılması önerilir

## Test Edilen Senaryolar

- [x] Var olan bir PBI'ın Total SP'sini değiştirme → Effort ve Remaining güncelleniyor
- [x] Bir PBI'ı Done yapma → Effort sabit kalıyor, Remaining düşüyor
- [x] Bir PBI'ı Removed yapma → hem Effort'tan hem Remaining'den düşüyor
- [x] Feature altına yeni PBI ekleme → Effort ve Remaining artıyor
- [x] Bir PBI'ı Feature'dan unlink etme → eski Feature'ın Effort ve Remaining'i güncelleniyor
