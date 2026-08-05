# PBI Otomatik Durum Güncelleme (Azure DevOps + n8n)

Bir PBI **"Test but not done"** durumundayken, altındaki tüm child work item'lar
(Bug/Task/vb.) **Done** ya da **Committed** durumuna geçtiğinde, PBI'nin kendisini
otomatik olarak **Committed** durumuna çeken n8n workflow'u.


## Hedef

- Bir child work item (Bug/Task) Done veya Committed durumuna geçtiğinde webhook tetiklenir
- İlgili PBI bulunur, PBI'nin state'i gerçekten "Test but not done" mı kontrol edilir
- PBI'nin Feature'ı doğrulanır
- PBI'nin **tüm** child'ları çekilir, hepsi Done veya Committed mi kontrol edilir
- Hepsi tamamsa, PBI'nin state'i **Committed** olarak güncellenir (PATCH)

**Kapsam notu:** İlk aşamada workflow sadece Feature ID = 130 altındaki PBI'lar için
çalışacak şekilde kısıtlandı. Ekibe/tüm Feature'lara açmadan önce bu filtre kaldırılmalı
veya genişletilmelidir.

## Kullanılan Alanlar (Reference Name'ler)

| Alan | Reference Name | Bulunduğu Yer |
|---|---|---|
| Durum | `System.State` | PBI, Bug, Task |
| İş Öğesi Tipi | `System.WorkItemType` | Tüm work item'lar |
| Üst Öğe İlişkisi (yukarı) | `relations[].rel = System.LinkTypes.Hierarchy-Reverse` | Work item relations |
| Alt Öğe İlişkisi (aşağı) | `relations[].rel = System.LinkTypes.Hierarchy-Forward` | Work item relations |

> **Not:** Parent/Child bilgisi `fields` içinde düz bir alan olarak gelmiyor;
> `$expand=relations` ile çekilen work item'ın `relations` array'i üzerinden,
> `url`'ün sonundaki ID parse edilerek bulunuyor.

## Ön Koşul

- Azure DevOps bağlantısı (Basic Auth + PAT) — önceki projelerdeki credential aynen
  kullanıldı, yeniden kurulum gerekmedi.
- Azure DevOps Service Hook subscription'ı: **Work item updated** event'i, **filtre yok**
  (Work Item Type filtresi kasıtlı olarak boş bırakıldı — aşağıda "Karşılaşılan Sorunlar"
  bölümünde neden anlatılıyor).
- n8n Webhook node'u **Production URL** ile Service Hook'a bağlı ve workflow **Active**.

## Workflow Akışı (Node Node)

```
 1. Webhook
    → 2. IF: Değişen item'ın state'i Done veya Committed mi?
       → [False] dur
       → [True] devam
    → 3. HTTP Request "Get Torun": değişen item'ı $expand=relations ile çek
    → 4. Code "Extract Parent ID": relations'tan Parent (PBI) ID'sini çıkar
    → 5. HTTP Request "Get PBI": PBI'yi $expand=relations ile çek
    → 6. IF: PBI'nin state'i "Test but not done" mı?
       → [False] dur
       → [True] devam
    → 7. Code "Extract Feature ID": PBI'nin relations'ından kendi Parent'ını (Feature) çıkar
       (relations'ı da output'a taşımayı unutma!)
    → 8. IF: Feature ID = 130 mu?
       → [False] dur
       → [True] devam
    → 9. Code "Extract Child IDs": PBI'nin relations'ından TÜM child ID'lerini çıkar
    → 10. IF: childIds.length > 0 mu?
        → [False] dur (henüz child eklenmemiş)
        → [True] devam
    → 11. HTTP Request "Get Child States": tüm child'ların state + type bilgisini toplu çek
    → 12. Code "Check All Done": hepsi Done/Committed mi kontrol et (allDoneOrCommitted)
    → 13. IF: allDoneOrCommitted true mu?
        → [False] dur
        → [True] devam
    → 14. HTTP Request (PATCH) "Set PBI Committed": PBI'nin state'ini Committed yap
```

## Adım Adım Kurulum

### 1. Webhook Node

- HTTP Method: `POST`
- Path: örn. `azure-pbi-committed-check`
- Test aşamasında **Test URL**, doğrulandıktan sonra **Production URL** Service Hook'a
  girildi ve workflow **Active** hale getirildi.

**Azure DevOps Service Hook kurulumu:**
1. Project Settings → Service Hooks → **+ Create Subscription**
2. Trigger: **Work item updated**
3. Filtreler: **boş bırakıldı** (Work Item Type filtresi eklenmedi — nedeni aşağıda)
4. Action: **Web Hooks**, URL: n8n Production URL

### 2. IF — Değişen item Done/Committed mi?

Condition (OR):
- `{{ $json.body.resource.fields["System.State"].newValue }}` equals `Done`
- `{{ $json.body.resource.fields["System.State"].newValue }}` equals `Committed`

> Payload'da `System.State` alanı düz string değil, `{ oldValue, newValue }` şeklinde
> obje olarak geliyor (Service Hook "changed field" bilgisini bu formatta taşıyor).

### 3. HTTP Request — "Get Torun"

```
GET https://dev.azure.com/{organization}/{project}/_apis/wit/workitems/{{ $json.body.resource.workItemId }}?$expand=relations&api-version=7.1
```

- Authentication: Basic Auth (mevcut PAT credential)

### 4. Code — "Extract Parent ID"

```javascript
const item = $input.all()[0].json;
const relations = item.relations || [];

const parentRel = relations.find(r => r.rel === 'System.LinkTypes.Hierarchy-Reverse');
const parentId = parentRel ? parseInt(parentRel.url.split('/').pop()) : null;

return { json: { parentId } };
```

### 5. HTTP Request — "Get PBI"

```
GET https://dev.azure.com/{organization}/{project}/_apis/wit/workitems/{{ $json.parentId }}?$expand=relations&api-version=7.1
```

### 6. IF — PBI state = "Test but not done" mı?

Condition:
- `{{ $json.fields["System.State"] }}` equals `Test but not done`

> State adının Azure DevOps'taki gerçek yazımıyla **birebir** eşleşmesi gerekiyor
> (büyük/küçük harf, boşluk dahil) 

### 7. Code — "Extract Feature ID"

```javascript
const item = $input.all()[0].json; // "Get PBI" node'unun çıktısı
const relations = item.relations || [];

const parentRel = relations.find(r => r.rel === 'System.LinkTypes.Hierarchy-Reverse');
const featureId = parentRel ? parseInt(parentRel.url.split('/').pop()) : null;

return {
  json: {
    featureId,
    pbiId: item.id,
    relations   // ÖNEMLİ: sonraki node'un (Extract Child IDs) buna ihtiyacı var
  }
};
```

> **Kritik hata kaynağı:** İlk yazımda `relations` output'a eklenmemişti, bu yüzden
> sonraki node child'ları bulamıyordu. Zincir boyunca ihtiyaç duyulacak veriyi
> (`relations`, `pbiId` gibi) her Code node'da bir sonrakine taşımak gerekiyor.

### 8. IF — Feature ID 

Condition:
- `{{ $json.featureId }}` equals `130`

### 9. Code — "Extract Child IDs"

```javascript
const pbiItem = $('Get PBI').first().json; // node adı BİREBİR canvas'taki isimle eşleşmeli
const relations = pbiItem.relations || [];

const childRels = relations.filter(r => r.rel === 'System.LinkTypes.Hierarchy-Forward');
const childIds = childRels.map(r => parseInt(r.url.split('/').pop()));

return { json: { childIds, pbiId: pbiItem.id } };
```

### 10. IF — Child var mı?

Condition:
- `{{ $json.childIds.length }}` larger than `0`

> Henüz hiç child eklenmemiş bir PBI için boş id listesiyle sonraki HTTP çağrısı
> yapılırsa Azure DevOps API hata döner, bu adım bunu önlüyor.

### 11. HTTP Request — "Get Child States"

```
GET https://dev.azure.com/{organization}/{project}/_apis/wit/workitems?ids={{ $json.childIds.join(',') }}&fields=System.Id,System.State,System.WorkItemType&api-version=7.1
```

> Query parametrelerini ayrı ayrı "Query Parameters" alanlarına yazmak yerine, tam
> query string'i doğrudan URL'e (Expression modunda) gömmek tercih edildi — ayrı alanlara
> yazıldığında değerin başında/sonunda görünmez boşluk kalıp `TF1535: Cannot find field
> System.State` hatasına sebep olmuştu.

### 12. Code — "Check All Done"

```javascript
const data = $input.all()[0].json;
const items = data.value || [];

const acceptedStates = ['Done', 'Committed'];

const allDoneOrCommitted = items.length > 0 && items.every(item =>
  acceptedStates.includes(item.fields['System.State'])
);

const childStates = items.map(item => ({
  id: item.id,
  type: item.fields['System.WorkItemType'],
  state: item.fields['System.State']
}));

return {
  json: {
    pbiId: $('Extract Child IDs').first().json.pbiId, // node adını canvas'taki gerçek isimle eşleştir
    allDoneOrCommitted,
    childStates
  }
};
```

### 13. IF — allDoneOrCommitted true mu?

Condition:
- `{{ $json.allDoneOrCommitted }}` is true

### 14. HTTP Request (PATCH) — "Set PBI Committed"

```
PATCH https://dev.azure.com/{organization}/{project}/_apis/wit/workitems/{{ $json.pbiId }}?api-version=7.1
```

- Body Content Type: **Raw**, Content-Type header: `application/json-patch+json`
- Body (Expression modunda):

```javascript
{{ JSON.stringify([
  {
    "op": "add",
    "path": "/fields/System.State",
    "value": "Committed"
  }
]) }}
```

## Karşılaşılan Sorunlar ve Çözümleri

| Sorun | Sebep | Çözüm |
|---|---|---|
| Webhook hiç execution üretmiyordu | Service Hook subscription'ında "Work Item Type: Product Backlog Item" filtresi vardı — sadece PBI değişince tetikleniyordu, child (Bug/Task) değişimlerini yakalamıyordu | Work Item Type filtresi tamamen kaldırıldı, filtreleme node içinde (IF ile) yapılmaya bırakıldı |
| PBI State IF'i hep false dönüyordu | State adı yanlış yazılmıştı (yazım/boşluk farkı) | Azure DevOps'taki gerçek state adıyla birebir karşılaştırıldı |
| `TF1535: Cannot find field System.State` | HTTP Request'te Query Parameters alanına yazılan `fields` değerinde görünmez boşluk vardı | Query string doğrudan URL'e Expression olarak gömüldü, Query Parameters alanı kullanılmadı |
| `childIds` sürekli boş ya da sadece 1 elemanlı geliyordu | İlk seferde: Feature ID'yi çıkaran Code node `relations`'ı output'a taşımamıştı, veri sonraki node'a ulaşmıyordu. İkinci seferde: `childIds`'i çıkaran Code node, aynı isimli birden fazla "HTTP Request" node'u olduğu için yanlış node'a (`Get Torun`) referans veriyordu | (1) Her Code node'da ihtiyaç duyulacak veri açıkça output'a eklendi. (2) Tüm HTTP Request node'larına benzersiz isim verildi (`Get Torun`, `Get PBI`, `Get Child States`), Code node'larındaki `$('...')` referansları bu isimlerle güncellendi |
| "1. seviye çocuk" ile "PBI" karıştırılıyordu | İlk teşhiste hiyerarşinin bir seviye daha derin olduğu sanılmıştı | Test edilince aslında "1. seviye çocuk" zaten PBI'nin kendisiydi (relations'ında 3 child + 1 parent görünüyordu) — gerçek sorun derinlik değil, yanlış node referansıydı |

## Bilinen Riskler / Sonraki Aşama İçin Notlar

- **Error handling yok:** Hiçbir HTTP Request node'unda başarısız çağrı (429/500 vb.)
  senaryosu ele alınmadı. Feature filtresi kaldırılıp tüm ekibe açılmadan önce eklenmeli.
- **Self-trigger riski:** PATCH ile PBI'nin state'i değiştirildiğinde bu da bir
  "work item updated" event'i üretip webhook'u tekrar tetikliyor. Şu anki akışta bu yeni
  event muhtemelen 2. adımdaki IF'te (Done/Committed kontrolü, PBI'nin kendi state'i bu
  değerlerden biri olmadığı için) eleniyor, ama bu davranış ayrıca test edilip
  doğrulanmalı.
- **WIQL alternatifi düşünülebilir:** Child ID'lerini `relations` array'inden parse etmek
  yerine (`url.split('/').pop()`), Sprint Sürüm Düzeltme projesinde olduğu gibi WIQL
  sorgusu (`System.Parent = {pbiId}`) ile çekmek daha az hataya açık ve daha okunaklı
  olabilir.
- **Kapsam genişletme:** İlk aşama sadece Feature ID = 130 ile sınırlı. Ekibe açmadan
  önce bu filtre kaldırılmalı ya da tüm Feature'ları kapsayacak şekilde genişletilmeli.

## Test Edilen Senaryolar

- [x] Webhook, child (Bug/Task) work item güncellendiğinde tetikleniyor
- [x] Sadece Done/Committed'e geçen değişiklikler işleme devam ediyor
- [x] İlgili PBI relations üzerinden doğru şekilde bulunuyor
- [x] PBI'nin state'i "Test but not done" değilse akış duruyor
- [x] Feature filtresi (130) doğru çalışıyor
- [x] PBI'nin tüm child'ları (sadece tetikleyen değil, hepsi) doğru listeleniyor
- [x] Child'lardan biri bile Done/Committed değilse PATCH tetiklenmiyor
- [x] Tüm child'lar Done/Committed olduğunda PBI otomatik Committed'e çekiliyor
