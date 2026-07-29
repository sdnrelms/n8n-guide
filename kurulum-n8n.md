# n8n Kurulumu: Docker ve Node.js Yöntemleri

Bu belge, n8n'i kendi bilgisayarında çalıştırmak için iki farklı kurulum yöntemini anlatır: Docker ile ve Node.js (npm) ile.

## Yöntem A — Docker ile Kurulum

### Adım 1 — Docker'ın kurulu olduğundan emin ol

```bash
docker --version
```

Kurulu değilse [Docker Desktop](https://www.docker.com/products/docker-desktop/) üzerinden indirip kur.

### Adım 2 — Volume oluştur (verilerin kalıcı olması için)

Container silinse/güncellense bile workflow'ların ve credential'ların kaybolmaması için önce bir Docker volume oluştur:

```bash
docker volume create n8n_data
```

Oluşan volume'u kontrol etmek istersen:

```bash
docker volume ls
docker volume inspect n8n_data
```

### Adım 3 — Container'ı oluştur ve çalıştır

**Tek seferlik / test amaçlı (container kapanınca silinir):**

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

- `-it` → interaktif terminal modu, logları anlık görürsün
- `--rm` → container durunca otomatik silinir (ama volume kalır, veri kaybolmaz)
- `--name n8n` → container'a isim verir, sonraki komutlarda bu isimle referans verirsin
- `-p 5678:5678` → n8n'in web arayüzüne `http://localhost:5678` üzerinden erişmeni sağlar
- `-v n8n_data:/home/node/.n8n` → oluşturduğun volume'u container içindeki veri klasörüne bağlar

**Kalıcı kullanım için (arka planda, sistem yeniden başlasa bile ayakta):**

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped \
  docker.n8n.io/n8nio/n8n
```

- `-d` → detached mode, terminali kapatsan da çalışmaya devam eder
- `--restart unless-stopped` → bilgisayar yeniden başlasa bile n8n otomatik ayağa kalkar

### Adım 4 — Container'ın çalıştığını doğrula

```bash
docker ps
```

Listede `n8n` container'ını **Up** durumunda görmelisin. Tarayıcıdan `http://localhost:5678` adresini açarak da doğrulayabilirsin.

### Adım 5 — docker-compose ile kurmak istersen (alternatif, önerilen)

Tek tek `docker run` komutları yazmak yerine, `docker-compose.yml` dosyasıyla tüm ayarları tek yerde tutmak daha yönetilebilir, özellikle ortam değişkenleri (timezone, encryption key vs.) eklemeye başladığında işine yarar:

```yaml
version: "3.7"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - TZ=Europe/Istanbul
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Bu dosyayı bir klasöre `docker-compose.yml` olarak kaydet, sonra:

```bash
docker compose up -d
```

ile başlat, durdurmak için:

```bash
docker compose down
```

(`down` container'ı siler ama volume'daki veri kalır; volume'u da silmek istersen `docker compose down -v` kullanılır, dikkatli kullan.)

### Container Yönetimi — Sık Kullanılan Komutlar

```bash
docker stop n8n          # durdur
docker start n8n         # tekrar başlat
docker restart n8n       # yeniden başlat
docker logs n8n          # logları gör
docker logs -f n8n       # logları canlı takip et
docker rm n8n             # container'ı sil (volume kalır)
```

---

## Yöntem B — Node.js (npm) ile Kurulum

### Adım 1 — Node.js'in kurulu olduğundan emin ol

```bash
node --version
npm --version
```

n8n'in desteklediği Node.js sürümünü (genelde LTS) kullanmak önemli, çok eski/yeni sürümlerde uyumsuzluk çıkabiliyor.

### Adım 2 — n8n'i global olarak kur

```bash
npm install n8n -g
```

### Adım 3 — n8n'i çalıştır

```bash
n8n
```

Bu komut n8n'i başlatır ve `http://localhost:5678` üzerinden erişilebilir hale getirir.

### ⚠️ SSL Sertifika Sorunu ve `NODE_TLS_REJECT_UNAUTHORIZED`

Node.js ile kurulumda dışarıya (örn. Azure DevOps, Gmail, Gemini API'lerine) HTTP isteği atarken SSL sertifika doğrulama hatası alıyorsan, her çalıştırmadan önce şu ortam değişkenini set etmen gerekiyor:

```bash
set NODE_TLS_REJECT_UNAUTHORIZED=0
n8n
```

(Windows CMD için `set`, PowerShell için `$env:NODE_TLS_REJECT_UNAUTHORIZED=0`, macOS/Linux için `export NODE_TLS_REJECT_UNAUTHORIZED=0`)

**Bu ne yapıyor:** Node.js'in dışarı giden HTTPS isteklerinde SSL sertifikalarını doğrulamasını tamamen kapatıyor. Genelde şirket ağı/proxy'si, kurumsal SSL inceleme (SSL inspection) sistemi ya da yerel sertifika deposuyla ilgili bir uyumsuzluk olduğunda bu hata çıkar.

**⚠️ Güvenlik notu:** Bu ayar SSL doğrulamasını **tamamen** kapatır, yani n8n artık sahte/geçersiz sertifikalı sunuculara da güvenli bağlanıyormuş gibi davranır (man-in-the-middle saldırılarına açık hale gelir). Sadece **yerel geliştirme ortamında**, sorunun kaynağını bulana kadar geçici bir çözüm olarak kullanılmalı; production ortamında ya da hassas veri (API key, token) taşıyan gerçek işlerde bu ayarla çalışmak risklidir.


---

## Hangi Yöntemi Seçmeli?

| Kriter | Docker | Node.js |
|---|---|---|
| Kurulum kolaylığı | Tek komut / compose dosyası | npm + Node.js sürüm uyumu gerekir |
| İzolasyon | Sistemden bağımsız, temiz | Sistem Node.js'ine bağımlı |
| SSL sorunları | Genelde yaşanmıyor | Bazı ağlarda `NODE_TLS_REJECT_UNAUTHORIZED` gerekebiliyor |
| Güncelleme | `docker pull` ile kolay | `npm update -g n8n` (global) |
| Kaynak kullanımı | Biraz daha ağır (container overhead) | Daha hafif |
| Ortak/kişisel olmayan bilgisayar | Container izole, sistem paketlerine dokunmaz | Global kurulum sistemi etkiler, lokal kurulum daha temiz kalır |
