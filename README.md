![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Status](https://img.shields.io/badge/status-actively%20learning-brightgreen?style=for-the-badge)

![Azure DevOps](https://img.shields.io/badge/Azure%20DevOps-0078D7?style=flat-square&logo=azuredevops&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail-D14836?style=flat-square&logo=gmail&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=flat-square&logo=telegram&logoColor=white)
![Gemini](https://img.shields.io/badge/Gemini-8E75B2?style=flat-square&logo=googlegemini&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=flat-square&logo=supabase&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)

# n8n Guide

Bu repo, n8n öğrenirken tuttuğum notlardan oluşuyor. 

Repo canlı, zamanla büyüyor: yeni bir servis bağladıkça, yeni bir şey öğrendikçe buraya ekliyorum.

## n8n Nedir?

[n8n](https://n8n.io/) (okunuşu "n-eight-n"), farklı uygulamaları ve servisleri birbirine bağlayıp otomasyon iş akışları (workflow) oluşturmanı sağlayan, açık kaynaklı bir workflow otomasyon aracı. Zapier veya Make (eski adıyla Integromat) gibi araçlara benzetilebilir, ama açık kaynaklı olması ve kendi sunucunda (self-hosted) çalıştırabilmen en büyük farkı.

Temel mantık şöyle:
- **Node'lar** bir işlemi temsil eder (bir API'ye istek atmak, veri dönüştürmek, e-posta göndermek gibi)
- Node'ları birbirine bağlayarak görsel bir akış (workflow) oluşturursun
- Bir **Trigger** (tetikleyici) — zamanlanmış görev, webhook, manuel çalıştırma vs. — workflow'u başlatır
- Kod yazmadan da çok şey yapılabilir, ama gerektiğinde **Code node** ile JavaScript de yazabilirsin
  
Örnek: "Her Cuma saat 17:00'de Azure DevOps'tan o hafta kapanan işleri çek, özet çıkar, Gmail ile ekibe gönder" gibi bir akışı kod yazmadan, sadece node'ları sürükleyip bağlayarak kurabilirsin.

Daha günlük bir örnek: "Bir web sitesindeki form doldurulduğunda, gelen bilgileri otomatik olarak bir Google Sheets tablosuna kaydet, aynı anda başvuran kişiye onay e-postası gönder, ekibe de Telegram'dan bildirim düş" — bunun gibi bir akış da yine kod yazmadan, birkaç node'u birbirine bağlayarak kurulabilir.

## Bu Repo Ne İşe Yarıyor?

- **Bağlantı kurulumları:** Hangi servisi n8n'e nasıl bağladığımın adım adım notları (API key/token nereden alınır, n8n'de credential nasıl kurulur)
- **n8n'in kendisi:** Kurulum yöntemleri (Docker, Node.js) ve karşılaştığım sorunların çözümleri
- **Basit projeler:** Zamanla öğrendiklerimi kullanarak yaptığım küçük otomasyon projeleri (ayrı bir klasörde toplanacak)

## İçindekiler

### Bağlantı Kurulumları

| Servis |
|---|
| [Azure DevOps](./azure-devops-n8n) | 
| [Gmail](./gmail-n8n) | 
| [Telegram](./telegram-n8n) | 
| [Gemini](./gemini-n8n) | 
| [Google Sheets](./sheets-n8n) | 
| [Supabase](./supabase-n8n) | 
| [GitHub](./github-n8n) | 

### n8n Kurulumu

| Konu | Açıklama |
|---|---|
| [Docker & Node.js Kurulumu](./n8n-kurulum) | İki farklı kurulum yöntemi, SSL sorunu ve çözümü |

### Projeler

| Proje | Açıklama |
|---|---|
| _(yaptıkça ekleniyor)_ | Öğrendiklerimi kullanarak yaptığım küçük otomasyonlar burada olacak |

## Klasör Yapısı
```
n8n-guide/
├── README.md # bu dosya
├── n8n-kurulum/ # n8n'in kendisini kurma (Docker / Node.js)
├── azure-devops-n8n/ # servis bağlantıları
├── gmail-n8n/
├── telegram-n8n/
├── gemini-n8n/
├── sheets-n8n/
├── supabase-n8n/
├── github-n8n/
├── projeler/ # gerçek otomasyon projeleri (yakında)
└── workflows/ # n8n workflow JSON export'ları
```

## Not

Bu repo bir öğrenme günlüğü niteliğinde — burada yazanlar benim kendi kurulumlarımda işe yarayan adımlar, resmi dokümantasyonun yerini tutmaz. Güncel ve kapsamlı bilgi için her zaman ilgili servisin resmi dokümantasyonuna da bakmanı öneririm.
