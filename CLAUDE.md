# CLAUDE.md — Project instructions (Claude Code)

This file defines which rule files Claude Code should load for each session.
Rules are modular: enable extra `@` lines as needed by task, or mention them in chat.

---

## Always load (baseline)

These files form the baseline **in this order**. Treat them as required for judgement, token discipline, and security.

@.rules/ai-agent.rules.md
@.rules/token-optimization.rules.md
@.rules/security.rules.md
@.rules/project-context.rules.md

---

## By task type (template — remove `#` to enable)

```text
# Backend
# @.rules/backend.rules.md

# Frontend
# @.rules/frontend.rules.md

# Database / schema / queries
# @.rules/database.rules.md

# REST / GraphQL design
# @.rules/api.rules.md

# Dockerfile / Compose
# @.rules/docker.rules.md
# @.rules/devops.rules.md

# Tests
# @.rules/testing.rules.md

# Performance
# @.rules/performance.rules.md

# Architecture / new services
# @.rules/architecture.rules.md
# @.rules/scalability.rules.md

# Errors / debugging
# @.rules/error-handling.rules.md

# Code quality / review
# @.rules/clean-code.rules.md
# @.rules/code-style.rules.md

# Git / PR workflow
# @.rules/git.rules.md

# Documentation
# @.rules/documentation.rules.md

# Planning / estimation
# @.rules/project-manager.rules.md

# Early-stage startup trade-offs
# @.rules/startup.rules.md

# Observability
# @.rules/monitoring.rules.md
```