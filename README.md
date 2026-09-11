![Claude Code](https://img.shields.io/badge/claude_code-2.1%2B-D97757?logo=anthropic&logoColor=ffffff)
![Docker](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=ffffff)
![git-cliff](https://img.shields.io/badge/changelog-git--cliff-000000)

# RuleForge

Boş bir dizine kopyalanıp Claude Code ile projeye başlamak için hazırlanmış şablon. Amaç: **önce kararlar, sonra iskelet, en son kod.**

## Kullanım

```bash
cp -r ~/GIT/RuleForge/template/. ~/yeni-projem/
cd ~/yeni-projem && claude
```

`docs/PROJECT.md` boş olduğu için Claude `kurulum` skill'ini çalıştırır:

```
keşif → kararlar (ADR) → iskelet → todo.md → ilk commit → dur
```

Uygulama kodu bu aşamada yazılmaz. Container'ları ayağa kaldırmak bile `docs/todo.md`'nin ilk kutucuğudur.

## İçindekiler

| Dosya | Ne yapar |
|---|---|
| `CLAUDE.md` | Her oturumda yüklenen proje kuralları: dil, iş akışı, commit protokolü, sınırlar |
| `.claude/settings.json` | Claude imzasını kapatır, tehlikeli komutları engeller veya onaya bağlar |
| `.claude/rules/docker.md` | Yalnızca Dockerfile / compose / nginx dosyalarına dokunulurken yüklenir |
| `.claude/rules/security.md` | Yalnızca kaynak dosyalara dokunulurken yüklenir: OWASP + SonarQube kuralları |
| `.claude/skills/kurulum/` | Karar fazını yürüten kurulum akışı |
| `docs/PROJECT.md` | Amaç, v1 kapsamı, bilerek kapsam dışı bırakılanlar |
| `docs/MAP.md` | Dizinler, feature indeksi, ortak yardımcılar |
| `docs/todo.md` | Fazlar ve kabul kriterli kutucuklar |
| `docs/decisions/` | Altyapı kararları (ADR) |
| `cliff.toml` | CHANGELOG üretimi, Türkçe grup başlıkları |
| `tmp/` | Kaynak materyal. Git'e ve imaja girmez, Claude yalnızca okur |

## Çalışma düzeni

- **Kutucuk döngüsü:** tek kutucuk → doğrula (build, kabul kriteri, test, lint) → `[x]` → `MAP.md` güncelle → CHANGELOG ile tek commit → push → durum özeti → `/clear`.
- **Faz kapanışı:** her fazın son kutucuğu güvenlik ve test kapanışıdır. Kapsam, bağımlılık taraması, imaj taraması, sır sızıntısı kontrolü geçmeden sonraki faza geçilmez.
- **Docker:** host'a yalnızca nginx port açar, tüm trafik oradan geçer. Sabit imaj sürümü, multi-stage build, root olmayan kullanıcı, her serviste healthcheck.
- **Kimlik:** commit'ler yalnızca senin git kimliğinle atılır. `Co-Authored-By`, `Generated with Claude` ve `Claude-Session` satırları çıkmaz; kurulum ilk commit'ten sonra bunu doğrular.
- **Kod:** fonksiyonel ve tekrarsız. Yorum yalnızca "neden" açık değilse. Feature blokları `--- START FEATURE: <ad> ---` ile işaretlenir, `MAP.md` bu adlara referans verir.

## Gereksinimler

- Claude Code 2.1+ (`.claude/rules/` ve `paths:` desteği için)
- Docker Compose v2+
- [git-cliff](https://git-cliff.org)

## Doğrulama

Şablon boş bir dizine kopyalanıp uçtan uca çalıştırıldı: backend (FastAPI) + db (PostgreSQL) + nginx, kurulum akışının ürettiği yapıyla.

_Test ortamı: Docker 29.7.2 · Compose v5.5.1 · git-cliff 2.14.1 · Claude Code 2.1.267 · 2026-09-12_

| Kontrol | Sonuç |
|---|---|
| Kopyalama, gizli dosyalar dahil | 14 dosya eksiksiz |
| `.env` ve `tmp/` git'e girmiyor | tüm geçmişte iz yok |
| `docker compose config --quiet` | temiz |
| Başlangıç sırası db → backend → nginx | healthcheck zinciriyle sırayla açıldı |
| nginx üzerinden `/api/health` | `200`, `{"status":"ok"}` |
| backend ve db host'a kapalı | `:8000` ve `:5432` erişilemez, yalnızca nginx portu açık |
| Root olmayan kullanıcı | backend `app`, nginx `nginx` |
| Güvenlik header'ları | `nosniff`, `DENY`, `Referrer-Policy` geldi |
| Dev override: bind mount + hot reload | kod değişti, yanıt yeniden başlatmadan güncellendi |
| `-f compose.yaml` (prod benzeri) | override yüklenmedi |
| Kutucuk akışı: CHANGELOG + commit | tek commit, doğru kimlik, Claude izi yok |
| `cliff.toml` çıktısı | Türkçe gruplarla üretildi, `--bumped-version` → `0.1.0` |

Test sırasında bulunup düzeltilen iki hata:

1. **Boş depoda ilk commit kırıktı.** `git cliff`, henüz commit'i olmayan depoda `reference 'refs/heads/main' not found` ile düşüyordu. İlk commit artık ters sırada: önce commit, sonra changelog, sonra `--amend`. Ayrıca `git init -b main` eklendi.
2. **Kökteki `.dockerignore` alt build context'lere uygulanmıyordu.** `build: ./backend` yazıldığında Docker dosyayı `backend/` içinde arar; `.env` veya `tmp/` imaja sızabilirdi. Artık her bileşen kendi `.dockerignore`'unu taşıyor.

Hata değil ama bilinmesi gereken: `server_tokens off` yalnızca sürüm numarasını gizler, `Server: nginx` başlığı kalır. Kaldırmak `headers-more` modülü ister, standart imajda yoktur.

## Geçmiş

Depo önceden `.rules/` altında 22 dosyalık, `CLAUDE.md`'deki `@` referanslarının elle açılıp kapatıldığı bir kural setiydi. Yerini yola göre otomatik yüklenen `.claude/rules/` dosyaları ve skill'ler aldı; eski set kaldırıldı. Git geçmişinde duruyor.
