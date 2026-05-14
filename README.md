# SGB API Bridge

[![Last sync](https://img.shields.io/endpoint?url=https://malikanmaz.github.io/sgb-api-bridge/badge.json)](https://malikanmaz.github.io/sgb-api-bridge/stats.json)
[![Delta sync](https://github.com/malikanmaz/sgb-api-bridge/actions/workflows/sync-delta.yml/badge.svg)](https://github.com/malikanmaz/sgb-api-bridge/actions/workflows/sync-delta.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

SGB (Siber Güvenlik Başkanlığı, eski **USOM** — Ulusal Siber Olaylara Müdahale Merkezi) tehdit beslemesini güvenlik duvarlarının (FortiGate, Sophos, Palo Alto, pfSense, Pi-hole, Squid) doğrudan tüketebileceği **düz metin** formatına dönüştüren açık kaynak proje.

> **Not:** USOM, 2026'da Siber Güvenlik Başkanlığı (SGB) bünyesinde yeniden yapılandırıldı. API uç noktası `www.usom.gov.tr` → `siberguvenlik.gov.tr` olarak değişti. Bu proje yeni endpoint'i kullanır.

Dört farklı dağıtım modelini destekler:

| Model | Senaryo | Kurulum dokümanı |
|---|---|---|
| **GitHub Pages** (genel) | Public CDN, sıfır kurulum, herkese açık feed | [docs/setup-github.md](docs/setup-github.md) |
| **Self-hosted GitLab** | Kurumun kendi GitLab sunucusu; internet'ten bağımsız | [docs/setup-gitlab.md](docs/setup-gitlab.md) |
| **Docker** | Tek konteyner, "indir-çalıştır-unut" | [docs/setup-docker.md](docs/setup-docker.md) |
| **Kubernetes** | CronJob + Deployment, kurumsal | [docs/setup-k8s.md](docs/setup-k8s.md) |

---

## Hızlı başlangıç (GitHub Pages — bu repo)

Aşağıdaki URL'leri firewall'una doğrudan ver:

| Tür | Adet | URL |
|---|---:|---|
| Domain (tam) | ~450K | `https://malikanmaz.github.io/sgb-api-bridge/domain-list.txt` |
| IPv4 | ~14K | `https://malikanmaz.github.io/sgb-api-bridge/ip-list.txt` |
| URL | ~7K | `https://malikanmaz.github.io/sgb-api-bridge/url-list.txt` |
| IPv6 | — | `https://malikanmaz.github.io/sgb-api-bridge/ip6-list.txt` |
| IPv6 subnet | — | `https://malikanmaz.github.io/sgb-api-bridge/ip6net-list.txt` |
| Stats | — | `https://malikanmaz.github.io/sgb-api-bridge/stats.json` |

> ⚠️ **FortiGate kullanıcıları:** FortiGate external resource başına en fazla ~131K satır destekler. Domain listesi bu sınırı aştığından aşağıdaki bölünmüş parçaları kullanın.

### FortiGate için domain listesi parçaları

Domain listesi 100.000 satırlık parçalara bölünmüştür. Her parça ayrı bir external resource olarak tanımlanır:

| Parça | URL |
|---|---|
| Part 1 | `https://malikanmaz.github.io/sgb-api-bridge/domain-list-part1.txt` |
| Part 2 | `https://malikanmaz.github.io/sgb-api-bridge/domain-list-part2.txt` |
| Part 3 | `https://malikanmaz.github.io/sgb-api-bridge/domain-list-part3.txt` |
| Part 4 | `https://malikanmaz.github.io/sgb-api-bridge/domain-list-part4.txt` |
| Part 5 | `https://malikanmaz.github.io/sgb-api-bridge/domain-list-part5.txt` |

## Hızlı başlangıç (Docker)

```bash
docker run -d \
  --name sgb-api-bridge \
  -p 8080:80 \
  -v sgb-api-bridge-data:/data \
  --restart unless-stopped \
  ghcr.io/bilsectr/sgb-api-bridge:latest
```

Detay: [docs/setup-docker.md](docs/setup-docker.md)

## Hızlı başlangıç (Kubernetes)

```bash
git clone https://github.com/malikanmaz/sgb-api-bridge.git
kubectl apply -k sgb-api-bridge/k8s/
```

Detay: [docs/setup-k8s.md](docs/setup-k8s.md)

---

## Nasıl çalışır?

- **Delta sync** — saatte bir, SGB API'sinden her tür için (`domain`, `url`, `ip`, `ip6`, `ip6net`) yalnız yeni kayıtları çeker (~1-3 dk). **Sürekli çalışan tek mekanizma budur.**
- **Full sync** — yalnızca **ilk kurulumda bir kez, elle** çalıştırılır (~10-15+ saat). Otomatik/zamanlı çalışmaz. Resume desteklidir; runner timeout'a takılırsa kaldığı yerden devam eder.
- **Liste bölme** — Delta sync tamamlandıktan sonra 100K satırı aşan listeler otomatik olarak parçalanır (`domain-list-part1.txt`, `domain-list-part2.txt` …).

SGB API kayıtları tarih sırasına göre newest-first dönüyor ve ID'ler global monoton artıyor. Delta job'ı her tür için `state/seen_ids.json`'daki `max_id`'den büyük kayıtlara ulaşana kadar sayfaları dolaşıp, bilinen kayda denk gelince durur.

> **Geçmiş veri zaten repo'da.** Bu repo'yu klonlayan / fork eden herkes, `docs/*-list.txt` ve `state/seen_ids.json` dosyalarını hazır alır. Delta sync bu noktadan devam eder — kendi ortamında full sync çalıştırman **gerekmez**. Docker imajı da bu veriyi içinde gömülü taşır (haftalık tazelenir).

## Cihaz konfigürasyon örnekleri

### FortiGate (CLI)

> Domain listesi 131K satır limitini aştığından bölünmüş parçalar kullanılmalıdır.

```
config system external-resource
    edit "SGB-Domain-P1"
        set type domain
        set resource "https://malikanmaz.github.io/sgb-api-bridge/domain-list-part1.txt"
        set refresh-rate 60
    next
    edit "SGB-Domain-P2"
        set type domain
        set resource "https://malikanmaz.github.io/sgb-api-bridge/domain-list-part2.txt"
        set refresh-rate 60
    next
    edit "SGB-Domain-P3"
        set type domain
        set resource "https://malikanmaz.github.io/sgb-api-bridge/domain-list-part3.txt"
        set refresh-rate 60
    next
    edit "SGB-Domain-P4"
        set type domain
        set resource "https://malikanmaz.github.io/sgb-api-bridge/domain-list-part4.txt"
        set refresh-rate 60
    next
    edit "SGB-Domain-P5"
        set type domain
        set resource "https://malikanmaz.github.io/sgb-api-bridge/domain-list-part5.txt"
        set refresh-rate 60
    next
    edit "SGB-IP"
        set type address
        set resource "https://malikanmaz.github.io/sgb-api-bridge/ip-list.txt"
        set refresh-rate 60
    next
end
```

Kendi base URL'ini kullanmak için (GitLab/Docker/K8s) yukarıdaki host kısmını değiştir. Tüm cihazlar (Sophos, Palo Alto, pfSense, Pi-hole, Squid) için detaylı örnekler: [docs/setup-github.md](docs/setup-github.md).

## Kendin koşturmak istersen (CLI)

```bash
git clone https://github.com/malikanmaz/sgb-api-bridge
cd sgb-api-bridge
pip install requests
python scripts/sync.py --mode delta    # ~1-3 dk — repo'daki veriden devam eder
python scripts/sync.py --mode loop     # docker için: delta'yı sürekli tetikler
python scripts/sync.py --mode full     # SADECE sıfırdan tam re-sync gerekirse (~10-15+ saat)
```

Environment variables:

| Variable | Default | Açıklama |
|---|---|---|
| `SGB_BRIDGE_ROOT` | repo kökü | State ve docs/'un kök dizini |
| `SGB_BRIDGE_DELTA_INTERVAL_SEC` | `3600` | Loop modunda delta sıklığı (sn) |
| `SGB_BRIDGE_DELTA_MAX_PAGES` | `1000` | Delta'nın tek seferde gezeceği maks. sayfa (bayat state güvenlik tavanı) |
| `SGB_BRIDGE_CHUNK_SIZE` | `100000` | Liste bölme eşiği (satır sayısı) |

## Veri kaynağı

SGB Open Threat Feed API: <https://siberguvenlik.gov.tr/api/address/index>

Kayıt kategorileri (`desc` alanı):

| Kod | Açıklama |
|---|---|
| PH | Oltalama (Phishing) |
| BP | Bankacılık - Oltalama |
| MD / MI / MU | Zararlı yazılım barındıran Domain / IP / URL |
| MC | Komuta-Kontrol Merkezi |
| CA | Siber saldırı (port tarama, brute force vb.) |

## Sorumluluk reddi

Bu proje **SGB / Siber Güvenlik Başkanlığı ile resmi bir bağlantısı olmayan**, kâr amacı gütmeyen, açık kaynak bir araçtır. Üretim sistemlerinde "as-is" kullanılır; veri doğruluğundan SGB sorumludur.

## Lisans

[MIT](LICENSE)
