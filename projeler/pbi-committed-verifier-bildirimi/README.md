# PBI Committed → Verifier Bildirimi (TFS/Azure DevOps Server)

Bir PBI (Product Backlog Item) **Committed** durumuna geçtiğinde, PBI üzerindeki **Verifier** alanında belirtilen kişiyi otomatik olarak Discussion'da (yorumlarda) tıklanabilir bir mention ile etiketleyip bilgilendiren bir n8n workflow'u. On-premises TFS/Azure DevOps Server üzerinde çalışacak şekilde kuruldu.

## Hedef

- Bir PBI'ın **State** alanı **Committed**'e yeni geçtiğinde (zaten Committed iken tekrar tekrar tetiklenmeden) otomatik bir bildirim tetiklensin
- PBI'daki **Verifier** kişisi, work item'ın Discussion kısmına gerçek bir mention (`@İsim`, tıklanabilir/bildirim gönderen link) olarak etiketlensin ve yanına otomatik bir mesaj yazılsın
- Verifier alanı boşsa hata vermeden, sessizce durup n8n'in execution log'unda görünür bir uyarı bıraksın

Bu, [Feature-PBI Rollup](../feature-pbi-rollup-tfs) projesinden **tamamen bağımsız, ayrı bir workflow**. Aynı "Work item updated" event'ini dinliyor olsalar da farklı işler yaptıkları ve farklı hata noktaları oldukları için ayrı tutuldular.

## Kullanılan Alan

| Alan | Reference Name | Bulunduğu Yer |
|---|---|---|
| Verifier | `TAKIM_ADI.Verifier` | PBI |

Verifier alanının değeri bir kişi objesidir: `{ displayName, id, uniqueName, url, imageUrl, links, ... }`. Mention için `displayName` ve `id` (GUID) kullanılır.

## Ön Koşul

Genel PAT/credential kurulumu ve on-prem TFS URL formatı için [Azure DevOps bağlantı belgesine](../../azure-devops-n8n/01-baglanti-kurma.md) ve [Feature-PBI Rollup projesine](../feature-pbi-rollup-tfs) bakılabilir.

## Workflow Akışı 

```
Webhook (PBI updated)
  → Code: workItemId çıkar
  → Code: State gerçekten Committed'e mi döndü? (shouldNotify)
  → IF: shouldNotify?
  → HTTP Request: work item detayını relations + tüm alanlar dahil çek ($expand=all)
  → Code: featureId (parent Feature) çıkar
  → IF: Verifier dolu mu?
  → Code: Verifier'ın mention HTML'ini üret
  → IF: featureId sadece test Feature'ı mı? (test aşamasında güvenlik amaçlı)
  → HTTP Request (PATCH): Discussion'a mention'lı yorum ekle
```

## Adım Adım Açıklama

### 1. Webhook + Service Hook

Rollup projesinden **ayrı, yeni bir Webhook node** kullanıldı (yeni bir workflow olduğu için). Azure DevOps'ta ayrı bir Service Hook kuruldu:
- Trigger: **Work item updated**
- Filtre: **Work item type = Product Backlog Item**
- Action URL: bu workflow'un kendi Webhook Production URL'i

> ⚠️ Aynı "Work item updated" event'i için birden fazla Service Hook (farklı n8n workflow'larına giden) tanımlanabilir, TFS her hook'a paralel istek atar. Sorun değildir.

### 2. Code — workItemId çıkarma

```javascript
const workItemId = $json.body?.resource?.workItemId
  || $json.body?.resource?.id
  || $json.resource?.workItemId
  || $json.resource?.id;

return { json: { workItemId } };
```

### 3. Code — State değişikliği kontrolü (shouldNotify)

Amaç: sadece **az önce** Committed'e geçilmişse tetiklenmek, State zaten Committed iken yapılan başka güncellemelerde (örn. SP değişince) tekrar tekrar bildirim göndermemek.

TFS'in webhook payload'ında değişen alanların eski/yeni değeri **`resource.fields`** altında `{ oldValue, newValue }` formatında gelir (ilk denemede `resource.revision.fields` sanılmıştı, gerçek yol `resource.fields` çıktı).

```javascript
const webhookJson = $('Webhook').first().json; // 'Webhook' kısmını canvas'taki gerçek node adıyla değiştir

const stateChange = webhookJson.body?.resource?.fields?.['System.State'];

if (!stateChange) {
  return { json: { shouldNotify: false, workItemId: $json.workItemId } };
}

const isNowCommitted = stateChange.newValue === 'Committed';
const wasNotCommittedBefore = stateChange.oldValue !== 'Committed';

return { json: { shouldNotify: isNowCommitted && wasNotCommittedBefore, workItemId: $json.workItemId } };
```

> **Mod:** Run Once for Each Item.

### 4. IF — shouldNotify kontrolü

Koşul tipi **Boolean** seçilmeli (String ile boolean karşılaştırması `"true is a boolean but was expecting a string"` hatası verir):

```
{{ $json.shouldNotify }}  is true
```

### 5. HTTP Request — work item detayını relations + tüm alanlar dahil çek

```
GET https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems/{{ $json.workItemId }}?api-version=7.1&$expand=all
```

> **Not:** İlk denemede sadece `fields=TAKIM_ADI.Verifier` parametresiyle Verifier çekilmiş, `relations`'a ihtiyaç duyulunca `$expand=relations` eklenmiş ama `fields` parametresiyle birlikte kullanınca relations boş dönmüştü. Çözüm: `fields` parametresini tamamen kaldırıp `$expand=all` kullanmak — bu hem tüm alanları hem tüm ilişkileri tek istekte getirir.

### 6. Code — featureId çıkarma

```javascript
const rels = $json.relations || [];
const parentRel = rels.find(r => r.rel === 'System.LinkTypes.Hierarchy-Reverse');
const featureId = parentRel ? parentRel.url.split('/').pop() : null;

return { json: { ...$json, featureId } };
```

> `...$json` ile mevcut tüm veri (fields, id vs.) korunuyor, üzerine sadece `featureId` ekleniyor — sonraki node'ların hem Verifier bilgisine hem featureId'ye erişebilmesi için önemli.

### 7. IF — Verifier dolu mu?

```
{{ $json.fields['TAKIM_ADI.Verifier'].displayName }}  Is Not Empty
```

### 8. Code — mention HTML'i üret

TFS'te gerçek bir mention (tıklanabilir, bildirim gönderen) oluşturmak için düz `@İsim` metni yeterli değildir; kullanıcının GUID'ini içeren özel bir HTML formatı gerekir:

```javascript
const verifier = $json.fields['TAKIM_ADI.Verifier'];
const verifierName = verifier?.displayName;
const verifierId = verifier?.id;

const mentionHtml = `<a href="#" data-vss-mention="version:2.0,${verifierId}">@${verifierName}</a>`;

return { json: { mentionHtml, verifierName, workItemId: $json.id, featureId: $json.featureId } };
```

> `featureId: $json.featureId` satırı unutulmamalı — bu node, önceki node'un tüm çıktısını otomatik miras almaz, sadece `return` edilen alanlar bir sonrakine geçer. İlk denemede bu satır unutulunca, sonraki IF node'da `featureId` kayboluyordu.

### 9. IF — test güvenliği (featureId)

```
{{ $json.featureId }}  Equals  <TEST_FEATURE_ID>
```

> Rollup projesindekiyle aynı mantık: test aşamasında sadece belirli bir Feature'a bağlı PBI'larda çalışır, ekibin diğer verilerini etkilemez. Ekibe açılırken kaldırılmalı veya genişletilmeli.

### 10. HTTP Request (PATCH) — Discussion'a yorum ekle

TFS'te work item yorumları **`System.History`** alanına yazılır:

```
PATCH https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems/{{ $json.workItemId }}?api-version=7.1
```

- Body Content Type: **Raw**, Content-Type header: `application/json-patch+json`
- Body (expression modunda):

```javascript
{{ JSON.stringify([
  {
    "op": "add",
    "path": "/fields/System.History",
    "value": $json.mentionHtml + " bu PBI Committed durumuna geçti, kontrol edebilir misin?"
  }
]) }}
```

## Karşılaşılan Sorunlar ve Çözümleri

| Sorun | Sebep | Çözüm |
|---|---|---|
| Service Hook "Enabled (Restricted)" durumuna düşüyor, tekrar tekrar | Önceki başarısız (500 hatalı) test denemeleri Azure'u kısıtlamaya itiyor | Hook silinip sıfırdan oluşturuldu; asıl kaynak sorun (kod hatası) çözülünce tekrar oluşmadı |
| Test URL ile tarayıcıdan denemede "is not registered" hatası | "Listen for Test Event" süresi dolmuş / dinleme aktif değilmiş | Dinlemeyi başlatıp hemen ardından test edildi, süre aşımına dikkat edildi |
| "This webhook is not GET, did you mean POST" | Tarayıcıdan GET ile test edilmişti, webhook POST bekliyor | Bu aslında webhook'un kayıtlı olduğunu doğrulayan olumlu bir sinyaldi, gerçek POST testine geçildi |
| Node'ları ayırınca hiç execution düşmedi | Webhook node bağlantı/state karmaşasına girmişti | Webhook node silinip yeniden eklendi, temiz bağlantı kuruldu |
| Committed yaptıktan sonra execution düşmedi | Workflow **Active** değildi / Production URL kullanılmıyordu, hâlâ Test URL + Listen modundaydı | Workflow Active yapıldı, Production URL Service Hook'a girildi |
| `shouldNotify` hep `false` | State değişikliği bilgisinin gerçek payload yolu yanlış tahmin edilmişti (`revision.fields` denendi, gerçek yol `resource.fields` çıktı) | Debug ile (`JSON.stringify(resource)`) gerçek yapı görülüp yol düzeltildi |
| State kontrolü node'unda "no fields" | Bu node, `workItemId` node'unun ardında çalışıyordu, `$json` artık webhook'un ham verisini taşımıyordu | `$('Webhook').first().json` ile doğrudan webhook node'una referans verildi |
| IF node'da "true is a boolean but was expecting a string" | Koşul tipi String seçiliyken boolean değer karşılaştırılmaya çalışılmıştı | Koşul tipi Boolean yapıldı |
| featureId kontrolü IF'i hep false dönüyor | `featureId`'yi taşıyan node ile debug/IF node arasında bir ara node (mentionHtml üreten) `featureId`'yi kendi `return`ünde taşımıyordu, veri kayboluyordu | mentionHtml üreten node'un `return`üne `featureId: $json.featureId` eklendi |
| Discussion'a eklenen `@İsim` tıklanabilir/link gibi görünmüyor | Düz metin `@İsim` TFS'te gerçek mention sayılmıyor, bildirim tetiklemiyor | Verifier'ın `id` (GUID) alanı kullanılarak `data-vss-mention="version:2.0,{GUID}"` içeren HTML formatı üretildi |

## Test Edilen Senaryolar

- [x] Bir PBI'ı Committed yapma (Verifier dolu) → Discussion'a mention'lı yorum ekleniyor
- [x] Aynı PBI'da başka bir alan değişse (State Committed kalırken) → tekrar tetiklenmiyor (shouldNotify false)
- [x] Sadece test Feature'ına bağlı PBI'larda çalışıyor, diğer Feature'lara dokunmuyor
