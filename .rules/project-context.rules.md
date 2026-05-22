# Project Context Rules

## 1. Role Definition
**Project Context Manager**
At the start of every session, check for a cached project context file before touching any source code. If it exists, read it and proceed. If it doesn't, scan the project once, write the file, and use it for the rest of the session. Never re-derive what is already documented.

---

## 2. Core Principles
- **Scan Once, Reuse Always**: Project structure is stable across sessions. Derive it once, persist it.
- **Context File is Ground Truth**: Treat `project_context.md` as the authoritative project summary until proven stale.
- **Minimal Scan Depth**: Scan only what's needed to understand the project — not every file, not every line.
- **Keep it Current**: Update the file when architecture changes, not on every session.
- **Human-Readable**: The context file must be readable by both agents and human developers.

---

## 3. Hard Rules
- **Never scan the full project if `project_context.md` exists.** Read the file, trust it, proceed.
- **Never skip creating `project_context.md`** on first session. The next session depends on it.
- **Never include file contents** in `project_context.md` — only paths, patterns, and summaries.
- **Never include generated files, lock files, or build artifacts** in the context.
- **Never let `project_context.md` exceed 150 lines.** Compress; don't enumerate.
- **Always note the creation date** so staleness can be detected.
- **Update the file immediately** when a significant architectural change is made in the current session.

---

## 4. Session Bootstrap Protocol

### On Every Session Start
```
1. CHECK: Does `project_context.md` exist in the project root?

   YES → Read it (costs ~1k tokens)
         Proceed with task. Done.

   NO  → Run First-Session Scan (see Section 5)
         Write `project_context.md`
         Use it for the rest of the session
```

### Staleness Detection
```
Treat `project_context.md` as stale if:
  - The date is older than 30 days AND the user mentions recent architectural changes
  - Key files listed no longer exist
  - Tech stack in the file doesn't match package.json / pyproject.toml / go.mod

If stale: regenerate the file, update the date.
If uncertain: trust the file, flag the uncertainty to the user.
```

---

## 5. First-Session Scan Protocol

### Scan Order (stop when you have enough)
```
1. Root directory listing (1 level deep)
2. Package manifest: package.json / pyproject.toml / go.mod / Cargo.toml / pom.xml
3. Entry point: main.ts / index.ts / main.py / cmd/ / app.py (whichever applies)
4. Config files: .env.example / docker-compose.yml / Dockerfile / .github/workflows/
5. Source structure: src/ / app/ / lib/ / internal/ (1-2 levels deep)
6. Test structure: test/ / tests/ / __tests__/ / spec/ (top level only)
7. README.md (first 60 lines only)
```

### What to Extract
```
From package manifest:  tech stack, framework, key dependencies, scripts
From entry point:       app bootstrap pattern, DI setup, middleware chain
From config files:      required env vars, infrastructure dependencies
From source structure:  module boundaries, naming conventions, layer pattern
From README:            setup commands, project purpose, any special notes
```

---

## 6. `project_context.md` File Format

```markdown
# Project Context
_Generated: YYYY-MM-DD | Updated: YYYY-MM-DD_

## Stack
- Language: [e.g. TypeScript 5.x]
- Runtime: [e.g. Node.js 20, Bun]
- Framework: [e.g. NestJS 10, Express, FastAPI]
- Database: [e.g. PostgreSQL via Prisma ORM]
- Key libs: [e.g. zod, bull, passport-jwt]

## Structure
```
src/
  modules/      # feature modules (NestJS pattern)
  common/       # shared utilities, guards, decorators
  config/       # env validation, app config
tests/          # integration tests (jest)
prisma/         # schema + migrations
```

## Entry Points
- App bootstrap: `src/main.ts`
- Module root: `src/app.module.ts`
- CLI (if any): `src/cli.ts`

## Key Commands
- Dev: `npm run start:dev`
- Test: `npm run test`
- Build: `npm run build`
- Migrate: `npx prisma migrate dev`

## Patterns & Conventions
- [e.g. Repository pattern: service → repository → prisma]
- [e.g. DTOs validated with zod at controller boundary]
- [e.g. snake_case for DB columns, camelCase for TS]
- [e.g. Feature flags via env vars, no runtime toggle system]

## Environment
Required env vars (see `.env.example`):
- `DATABASE_URL`, `JWT_SECRET`, `REDIS_URL`

## Constraints & Gotchas
- [e.g. Multi-tenant: every query must include `tenant_id`]
- [e.g. Migrations run manually before deploy, not auto]
- [e.g. Rate limiter is per-IP only — no per-user limit yet]

## Incomplete / In Progress
- [e.g. Notification module not yet wired to email provider]
```

---

## 7. Update Triggers

Update `project_context.md` during the current session when:
```
✓ A new module or service is added
✓ A new major dependency is introduced
✓ The database ORM or migration strategy changes
✓ A new required environment variable is added
✓ The directory structure changes significantly
✓ A previously listed "in progress" item is completed
```

Do NOT update for:
```
✗ Bug fixes that don't change architecture
✗ Adding a new endpoint within an existing module
✗ Refactoring within a single file
✗ Test additions
```

---

## 8. Integration with Other Rules

```
Token Optimization (token-optimization.rules.md):
  project_context.md IS the "grep before read" at project level.
  Reading it replaces scanning 10-20 files. Always read it first.

AI Agent (ai-agent.rules.md):
  Step 1 of "Before Starting Any Task" (LOCATE) is satisfied by reading
  project_context.md before touching any source file.

Security (security.rules.md):
  project_context.md must never contain secrets, credentials, or
  environment variable values — only key names (e.g., `JWT_SECRET`).
```

---

## 9. Anti-Patterns
- **Blind Scan Every Session**: Re-reading package.json, src/, README on every session when `project_context.md` exists.
- **Over-detailed Context**: Listing every file path in the project. Keep it structural, not exhaustive.
- **Stale Trust**: Using a 6-month-old context file without checking if key files still exist.
- **Secrets in Context**: Writing `JWT_SECRET=abc123` instead of just `JWT_SECRET` (required).
- **Agent-Only Format**: Writing the file in a format only agents can parse. Humans read this too.
- **Never Updating**: Leaving "In Progress" items in the file after they're completed for months.
