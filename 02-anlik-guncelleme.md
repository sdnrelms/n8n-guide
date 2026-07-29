# n8n ↔ Azure DevOps: Anlık Güncelleme Mantığı (Feature-PBI Rollup)

Bu belge, bağlantı kurulduktan sonra PBI değişikliklerini yakalayıp bağlı oldukları Feature'ın toplam skorunu otomatik güncelleme akışını anlatır.

> Ön koşul: **[01-baglanti-kurma.md](./01-baglanti-kurma.md)** tamamlanmış olmalı.

## Hedef

Üç senaryoyu tek bir mantıkla çözüyoruz:

- Child'ların (PBI) toplamı parent'ı (Feature) besleyecek
- Bir PBI **Done** olunca skoru toplamdan düşecek
- Yeni bir PBI eklenince toplam otomatik güncellenecek

> **Temel mantık:** Her tetiklenmede tüm kardeş PBI'lar sıfırdan yeniden toplanır ve Feature'a yazılır. Ayrı "ekle/çıkar" state'i tutmaya gerek yok — bu hem daha güvenli hem daha az hataya açık.

## Adım 1 — Gelen PBI'dan Parent (Feature) ID'sini çıkar

Webhook payload'ı içinde `resource.relations` dizisinde `rel: "System.LinkTypes.Hierarchy-Reverse"` olan kayıt parent'ı gösterir.

**Code node:**
```javascript
const rels = $json.resource.relations || [];
const parentRel = rels.find(r => r.rel === 'System.LinkTypes.Hierarchy-Reverse');
if (!parentRel) return []; // parent yoksa devam etme
const parentId = parentRel.url.split('/').pop();
return [{ json: { featureId: parentId } }];
```

> Gerçek payload'ı ilk testte dikkatlice incele — bazı sürümlerde bu bilgi `resource.fields['System.Parent']` altında da gelebiliyor.

## Adım 2 — Feature'a bağlı tüm PBI'ları WIQL ile çek

**HTTP Request node** (Method: POST):
```
https://dev.azure.com/{org}/{project}/_apis/wit/wiql?api-version=7.1
```

**Body:**
```json
{
  "query": "SELECT [System.Id] FROM WorkItems WHERE [System.Parent] = {{ $json.featureId }} AND [System.WorkItemType] = 'Product Backlog Item'"
}
```

## Adım 3 — PBI'ların State + skor alanlarını çek

Önce ID'leri birleştir (Code node), sonra:

```
GET https://dev.azure.com/{org}/{project}/_apis/wit/workitems?ids={idList}&fields=System.State,Custom.SkorAlaninizinAdi&api-version=7.1
```

> `Custom.SkorAlaninizinAdi` yerine gerçek alan referans adını kullan (`Microsoft.VSTS.Scheduling.RemainingWork` veya kendi custom alanınız). Reference name'i doğrulamak için:
> ```
> GET https://dev.azure.com/{org}/{project}/_apis/wit/fields?api-version=7.1
> ```

## Adım 4 — Hesaplama (Code node)

```javascript
const items = $json.value;
let total = 0;
for (const item of items) {
  const state = item.fields['System.State'];
  const score = item.fields['Custom.SkorAlaninizinAdi'] || 0;
  if (state !== 'Done') {
    total += score;
  }
}
return [{ json: { featureId: $('extract-parent').first().json.featureId, total } }];
```

Bu tek formül üç maddeyi de otomatik çözer: toplama, Done olunca düşme, yeni ekleme — çünkü her tetiklenmede tüm kardeşler sıfırdan toplanır.

## Adım 5 — Feature'ı PATCH ile güncelle

```
PATCH https://dev.azure.com/{org}/{project}/_apis/wit/workitems/{{ $json.featureId }}?api-version=7.1
Content-Type: application/json-patch+json
```

**Body:**
```json
[
  {
    "op": "replace",
    "path": "/fields/Custom.SkorAlaninizinAdi",
    "value": "={{ $json.total }}"
  }
]
```

> **Not:** `add` yerine `replace` kullanılmalı, çünkü alan zaten dolu olacak (sürekli güncelleniyor). Azure bazen `add`'i de tolere eder ama teknik olarak doğrusu `replace`'tir.

## Test Senaryoları

1. Var olan bir PBI'ın skorunu değiştir → Feature güncelleniyor mu?
2. Bir PBI'ı **Done** yap → toplamdan düşüyor mu?
3. Feature altına yeni bir PBI ekle → toplam artıyor mu?

## Kritik Dikkat Noktaları

- **Sonsuz döngü riski:** Service Hook filtresi kesinlikle **PBI** ile sınırlı kalmalı, Feature asla dahil edilmemeli.
- **Payload doğrulama:** İlk gerçek webhook payload'ı incelenmeden Adım 1'deki kod kesinleştirilmemeli — alan adları Azure sürümüne göre farklılık gösterebilir.
- **Rate limit:** Sık tetiklenen büyük projelerde Azure DevOps API istek limitlerine dikkat edilmeli; gerekirse kısa bir **Wait node** ile debounce eklenebilir.
