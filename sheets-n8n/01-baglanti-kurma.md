# n8n ↔ Google Sheets: Bağlantı Kurma

Bu belge, n8n üzerinden Google Sheets node'unu kullanabilmek için Google Cloud üzerinde OAuth2 kimlik doğrulama kurulumunu anlatır.

> Not: Bu akış Gmail kurulumuyla neredeyse birebir aynı. Gmail için oluşturduğun Google Cloud projesini burada da kullanabilirsin, yeni proje açmana gerek yok.

## Adım 1 — Google Cloud'da Sheets API'yi etkinleştir

1. [Google Cloud Console](https://console.cloud.google.com/) → Gmail için kullandığın projeyi seç (ya da yeni proje aç)
2. Sol menüden **APIs & Services → Library**
3. Arama kutusuna `Google Sheets API` yaz → **Enable**
4. Aynı şekilde eğer Drive üzerinden dosya seçtirme özelliğini kullanacaksan `Google Drive API`'yi de etkinleştirmen gerekebilir (n8n'in Sheets node'u dosya listeleme için Drive API'yi de kullanabiliyor)

## Adım 2 — OAuth consent screen kontrolü

Gmail kurulumunda bunu zaten yapmıştın:

1. **APIs & Services → OAuth consent screen**
2. **Scopes** kısmına Sheets ile ilgili scope'u ekle (örn. `.../auth/spreadsheets`)
3. **Test users** listesinde kendi hesabının olduğunu doğrula (Gmail için eklemiştin, aynı liste)

> Eğer Gmail kurulumunda consent screen'i zaten dolduruysan bu adımda sadece yeni scope eklemen yeterli, sıfırdan doldurmana gerek yok.

## Adım 3 — Aynı OAuth Client ID'yi kullan (ya da yeni oluştur)

İki seçeneğin var:

**Seçenek A (önerilen):** Gmail için oluşturduğun **Client ID / Client Secret**'i Sheets için de kullan — aynı Google Cloud projesindeysen ve scope'ları consent screen'e eklediysen tek bir OAuth client hem Gmail hem Sheets için çalışır.

**Seçenek B:** Ayrı bir Client ID oluşturmak istersen, Gmail rehberindeki Adım 4'ü tekrarla (**Credentials → Create Credentials → OAuth client ID**, redirect URI'yi n8n'den kopyala).

## Adım 4 — n8n'de credential oluştur

1. n8n → **Credentials → New → Google Sheets OAuth2 API**
2. **Client ID** ve **Client Secret**'i yapıştır (Gmail'dekiyle aynı ya da yeni oluşturduğun)
3. **Connect my account** → Google hesabını seç, izinleri onayla
4. Bağlantı başarılıysa credential aktif görünür

## Adım 5 — Test et

1. Yeni bir workflow'da **Google Sheets** node ekle
2. Credential olarak oluşturduğun hesabı seç
3. Operation: **Append Row** ya da **Read Rows** seç (test için ne uygunsa)
4. Bir Spreadsheet ID ve Sheet adı belirt (Spreadsheet ID, Sheets URL'sindeki `/d/{ID}/edit` kısmındaki ID'dir)
5. Çalıştır, sonucu kontrol et

## Dikkat Edilecek Noktalar

- **Aynı proje, farklı scope:** Gmail ve Sheets aynı OAuth client'ı paylaşabilir ama consent screen'de her ikisinin scope'unun da eklenmiş olması gerekir, yoksa "insufficient scope" hatası alırsın.
- **Spreadsheet ID karışıklığı:** Sheet adı ile Spreadsheet ID'yi karıştırma — ID, URL'nin ortasındaki uzun karakter dizisidir, sekme (tab) adı değildir.
- **Test mode kısıtı:** Gmail'de olduğu gibi consent screen "Testing" modundaysa sadece test user listesindeki hesaplar bağlanabilir.
