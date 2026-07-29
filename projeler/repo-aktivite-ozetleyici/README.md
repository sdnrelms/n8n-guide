# 📝 Repo Aktivite Özetleyici

GitHub reposundaki son commit'leri çekip, Gemini ile Türkçe bir özet paragrafına dönüştüren ve sonucu Supabase'e kaydeden bir n8n otomasyonu.

## Akış

Schedule / Manual Trigger → HTTP Request (GitHub) → Code (prompt hazırlama) → Gemini → Code (format) → Supabase

1. **Schedule Trigger** — belirlenen sıklıkta (örn. günlük) workflow'u otomatik tetikler (test için Manual Trigger da mevcut)
2. **HTTP Request** — GitHub REST API'sinden son 10 commit'i çeker
3. **Code** — Commit verisini Gemini'ye gönderilecek prompt'a çevirir
4. **Gemini (Message a model)** — Ham veriyi okunabilir bir Türkçe özete dönüştürür
5. **Code** — Gemini çıktısını Supabase tablosuna uygun formata çevirir
6. **Supabase (Create a row)** — Özeti `repo-ozetleri` tablosuna kaydeder

## Kurulum

Bu projeyi çalıştırmak için gereken bağlantı kurulumları:
- [GitHub bağlantısı](../../github-n8n)
- [Gemini bağlantısı](../../gemini-n8n)
- [Supabase bağlantısı](../../supabase-n8n)

### Supabase tablosu

```sql
create table "repo-ozetleri" (
  id bigint generated always as identity primary key,
  created_at timestamptz default now(),
  repo_adi text,
  ozet text
);
```

## Kullanım

1. `workflow.json` dosyasını n8n'e import et (Workflows → Import from File)
2. Her node'daki credential alanlarını kendi credential'larınla eşleştir (HTTP Request → GitHub API, Message a model → Gemini API, Create a row → Supabase API)
3. HTTP Request node'undaki URL'de repo adını kendi repo'nla değiştir
4. Code node'undaki `repoAdi` değerini güncelle
5. Schedule Trigger'daki sıklığı ihtiyacına göre ayarla, workflow'u **Active** yap

## Notlar

- Gemini node'un çıktı yapısı sürüme göre değişebilir (`content.parts[0].text`), farklı bir sürümde farklı olabilir — çalıştırıp çıktıyı kontrol etmek gerekir
- Model olarak `gemini-3-flash-preview` kullanıldı, kendi hesabında farklı bir model aktifse ona göre değiştirilebilir
