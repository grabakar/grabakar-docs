# Technical Assessment — GrabaKar

**Date:** 2026-03-13
**Last updated:** 2026-03-13
**Scope:** Backend, Frontend, Infrastructure, Documentation, Workflows
**Project:** GrabaKar — Offline-first B2B vehicle plate engraving PWA

---

## Executive Summary

GrabaKar is a well-architected project with clean separation of concerns across four repositories (backend, frontend, docs, infra). The documentation is comprehensive. The backend service layer, tenant isolation pattern, and offline-first data model are solid foundations.

An initial assessment identified critical security gaps, bugs in both stacks, and infrastructure that was not production-ready. A first round of remediation has resolved 15 issues including the most critical security gaps (default permissions, secret key handling), all known bugs (empty CSV export, render-time side effects, infinite recursion), and infrastructure hardening (restart policies, port binding, health checks). Remaining open items are tracked in Section 7.

### Overall Scorecard

| Area | Grade | Previous | Verdict |
|------|-------|----------|---------|
| Project Organization | **A-** | A- | Clean multi-repo structure, thorough docs, stale references fixed |
| Backend Architecture | **A-** | B+ | Good service layer, proper tenant mixin, DRY constants extracted |
| Frontend Architecture | **B+** | B | Solid offline-first foundation, key bugs fixed |
| Security | **C** | D+ | Critical backend gaps fixed; frontend CSP + token encryption still open |
| Testing | **C+** | C+ | Backend decent (67 tests), frontend gaps remain |
| Infrastructure | **C+** | C- | Restart policies, port binding, health checks added; production hardening still needed |
| Documentation | **A-** | B+ | Broken references fixed, assessment added |
| CI/CD & Workflows | **D** | D | No workflows found in workspace (unchanged) |

---

## 1. Project Organization

### Structure

```
grabado-patente-app/
├── grabakar-backend/    Django 5.1 + DRF + Celery + PostgreSQL
├── grabakar-frontend/   React 19 + Vite 7 + Dexie + Capacitor
├── grabakar-docs/       40+ Markdown files across 7 sections
└── grabakar-infra/      Docker Compose orchestrator + scripts
```

### Strengths

- **Multi-repo separation** is appropriate for the team size and deployment model.
- **Documentation-first approach**: Product vision, features, API contracts, data models, and security are all documented before or alongside implementation.
- **Agent/prompt system**: Dedicated agent rules for frontend, backend, review, and CI ensure consistent AI-assisted development.
- **PDCA methodology**: Documented and embedded in agent rules.

### Issues

| ID | Issue | Severity | Status |
|----|-------|----------|--------|
| ORG-1 | ~~Broken doc references: `backlog/`, `fases/` folders don't exist~~ | Medium | **FIXED** — Updated to `features/`, `planificacion/BACKLOG.md` |
| ORG-2 | Missing `AGENT_REPORTES.md` referenced in status report | Low | Open |
| ORG-3 | `planificacion/ciclos/` referenced but doesn't exist | Low | Open |
| ORG-4 | `Plate-number-app.md` duplicates content from 15+ other docs | Low | Open |
| ORG-5 | Sync payload key inconsistency: `batch` vs `registros` | Medium | Open |
| ORG-6 | Patente max length: 8 vs 10 across docs | Medium | Open |
| ORG-7 | No `.github/workflows` present in any repo within workspace | High | Open |

---

## 2. Backend Assessment

### Architecture (Grade: A-)

Single Django app (`api/`) with proper separation:

```
api/
├── models.py          5 models + DEFAULT_VIDRIOS_CONFIG constant, UUID PKs, tenant FK
├── views/             4 viewsets/views (stub views.py removed)
├── serializers/       3 serializer modules
├── services/          3 service modules (report, XLSX, sync)
├── tasks/             2 Celery task modules
├── tests/             7 test modules (~67 tests)
├── utils/             Custom exception handler
└── permissions.py     IsTenantAdmin permission
```

**Strengths:**
- Business logic in services, views stay thin.
- `TenantQuerySetMixin` provides consistent tenant isolation.
- Custom exception handler standardizes error responses in Spanish.
- Good model design: `TextChoices` enums, UUID fields, proper indexes.
- Split settings: `base.py`, `local.py`, `production.py`, `test.py`.
- `DEFAULT_PERMISSION_CLASSES: [IsAuthenticated]` enforced globally.
- `SECRET_KEY` fails loudly if env var is missing (dev key set only in `local.py`).

### Security Issues

| ID | Issue | Severity | Status |
|----|-------|----------|--------|
| SEC-B1 | ~~No `DEFAULT_PERMISSION_CLASSES` in DRF config~~ | Critical | **FIXED** — Added `IsAuthenticated` as default |
| SEC-B2 | ~~`GrabadoViewSet` has no `permission_classes`~~ | Critical | **FIXED** — Covered by global default (SEC-B1) |
| SEC-B3 | ~~`SECRET_KEY` fallback to `'change-me'`~~ | Critical | **FIXED** — `base.py` raises `ValueError`; `local.py` sets dev-only default |
| SEC-B4 | Mass assignment in `SyncService._crear_grabado` | **High** | Open |
| SEC-B5 | No rate limiting on login | **High** | Open |
| SEC-B6 | Platform report leaks cross-tenant data | **High** | Open |
| SEC-B7 | String datetime comparison in sync | **Medium** | Open |

### Bugs

| ID | Bug | Severity | Status |
|----|-----|----------|--------|
| BUG-B1 | ~~`ReporteMensualView._respuesta_csv` writes headers only, no data rows~~ | High | **FIXED** — Added data row loop |
| BUG-B2 | ~~ASGI settings module points to empty `config.settings`~~ | Medium | **FIXED** — Changed to `config.settings.local` |
| BUG-B3 | ~~Stub `api/views.py` conflicts with `api/views/` package~~ | Low | **FIXED** — File deleted |

### Code Quality Issues

| ID | Issue | Status |
|----|-------|--------|
| CQ-B1 | No `transaction.atomic()` in batch sync — partial saves possible | Open |
| CQ-B2 | ~~`vidrios_config` default copy-pasted in 3 places~~ | **FIXED** — Extracted to `DEFAULT_VIDRIOS_CONFIG` in `models.py` |
| CQ-B3 | Deprecated `unique_together` — Django 5 recommends `UniqueConstraint` | Open |
| CQ-B4 | No `created_at`/`updated_at` audit timestamps on any model | Open |
| CQ-B5 | `factory_boy` and `Faker` installed but never used | Open |
| CQ-B6 | `django-ratelimit` installed but never used | Open |
| CQ-B7 | Purge task is tenant-unscoped with hard-coded 30-day retention | Open |
| CQ-B8 | `psycopg2-binary` used instead of `psycopg2` (not recommended for production) | Open |

### Testing (Grade: B-)

- **67 test cases** across 7 files covering auth, CRUD, filters, sync, reports, health.
- Tenant isolation verified in grabados, sync, and reports.
- **Gaps:** No tests for Celery tasks, custom exception handler, admin panel, concurrent sync, monthly CSV export (would have caught BUG-B1).
- Fixture duplication across test files instead of using shared `conftest.py`.

---

## 3. Frontend Assessment

### Architecture (Grade: B+)

```
src/
├── components/    5 UI components
├── contexts/      AuthContext, ThemeContext
├── db/            Dexie schema + helpers
├── hooks/         useConnectivity, useSyncStatus
├── pages/         4 pages
├── services/      5 services (Auth, Sync, Connectivity, Print, Scanner)
├── types/         3 type modules
└── utils/         3 utility modules
```

**Strengths:**
- Clean separation: services for logic, contexts for state, db for persistence.
- Dexie/IndexedDB for offline-first storage with `useLiveQuery` for reactive UI.
- Exponential backoff in SyncManager (x3, max 30min) with bounded auth retries.
- ConnectivityService pings actual server health endpoint.
- Lean dependency footprint (8 runtime deps).
- VIN validation checks both length and character set (ISO 3779).

### Security Issues

| ID | Issue | Severity | Status |
|----|-------|----------|--------|
| SEC-F1 | Tokens stored as plaintext in IndexedDB | **Critical** | Open |
| SEC-F2 | No Content Security Policy (CSP) | **Critical** | Open |
| SEC-F3 | No HTTPS enforcement / `upgrade-insecure-requests` | **High** | Open |
| SEC-F4 | ~~VIN validation only checks length, not character set~~ | Medium | **FIXED** — Now validates against ISO 3779 charset |

### Bugs

| ID | Bug | Severity | Status |
|----|-----|----------|--------|
| BUG-F1 | ~~`navigate()` called during render in `LoginPage`~~ | High | **FIXED** — Replaced with `<Navigate>` component |
| BUG-F2 | Non-atomic grabado + impresiones save — crash = orphaned record | **High** | Open |
| BUG-F3 | ~~Potential infinite recursion on 401 in `SyncManager.enviarBatch`~~ | High | **FIXED** — Added `authRetries` counter (max 1) |
| BUG-F4 | Patente normalization triggers double-render via watch/setValue loop | **Low** | Open |
| BUG-F5 | ~~Duplicate test file (`database.test.ts` in two locations)~~ | Low | **FIXED** — Removed simpler duplicate, kept `__tests__/` version |

### Code Quality Issues

| ID | Issue | Status |
|----|-------|--------|
| CQ-F1 | `window.confirm()` on mobile — blocks thread, poor UX | Open |
| CQ-F2 | No React Error Boundary — unhandled error = white screen | Open |
| CQ-F3 | All CSS in single `index.css` (72 lines) | Open |
| CQ-F4 | `Configuracion.valor` typed as `unknown` — forces `as` casts everywhere | Open |
| CQ-F5 | Emoji icons instead of proper icon library | Open |
| CQ-F6 | No loading/submitting state on forms — double-submit possible | Open |
| CQ-F7 | API base URL hardcoded as `'/api/v1'` — breaks Capacitor native builds | Open |

### PWA Readiness (Grade: D)

| Requirement | Status |
|-------------|--------|
| Service worker (Workbox) | Present |
| Static asset precaching | Present |
| Config API runtime caching | Present |
| Web app manifest (name, icons, display, colors) | **Missing** |
| Offline fallback page | **Missing** |
| Service worker update notification | **Missing** |
| Installability criteria (Chrome) | **Not met** |

### Testing (Grade: D+)

- 6 test files covering database, AuthService, SyncManager, patente utils, VIN validation, deviceId.
- **Zero component/page tests** despite `@testing-library/react` being installed.
- Two SyncManager tests assert `expect(true).toBe(true)` — testing nothing.
- No integration tests, no E2E tests.

### Offline-First Gaps

| Gap | Impact |
|-----|--------|
| No conflict resolution for same patente on two devices | Data inconsistency |
| Records with `estado_sync: 'error'` are never retried | Data loss |
| No sync-down mechanism | Stale local config |
| No storage quota management | Browser may evict IndexedDB |

---

## 4. Infrastructure Assessment

### Docker Compose (Grade: C+)

**Strengths:**
- Multi-stage Dockerfile minimizes image size.
- PostgreSQL health check present.
- Redis health check added.
- Named volumes for data persistence.
- Source mounts + `--reload` for dev hot-reload.
- `restart: unless-stopped` on all services.
- PostgreSQL and Redis bound to `127.0.0.1` only.
- `.dockerignore` excludes `.git/`, `.env`, `__pycache__`, test artifacts.

**Remaining Issues:**

| ID | Issue | Severity | Status |
|----|-------|----------|--------|
| INF-1 | ~~No restart policies on any service~~ | High | **FIXED** — `restart: unless-stopped` added to all services |
| INF-2 | Migration race: Celery starts before Django migrations finish | **High** | Open |
| INF-3 | ~~Redis exposed on `0.0.0.0:6379`~~ | Critical | **FIXED** — Bound to `127.0.0.1`, health check added |
| INF-4 | ~~PostgreSQL exposed on `0.0.0.0:5432`~~ | High | **FIXED** — Bound to `127.0.0.1` |
| INF-5 | Backend container runs as root | **High** | Open |
| INF-6 | No resource limits (memory/CPU) on any service | **Medium** | Open |
| INF-7 | `npm install` runs on every frontend container start | **Medium** | Open |
| INF-8 | ~~No `.dockerignore`~~ | Medium | **FIXED** — Created for backend |
| INF-9 | ~~Redis has no health check~~ | Medium | **FIXED** — `redis-cli ping` health check added |
| INF-10 | No logging driver or rotation configured | **Low** | Open |

### Environment Management (Grade: C-)

| ID | Issue | Status |
|----|-------|--------|
| ENV-1 | `.env` is a verbatim copy of `.env.example` — `SECRET_KEY=change-me-in-production` | Open |
| ENV-2 | Infra `.env.example` missing email, storage, and monitoring variables from backend | Open |
| ENV-3 | No provisions for `SENTRY_DSN`, `LOG_LEVEL`, CSRF trusted origins, or cloud storage | Open |

### Scripts (Grade: B-)

| Script | Issues |
|--------|--------|
| `setup.sh` | Suppresses git errors with `2>/dev/null`, no branch pinning |
| `update.sh` | Suppresses errors, sequential pulls |
| `review_precheck.sh` | Always exits 0 — cannot be used as CI gate |

### Production Readiness (Grade: F)

Missing entirely: reverse proxy (Nginx), TLS termination, monitoring/alerting, database backups, secrets management, blue-green deployment, log aggregation.

---

## 5. CI/CD & Workflows

**Grade: D**

No `.github/workflows` directory exists in any repository within this workspace. The documentation (`TESTING.md`, `DEPLOYMENT.md`) describes GitHub Actions workflows, but they are not implemented:

- `TESTING.md` contains example CI YAML for ruff + pytest and Vitest
- `DEPLOYMENT.md` describes staging/production GCP deployment
- `AGENT_CI.md` defines a CI agent prompt

**What's missing:**
- Lint + test workflow (backend)
- Lint + test + build workflow (frontend)
- Docker build + push workflow
- Deployment workflow (staging, production)
- Pre-commit hooks
- Branch protection rules
- Automated code review

---

## 6. Documentation Health

### Strengths
- 40+ well-structured Markdown files covering product, features, architecture, API contracts, data models, security, deployment, and testing.
- Agent prompt system with implementation plans.
- Backend status report with completion tracking.
- Technical assessment document with remediation tracking.

### Issues

| Category | Count | Examples | Status |
|----------|-------|---------|--------|
| Broken folder references | ~~3~~ 1 | `planificacion/ciclos/` | 2 of 3 **FIXED** |
| Field name inconsistencies | 3 | `batch`/`registros`, `orden`/`numero_vidrio`, `Ley`/`LeyCaso` | Open |
| Value inconsistencies | 2 | Patente max length (8 vs 10), IndexedDB table names | Open |
| Missing referenced files | 1 | `AGENT_REPORTES.md` | Open |
| Redundant content | 1 | `Plate-number-app.md` overlaps with 15+ files | Open |

---

## 7. Remaining Action Items

### P0 — Critical (Fix Before Any Deployment)

| # | Action | Component |
|---|--------|-----------|
| 1 | Add CSP headers to `index.html` and/or via Vite plugin | Frontend |
| 2 | Encrypt tokens in IndexedDB or use Capacitor secure storage | Frontend |
| 3 | Add PWA manifest (name, icons, display, theme_color) | Frontend |

### P1 — High (Fix Before Production)

| # | Action | Component |
|---|--------|-----------|
| 4 | Whitelist fields in `SyncService._crear_grabado` | Backend |
| 5 | Add rate limiting to `LoginView` | Backend |
| 6 | Restrict `ReportePlataformaView` to superadmin | Backend |
| 7 | Wrap sync batch in `transaction.atomic()` | Backend |
| 8 | Make grabado + impresiones save atomic (Dexie transaction) | Frontend |
| 9 | Add React Error Boundary | Frontend |
| 10 | Add non-root USER in Dockerfile | Infrastructure |
| 11 | Separate migration step from gunicorn startup | Infrastructure |
| 12 | Create GitHub Actions CI workflows for both stacks | CI/CD |

### P2 — Medium (Improve Quality)

| # | Action | Component |
|---|--------|-----------|
| 13 | Add `created_at`/`updated_at` to all models | Backend |
| 14 | Parse datetimes properly in sync service | Backend |
| 15 | Replace `window.confirm()` with custom modal | Frontend |
| 16 | Add component/page tests with RTL | Frontend |
| 17 | Make API URL configurable via `VITE_API_BASE_URL` | Frontend |
| 18 | Add resource limits to Docker services | Infrastructure |
| 19 | Fix `review_precheck.sh` to exit non-zero on findings | Infrastructure |
| 20 | Align sync payload key (`batch` vs `registros`) across docs | Documentation |

### P3 — Low (Clean Up)

| # | Action | Component |
|---|--------|-----------|
| 21 | Replace `unique_together` with `UniqueConstraint` | Backend |
| 22 | Remove unused `factory_boy` and `Faker` (or use them) | Backend |
| 23 | Replace emoji icons with proper icon library | Frontend |
| 24 | Fix `expect(true).toBe(true)` placeholder tests | Frontend |
| 25 | Add offline fallback page for PWA | Frontend |

---

## 8. Security Summary

### Attack Surface (Post-Remediation)

```
┌─────────────────────────────────────────────────┐
│                  INTERNET                        │
└──────────────────┬──────────────────────────────┘
                   │ No TLS termination
                   │ No WAF / rate limiter
┌──────────────────▼──────────────────────────────┐
│  Django (port 8000)                             │
│  ┌────────────────────────────────────────────┐ │
│  │ ✅ IsAuthenticated enforced globally       │ │
│  │ ✅ SECRET_KEY fails if env var missing     │ │
│  │ ⚠  LoginView — no rate limiting           │ │◄── SEC-B5
│  │ ⚠  SyncService — mass assignment          │ │◄── SEC-B4
│  │ ⚠  PlatformReport — cross-tenant leak     │ │◄── SEC-B6
│  └────────────────────────────────────────────┘ │
└──────────────────┬──────────────────────────────┘
┌──────────────────▼──────────────────────────────┐
│  ✅ PostgreSQL (127.0.0.1:5432 only)           │
│  ✅ Redis (127.0.0.1:6379 only, health check)  │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  Frontend PWA (user device)                     │
│  ┌────────────────────────────────────────────┐ │
│  │ ⚠  IndexedDB — plaintext JWT tokens       │ │◄── SEC-F1
│  │ ⚠  No CSP headers                         │ │◄── SEC-F2
│  │ ⚠  No HTTPS enforcement                   │ │◄── SEC-F3
│  │ ✅ VIN validation: length + charset        │ │
│  │ ✅ SyncManager: bounded auth retries       │ │
│  └────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### OWASP Top 10 Mapping

| OWASP Category | Status | Issues |
|----------------|--------|--------|
| A01 Broken Access Control | **Improved** — ~~SEC-B1, SEC-B2~~ fixed; SEC-B6 remains | SEC-B6 |
| A02 Cryptographic Failures | **Improved** — ~~SEC-B3~~ fixed; SEC-F1 remains | SEC-F1 |
| A03 Injection | Partially mitigated | SEC-B4 (mass assignment) |
| A04 Insecure Design | **Needs work** | No rate limiting, no error boundary |
| A05 Security Misconfiguration | **Improved** — ~~INF-3, INF-4~~ fixed; SEC-F2 remains | SEC-F2 |
| A06 Vulnerable Components | Low risk | Dependencies are recent versions |
| A07 Auth Failures | **Vulnerable** | SEC-B5 (no brute-force protection) |
| A08 Data Integrity Failures | Moderate | CQ-B1 (no atomic transactions) |
| A09 Logging & Monitoring | **Missing** | No logging infrastructure |
| A10 SSRF | Low risk | No user-controlled URLs |

---

## 9. Credential Exposure

**WARNING:** The file `/Users/franciscocollarte/Documents/cursor-workflow/mcp.json.backup` contains:
- A real Home Assistant JWT token (`HA_TOKEN`)
- A real Exa API key

This file should be excluded from version control and the credentials should be rotated if the file has been shared.

---

## 10. Remediation Log

| Date | Items Fixed | Summary |
|------|-------------|---------|
| 2026-03-13 | SEC-B1, SEC-B2, SEC-B3, BUG-B1, BUG-B2, BUG-B3, CQ-B2, SEC-F4, BUG-F1, BUG-F3, BUG-F5, INF-1, INF-3, INF-4, INF-8, INF-9, ORG-1 | Added `DEFAULT_PERMISSION_CLASSES`, hardened `SECRET_KEY`, fixed monthly CSV export, fixed ASGI path, deleted stub `views.py`, extracted `vidrios_config` constant, fixed VIN validation charset, fixed `LoginPage` render-time navigate, added SyncManager retry bound, removed duplicate test file, added restart policies + `127.0.0.1` port binding + Redis health check + `.dockerignore`, fixed broken doc references in agent prompts |

---

*Assessment generated on 2026-03-13. Review periodically as the project evolves.*
