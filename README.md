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

## Eski kural seti

`.rules/` altındaki 22 dosyalık eski set, `@` referanslarının elle açılıp kapatıldığı döneme ait. Yerini `.claude/rules/` (yola göre otomatik yüklenen kurallar) ve skill'ler aldı. Referans olarak duruyor, yeni projelerde kullanılmıyor.
