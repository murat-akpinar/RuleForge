![Claude Code](https://img.shields.io/badge/claude_code-CLAUDE.md-D97757?logo=anthropic&logoColor=ffffff)
![Cursor](https://img.shields.io/badge/cursor-.cursorrules-000000?logo=cursor&logoColor=ffffff)
![Windsurf](https://img.shields.io/badge/windsurf-.windsurfrules-0EA5E9?logoColor=ffffff)
![Copilot](https://img.shields.io/badge/copilot-instructions.md-4B32C3?logo=githubcopilot&logoColor=ffffff)
![Markdown](https://img.shields.io/badge/format-Markdown-000000?logo=markdown&logoColor=ffffff)

# AI Engineering Rules

AI coding agent'larının (Claude, Cursor, Windsurf, Copilot vb.) junior gibi değil **senior engineer gibi** davranmasını sağlayan kural seti.

---

## Kurulum

`.rules/` klasörünü ve `CLAUDE.md`'yi projenizin root'una kopyalayın:

```bash
cp -r .rules /sizin-projeniz/
cp CLAUDE.md /sizin-projeniz/
```

İlk session'da Claude otomatik olarak projenizi tarar ve `project_context.md` oluşturur. Sonraki session'larda bu dosyayı okur — projeyi her seferinde yeniden taramaz.

---

## Hangi Tool, Nasıl Kullanır?

### Claude Code
`CLAUDE.md` dosyası zaten hazır — kopyalamanız yeterli. `@` referansları rule dosyalarını otomatik yükler.

### Cursor
`.cursorrules` dosyasına ekleyin:
```
Read and follow the rules in .rules/ai-agent.rules.md before every task.
For backend: also follow .rules/backend.rules.md
For frontend: also follow .rules/frontend.rules.md
For security-sensitive work: also follow .rules/security.rules.md
```

### Windsurf
`.windsurfrules` dosyasına aynı şekilde yazın.

### Cline / Roo Code
`.clinerules` dosyasına yazın.

### Copilot
`.github/copilot-instructions.md` dosyasına yazın.

---

## Hangi Dosyayı Ne Zaman Yüklersiniz?

| Görev | Yüklenecek Rule'lar |
|---|---|
| Her zaman | `ai-agent` + `token-optimization` + `security` + `project-context` |
| Backend feature | + `backend` |
| Frontend feature | + `frontend` |
| API tasarımı | + `api` + `security` |
| Veritabanı değişikliği | + `database` |
| Docker / deploy | + `docker` + `devops` |
| Kod incelemesi | + `clean-code` + `security` |
| Bug fix | + `error-handling` |
| Test yazımı | + `testing` |
| Performans sorunu | + `performance` + `database` |
| Yeni servis tasarımı | + `architecture` + `scalability` |
| Sprint / planlama | + `project-manager` |

> **İpucu:** İkiden fazla ek rule yüklemenize nadiren gerek olur. Token bütçesini koruyun.

---

## Rule Dosyaları

```
.rules/
├── ai-agent.rules.md           ← Her zaman yükle
├── token-optimization.rules.md ← Her zaman yükle
├── security.rules.md           ← Her zaman yükle
├── project-context.rules.md    ← Her zaman yükle (session bootstrap)
│
├── backend.rules.md
├── frontend.rules.md
├── database.rules.md
├── api.rules.md
├── docker.rules.md
├── testing.rules.md
├── performance.rules.md
├── architecture.rules.md
├── error-handling.rules.md
├── monitoring.rules.md
├── scalability.rules.md
├── clean-code.rules.md
├── code-style.rules.md
├── git.rules.md
├── devops.rules.md
├── documentation.rules.md
├── project-manager.rules.md
└── startup.rules.md
```

---

## Token Budget

```
Minimal  (quick tasks):   ai-agent + token-optimization + 1 domain   ~4k tokens
Standard (feature dev):   baseline + architecture + backend/frontend  ~10k tokens
Full     (system design): tüm ilgili rule'lar                         ~25k tokens
```

---

## Workflow Örnekleri

**Login feature (JWT + OAuth):**
```
Yükle: ai-agent + security + backend + api
1. Mevcut auth pattern'i analiz et
2. JWT refresh token rotasyonu uygula
3. Rate limiting ekle
4. Human review flag: "Auth değişikliği — merge öncesi inceleme gerekli"
```

**Performans sorunu (8 saniyelik checkout):**
```
Yükle: ai-agent + performance + database
1. Kör optimize etme — önce ölç
2. N+1 query kontrolü
3. Cache fırsatlarını belirle
4. Fix sonrası alert eşiği tanımla
```

**Yeni servis tasarımı (notification service):**
```
Yükle: ai-agent + architecture + backend + scalability + monitoring
1. ADR yaz: neden ayrı servis?
2. Async queue pattern tanımla
3. SLO ve alerting stratejisi belirle
4. İnsan onayı gereken kararları listele
```

**Risk taşıyan değişiklik (multi-tenant DB migration):**
```
Yükle: ai-agent + database + architecture + security
1. STOP — irreversible olarak sınıflandır
2. Additive → backfill → switch adımlarıyla plan yap
3. Her query'de tenant izolasyonu doğrula
4. Rollback script hazırla — insan onayı olmadan çalıştırma
```

---

## Multi-Agent Modeli

```
┌──────────────────────────────────────────┐
│           ORCHESTRATOR AGENT              │
│  ai-agent + project-manager               │
└──────┬──────────────┬────────────────────┘
       │              │
  ┌────▼────┐    ┌────▼────┐    ┌──────────┐
  │ARCHITECT│    │  CODER  │    │ REVIEWER │
  │ arch    │    │ backend │    │ security │
  │ scale   │    │ frontend│    │ clean-cod│
  └────┬────┘    └────┬────┘    └────┬─────┘
       └──────────────┴──────────────┘
                      │
               ┌──────▼──────┐
               │  QA AGENT   │
               │ testing     │
               │ monitoring  │
               └─────────────┘
```

Handoff payload her agent çıkışında şunları içerir:
`completed` · `artifacts` · `decisions` · `blockers` · `risks`

---

## Yeni Rule Dosyası Şablonu

```markdown
## 1. Role Definition
## 2. Core Principles
## 3. Hard Rules
## 4. Preferred Patterns
## 5. AI Decision Rules
## 6. Code Generation Standards
## 7. Anti-Patterns
## 8. Output Expectations
```

**Planlanan eklemeler:** `mobile.rules.md` · `ml.rules.md` · `data-pipeline.rules.md` · `compliance.rules.md`

---

## Startup vs Enterprise

| | Startup | Enterprise |
|---|---|---|
| Öncelik | `startup` → `ai-agent` → `security` | `security` → `architecture` → hepsi |
| Test | %60 kritik path | %80+ CI gate |
| Docs | README + runbook | ADR zorunlu, API spec |
| DevOps | PaaS kabul | Full CI/CD + security gate |
| Monitoring | Sentry + uptime | SLO + error budget |
