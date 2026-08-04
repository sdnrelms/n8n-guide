# Sprint Bazlı Sürüm Düzeltme (TFS/Azure DevOps Server)

Bir sprint (Iteration Path) altındaki PBI ve Bug'ların **Sürüm** alanını toplu olarak
görüntüleyip, yanlış/eksik olanları tek bir formdan güncellemeyi sağlayan n8n workflow'u.
Feature-PBI Rollup projesinden **ayrı ve bağımsız** bir workflow'dur — sadece aynı n8n
instance'ı ve aynı TFS credential'ı paylaşılır.

## Hedef

- Bir sprint seçilir (Iteration Path)
- O sprint içindeki, ilgili Feature'a bağlı **tüm PBI'lar ve onların altındaki Bug'lar**
  listelenir (mevcut sürüm değerleriyle birlikte)
- Tek bir "Yeni Sürüm Değeri" girilir
- Listelenen tüm PBI + Bug'ların Sürüm alanı bu değerle güncellenir
- (Sonraki adım — henüz eklenmedi) Değişiklik özeti ekip görevlilerine mail olarak gönderilir

**Test güvenliği:** Ekibin canlı verisini bozmamak için, WIQL sorgusuna geçici olarak
`System.Parent = <TEST_FEATURE_ID>` şartı eklendi. Böylece sadece test amaçlı kullanılan
Feature'ın altındaki PBI/Bug'lar işleme dahil oluyor. **Ekibe açmadan önce bu şart
kaldırılmalı veya genişletilmelidir.**

## Kullanılan Alanlar (Reference Name'ler)

| Alan | Reference Name | Bulunduğu Yer |
|---|---|---|
| Sürüm | `Microsoft.VSTS.Build.IntegrationBuild` | PBI, Bug |
| Iteration Path | `System.IterationPath` | PBI, Bug |
| Başlık | `System.Title` | PBI, Bug |
| Parent (bağlı olduğu üst öğe) | `System.Parent` | PBI, Bug |

> Reference name teyidi, önceki Feature-PBI Rollup projesinde olduğu gibi
> `GET .../_apis/wit/fields?api-version=7.1` ile ya da doğrudan bir work item'ı çekip
> `fields` içindeki değerle ekrandaki değeri karşılaştırarak yapıldı.

> **Bug'ların hiyerarşisi:** Bu süreçte Bug'lar Feature'a değil, **PBI'a bağlı**
> (Feature → PBI → Bug). Bu yüzden Bug'ları bulmak için önce PBI ID'leri, sonra
> `System.Parent IN (pbiId1, pbiId2, ...)` ile ayrı bir sorgu gerekiyor.

## Ön Koşul

- TFS bağlantısı (Basic Auth + PAT) — Feature-PBI Rollup projesindeki credential aynen
  kullanılabilir, yeniden kurulumu gerekmez.
- Aynı n8n instance'ı üzerinde **yeni ve ayrı bir workflow** olarak oluşturuldu
  (isim: "Sprint Sürüm Düzeltme").

## Workflow Akışı (Şu Ana Kadarki Hali)

```
1. Form Trigger: sprint (Iteration Path) gir
  → 2. Code: WIQL body hazırla (PBI sorgusu + test Feature filtresi + sprint bilgisini output'a taşı)
  → 3. HTTP Request (WIQL): sprint + test Feature altındaki PBI ID'lerini çek
  → 4. Code: PBI ID'lerini listele + Bug WIQL body hazırla (birleştirilmiş node)
  → 5. HTTP Request (WIQL): o PBI'ların altındaki Bug ID'lerini çek
  → 6. Code: PBI ID'leri + Bug ID'lerini birleştirip idList oluştur
  → 7. HTTP Request: idList'teki tüm work item'ların Title + Sürüm bilgisini toplu çek
  → 8. Code: listText hazırla (PBI/Bug listesini okunabilir metne çevir)
  → 9. Form (2. ekran): mevcut liste gösterilir + "Yeni Sürüm Değeri" alanı alınır
  → 10. Code: PATCH için veri hazırla (id, title, eskiSurum, yeniSurum — her item ayrı satır)
  → 11. HTTP Request (PATCH): her work item için Sürüm alanını güncelle
```

> Mail adımı henüz eklenmedi — 10. adımda `eskiSurum` bilgisi zaten saklanıyor,
> ileride mail raporu için doğrudan kullanılabilir.

## Adım Adım Kurulum

### 1. Form Trigger

- Form Title: "Sprint Sürüm Düzeltme"
- Tek alan: **sprint (Iteration Path)** — Text, Required
- Girilecek değer TFS'teki **tam** Iteration Path olmalı (kısa isim değil). Doğru formatı
  görmek için bir PBI'ı şu endpoint ile sorgulamak gerekti:
  ```
  GET https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems/{PBI_ID}?fields=System.IterationPath&api-version=7.1
  ```
  Dönen `fields.System.IterationPath` değeri birebir kullanılmalı (Türkçe karakter/harf
  farkları TF51011 "specified iteration path doesn't exist" hatasına sebep olabiliyor).

### 2. Code — WIQL body hazırla (PBI sorgusu)

```javascript
const formItem = $input.all()[0].json;
const sprint = Object.values(formItem)[0]; // form alan adı ne olursa olsun ilk değeri al

const TEST_FEATURE_ID = <TEST_FEATURE_ID>; // ekibe açarken kaldırılacak/genişletilecek

const query = "SELECT [System.Id] FROM WorkItems WHERE [System.IterationPath] = '" + sprint + "' AND [System.WorkItemType] = 'Product Backlog Item' AND [System.Parent] = " + TEST_FEATURE_ID;

return { json: { wiqlBody: JSON.stringify({ query }), sprint } };
```

> **Mod:** Run Once for Each Item.
> `sprint` değeri output'a eklendi çünkü ilerideki Bug sorgusunda tekrar lazım olabilir
> (şu anki Bug sorgusunda kullanılmıyor, ama referans olarak saklanıyor).

### 3. HTTP Request (WIQL) — PBI ID'lerini çek

```
POST https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/wiql?api-version=7.1
```

- Body Content Type: **Raw**, Content-Type header: `application/json`
- Body (Expression modunda): `{{ $json.wiqlBody }}`
- Authentication: Basic Auth (mevcut credential)

Dönüş: `{ workItems: [{ id, url }, ...] }`

### 4. Code — PBI ID listele + Bug WIQL body hazırla (birleştirilmiş)

```javascript
const data = $input.all()[0].json;
const workItems = data.workItems;
const pbiIds = workItems.map(wi => wi.id);

const sprint = $('Code').first().json.sprint; // 2. node'un canvas'taki gerçek adı
const parentClause = pbiIds.join(', ');
const query = "SELECT [System.Id] FROM WorkItems WHERE [System.WorkItemType] = 'Bug' AND [System.Parent] IN (" + parentClause + ")";

return { json: { wiqlBody: JSON.stringify({ query }), pbiIds } };
```

> Başlangıçta bu adım iki ayrı Code node'a (PBI ID listele / Bug WIQL hazırla) bölünmüştü,
> sadece okunabilirlik için. Teknik bir zorunluluk olmadığı görülünce tek node'da birleştirildi.

### 5. HTTP Request (WIQL) — Bug ID'lerini çek

3. adımdaki node'un birebir kopyası, tek fark Body: `{{ $json.wiqlBody }}` (bu node'un
kendi `wiqlBody`'si, yani 4. adımdan geleni kullanır).

### 6. Code — PBI + Bug ID'lerini birleştir

```javascript
const pbiIds = $('Code1').first().json.pbiIds; // 4. node'un canvas'taki gerçek adı
const bugData = $input.all()[0].json;
const bugItems = bugData.workItems || [];
const bugIds = bugItems.map(wi => wi.id);
const allIds = [...pbiIds, ...bugIds];
return { json: { idList: allIds.join(',') } };
```

### 7. HTTP Request — Title + Sürüm bilgilerini toplu çek

```
GET https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems
```

Query Parameters:
| Name | Value |
|---|---|
| `ids` | `{{ $json.idList }}` |
| `fields` | `System.Title,Microsoft.VSTS.Build.IntegrationBuild` |
| `api-version` | `7.1` |

Dönüş: `{ value: [{ id, fields: { 'System.Title': ..., 'Microsoft.VSTS.Build.IntegrationBuild': ... } }, ...] }`

### 8. Code — listText hazırla

```javascript
const data = $input.all()[0].json;
const items = data.value;

let listText = "Mevcut PBI/Bug'lar ve surumleri:" + "\n\n";
for (const item of items) {
  const title = item.fields['System.Title'];
  const version = item.fields['Microsoft.VSTS.Build.IntegrationBuild'] || '(bos)';
  listText += "#" + item.id + " - " + title + " - Mevcut: " + version + "\n";
}

return { json: { listText, items } };
```

> **Not:** Backtick (`` ` ``) template literal yerine `+` ile string birleştirme
> kullanıldı — backtick karakteri editöre yapıştırılırken bozulup düz tırnağa
> dönüşebiliyor, bu da `${...}` ifadelerinin değerlendirilmeden metne aynen
> yazılmasına sebep oluyordu.

### 9. Form (2. ekran) — listeyi göster + yeni sürüm değerini al

- Form Description (Expression modunda): `{{ $json.listText }}`
- Form Fields: **Yeni Sürüm Değeri** — Text, Required

### 10. Code — PATCH için veri hazırla

```javascript
const formItem = $input.all()[0].json;
const formValue = Object.values(formItem)[0]; // "Yeni Sürüm Değeri"

const pbiData = $('Code').first().json.items; // listText'i üreten (8.) node'un gerçek adı

const output = pbiData.map(item => ({
  json: {
    id: item.id,
    title: item.fields['System.Title'],
    eskiSurum: item.fields['Microsoft.VSTS.Build.IntegrationBuild'] || '(boş)',
    yeniSurum: formValue
  }
}));

return output;
```

> **Mod:** Run Once for All Items — tek input'tan, work item sayısı kadar output
> item'ı üretmek için gerekli.

### 11. HTTP Request (PATCH) — her work item'ı güncelle

```
PATCH https://{sunucu}:{port}/tfs/{collection}/{project}/_apis/wit/workitems/{{ $json.id }}?api-version=7.1
```

- Body Content Type: **Raw**, Content-Type header: `application/json-patch+json`
- Authentication: Basic Auth (mevcut credential)
- Body (Expression modunda):

```javascript
{{ JSON.stringify([
  {
    "op": "add",
    "path": "/fields/Microsoft.VSTS.Build.IntegrationBuild",
    "value": $json.yeniSurum
  }
]) }}
```

> `replace` yerine yine `add` kullanıldı — alan boşken `replace` sessizce başarısız
> olabiliyor.
> n8n, 10. adımdan gelen her item için bu node'u otomatik olarak ayrı ayrı çalıştırır
> (döngü kurmaya gerek yok).

## Karşılaşılan Sorunlar ve Çözümleri

| Sorun | Sebep | Çözüm |
|---|---|---|
| WIQL'de "expecting comparison operator" hatası | HTTP Request Body alanı Expression modunda değildi, `$json...` ifadesi düz metin olarak gönderiliyordu | Body input'unun yanındaki fx/Expression anahtarı aktif hale getirildi |
| Form'dan gelen alan adını bulmak zor oldu | Label'daki parantez/boşluk n8n'de farklı bir key'e dönüşebiliyor | `Object.values(formItem)[0]` ile alan adına bağımlı kalmadan ilk değeri almak tercih edildi |
| `Cannot read properties of undefined ($json)` | Node "Run Once for All Items" modundayken `$json` tek item'ı temsil etmiyor | `$input.all()[0].json` ile açıkça ilk item'a erişildi |
| `TF51011: the specified iteration path doesn't exist` | Forma yazılan sprint değeri (örn. "8. Sprint") gerçek Iteration Path formatıyla eşleşmiyordu | Bir PBI üzerinden `System.IterationPath` alanı API'den çekilip birebir kopyalandı |
| listText'te `#${item.id}...` gibi değerlendirilmemiş template literal göründü | Kod editöre yapıştırılırken backtick (`` ` ``) karakteri düz tırnağa dönüşmüştü | Template literal yerine `+` ile string birleştirme kullanıldı |
| Bug'lar WIQL sonucunda hiç görünmüyordu | Bug'lar Feature'a değil PBI'a bağlı (`System.Parent` PBI'ı gösteriyor, Feature'ı değil) | Önce PBI ID'leri bulunup, sonra `System.Parent IN (pbiId1, pbiId2, ...)` ile ayrı bir Bug sorgusu yapıldı |
| PATCH'te `TF401320: rule error for field IntegrationBuild — InvalidNotEmpty` | Bug work item'da bu alan için TFS process kuralı farklı davranıyor olabilir / gönderilen değer boş gitmiş olabilir | PATCH body'sindeki `value` alanının dolu geldiği doğrulandı, sorun giderildi |
| `workItems is not defined` hatası | Bug WIQL sonucunu işleyen node'da eski kod satırı (`$json.workItems` yerine düz `workItems`) kalıntı olarak durmuştu | Kod satırı `bugData.workItems` şeklinde düzeltildi |

## Test Edilen Senaryolar

- [x] Sprint (Iteration Path) formu doğru değerle çalışıyor
- [x] Test Feature altındaki PBI'lar WIQL ile doğru listeleniyor
- [x] PBI'ların altındaki Bug'lar ayrı sorgu ile doğru bulunuyor
- [x] 2. formda mevcut PBI + Bug listesi ve sürümleri doğru gösteriliyor
- [x] Yeni sürüm değeri girildiğinde tüm PBI + Bug'lar PATCH ile güncelleniyor

