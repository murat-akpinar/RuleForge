---
paths:
  - "**/Dockerfile*"
  - "**/compose*.y*ml"
  - "**/docker-compose*.y*ml"
  - "**/*.conf"
  - "nginx/**"
---

# Docker ve Ağ

## Ağ
- Host'a yalnızca nginx port açar: `ports: ["${HTTP_PORT:-80}:8080"]`. Başka hiçbir serviste `ports:` yoktur, veritabanı dahil.
- Servisler birbirine iç ağ üzerinden servis adıyla erişir. Veritabanına elle erişim: `docker compose exec db psql -U <user> <db>`.
- Tüm dış trafik nginx'ten geçer: `/` → frontend, `/api` → backend.

## İmajlar
- Sürümler sabit (`postgres:16.4-alpine`). `latest` kullanılmaz.
- nginx için `nginxinc/nginx-unprivileged` kullanılır; resmi `nginx` imajı master process'i root ile çalıştırır. Bu imaj container içinde 8080 dinler.
- Multi-stage build: derleme araçları runtime imajında kalmaz.
- Container root ile çalışmaz (`USER app`). Resmi postgres/redis imajlarına `user:` verme, kendileri düşürür.
- `.dockerignore` build context'in kökünden okunur. `build: ./backend` yazıldığında kökteki `.dockerignore` **geçerli değildir**; her bileşen kendi `.dockerignore`'unu taşır (en az `.env`, `tmp/`, testler, bağımlılık dizinleri). Kökteki dosya yalnızca `context: .` kullanıldığında devreye girer.

## Sağlık ve başlangıç sırası
- Her servisin `healthcheck`'i olur, `depends_on` → `condition: service_healthy`. Aksi halde nginx backend hazır olmadan açılır ve 502 döner.
- alpine/slim imajlarda `curl` yoktur: `wget -qO- http://localhost:8000/api/health || exit 1` ya da dile özgü bir kontrol kullan.
- Backend `/api/health` sunar: bağımlılıklarını (veritabanı vb.) kontrol eder, sürüm ve yapılandırma sızdırmaz.

## Ayarlar ve veri
- Ayarlar yalnızca env değişkenlerinden okunur. Yeni değişken eklenince `.env.example` da güncellenir.
- Kalıcı veri named volume'de tutulur. Loglar stdout'a yazılır, container içinde log dosyası tutulmaz.
- Dosya adları: `compose.yaml` (temel/prod) ve `compose.override.yaml` (dev: bind mount, hot reload). Override otomatik birleşir; `-f compose.yaml` verildiğinde yüklenmez, prod benzeri çalıştırma budur.

## nginx
- `server_tokens off;`
- `client_max_body_size` açıkça ayarlanır.
- Güvenlik header'ları: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, projeye uygun bir CSP.
- Upstream'e `X-Forwarded-For` ve `X-Forwarded-Proto` geçirilir; gerçek istemci IP'si loglanır.
- TLS gerekirse ayrı bir karar kaydı ve todo kutucuğu açılır. Lokal geliştirmede düz http kullanılır.

## Doğrulama
- `docker compose config --quiet` — sözdizimi kontrolü. Çıktı basan biçim env değerlerini çözüp gösterir, kullanma.
- `docker compose up -d && docker compose ps` — tüm servisler `healthy` olmalı.
