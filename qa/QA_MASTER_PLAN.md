# QA Master Plan — GrabaKar

## 1. Scope

This QA plan covers end-to-end testing of the GrabaKar plate-engraving app — a multi-tenant, offline-first PWA built with React/Capacitor (frontend) and Django/DRF (backend). Testing targets the **Android APK running on an emulator** as the primary execution environment for frontend flows, and direct **API/DB validations** for backend consistency and background tasks (Reportes/Celery).

## 2. Test Levels

| Level | Tool | Scope |
|-------|------|-------|
| **Unit** | pytest (backend), Vitest (frontend) | Business logic, validators, services |
| **Integration** | Django test client, React Testing Library | API endpoints, component interactions |
| **E2E / Device** | QA Agent + Android Emulator (ADB) | Full user flows on actual APK |
| **Smoke** | QA Agent automated suite | Critical-path verification per build |

## 3. Quality Attributes Under Test

| Attribute | Coverage |
|-----------|----------|
| **Functional correctness** | TC_LOGIN, TC_GRABADO, TC_MULTI_VIDRIO, TC_SYNC, TC_REPORTES |
| **Offline resilience** | TC_SINCRONIZACION_OFFLINE |
| **Security** | TC_SEGURIDAD |
| **Usability / UX** | TC_USABILIDAD |
| **Performance** | TC_RENDIMIENTO |
| **Multi-tenant branding** | TC_WHITE_LABELING |
| **Edge / boundary cases** | TC_EDGE_CASES |
| **API Data Consistency** | TC_BACKEND_DATA_CONSISTENCY |
| **Reports & Celery** | TC_BACKEND_REPORTES |

## 4. Test Case Naming Convention

```
TC-<MODULE>-<NNN>   e.g. TC-LOGIN-001
```

Each test case includes:
- **ID**: unique identifier
- **Priority**: P0 (blocker), P1 (critical), P2 (major), P3 (minor)
- **Preconditions**: state required before test
- **Steps**: numbered actions
- **Expected Result**: observable outcome
- **Edge / Negative**: border-case variant

## 5. Entry & Exit Criteria

### Entry (testing can start)
- APK builds successfully from the latest branch
- Backend is accessible (local Docker Compose or staging)
- Android emulator is booted with APK installed
- Test data (tenants, users, leyes) is seeded

### Exit (release is approved)
- All P0 tests pass
- ≥ 95% of P1 tests pass
- No open P0/P1 GitHub issues
- Smoke suite passes on target build

## 6. QA Agent Architecture

```
┌─────────────────────────────────────────────────┐
│             QA TESTER AGENT                     │
│                                                 │
│  1. Boot Android Emulator (emulator @avd)       │
│  2. Install APK (adb install)                   │
│  3. Execute test scenarios via ADB/Appium       │
│  4. Capture screenshots on each step            │
│  5. Log PASS/FAIL per test case                 │
│  6. On FAIL → create GitHub Issue via gh CLI    │
│  7. Generate test run report                    │
│                                                 │
│  Tools: ADB, Appium (optional), gh CLI          │
│  Output: /qa/reports/<run-timestamp>/            │
└─────────────────────────────────────────────────┘
```

## 7. Defect Severity Mapping

| Severity | Description | SLA |
|----------|-------------|-----|
| **S1 — Critical** | App crash, data loss, auth bypass | Fix before any release |
| **S2 — Major** | Feature broken, wrong data, sync failure | Fix within 48h |
| **S3 — Minor** | Cosmetic, UX rough edge, typo | Fix before next sprint |
| **S4 — Trivial** | Enhancement suggestion | Backlog |

## 8. Test Documents Index

| Document | Description |
|----------|-------------|
| [TC_LOGIN_SESION.md](TC_LOGIN_SESION.md) | Login, session, auth, token lifecycle |
| [TC_GRABADO_PATENTE.md](TC_GRABADO_PATENTE.md) | Grabado creation and form validation |
| [TC_FLUJO_MULTI_VIDRIO.md](TC_FLUJO_MULTI_VIDRIO.md) | Multi-glass Poka-Yoke flow |
| [TC_SINCRONIZACION_OFFLINE.md](TC_SINCRONIZACION_OFFLINE.md) | Offline storage, sync, conflict resolution |
| [TC_REPORTES.md](TC_REPORTES.md) | Reports generation and download |
| [TC_WHITE_LABELING.md](TC_WHITE_LABELING.md) | Multi-tenant branding / white-label |
| [TC_SEGURIDAD.md](TC_SEGURIDAD.md) | Security and access control |
| [TC_RENDIMIENTO.md](TC_RENDIMIENTO.md) | Performance and stress |
| [TC_USABILIDAD.md](TC_USABILIDAD.md) | Usability and UX |
| [TC_EDGE_CASES.md](TC_EDGE_CASES.md) | Cross-cutting edge cases |
| [TC_BACKEND_REPORTES.md](TC_BACKEND_REPORTES.md) | Backend: Automated XLSX/CSV/JSON generation & Celery |
| [TC_BACKEND_DATA_CONSISTENCY.md](TC_BACKEND_DATA_CONSISTENCY.md) | Backend: DB constraints, isolation, API validation |
| [QA_AGENT_WORKFLOW.md](QA_AGENT_WORKFLOW.md) | Automated QA agent for emulator testing |
| [QA_AGENT_SETUP.md](QA_AGENT_SETUP.md) | Emulator and tooling setup |
| [QA_GITHUB_ISSUES.md](QA_GITHUB_ISSUES.md) | GitHub issue templates and workflow |
| [QA_CHECKLIST_RELEASE.md](QA_CHECKLIST_RELEASE.md) | Pre-release checklist |

## 9. Environments

| Env | Backend | Frontend/APK | Purpose |
|-----|---------|--------------|---------|
| **Local** | Docker Compose | `npm run dev` or APK on emulator | Development testing |
| **Staging** | GCP Cloud Run (staging) | Staging APK | Pre-release validation |
| **Production** | GCP Cloud Run (prod) | Play Store APK | Post-release smoke |
