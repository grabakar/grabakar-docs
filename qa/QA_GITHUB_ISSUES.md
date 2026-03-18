# QA GitHub Issues -- Templates & Reporting Workflow

This document defines how the QA Agent (and manual testers) report defects as GitHub issues, including templates, labels, and workflow.

**Repository**: [`grabakar/grabakar-frontend`](https://github.com/grabakar/grabakar-frontend) and [`grabakar/grabakar-backend`](https://github.com/grabakar/grabakar-backend)

**Related documents**:
- [QA_IN_DEVICE_RUNBOOK.md](QA_IN_DEVICE_RUNBOOK.md) -- the "Bug Reporting via GitHub Issues" section is the canonical step-by-step procedure.
- [QA_MASTER_PLAN.md](QA_MASTER_PLAN.md) -- defect severity mapping and exit criteria.
- [QA_AGENT_WORKFLOW.md](QA_AGENT_WORKFLOW.md) -- automated agent execution flow.

**Product behavior (Nuevo Grabado)**: El campo "Ley / Caso" no se muestra en el formulario. Se asigna automáticamente con la primera ley de `leyes_activas`. Los escenarios con `ley_caso_id` inválido (p. ej. en sync) corresponden a payload manipulado o backend, no a selección del usuario.

---

## Issue Templates (installed in repo)

Two YAML-based issue form templates live in the repo at `.github/ISSUE_TEMPLATE/`:

| Template file | Use case | Default labels |
|---------------|----------|----------------|
| `qa-auto-bug.yml` | Defect found by the QA tester agent during in-device testing | `bug`, `qa-auto` |
| `qa-manual-bug.yml` | Defect found during manual testing on device or emulator | `bug`, `qa-manual` |

A `config.yml` in the same directory disables blank issues and links to the QA documentation.

### QA Agent Automated Bug Report (summary)

Title format: `[QA-AUTO] TC-<ID>: <brief description in Spanish>`

Fields:
- Test Case ID, Module, Priority, Severity, Build, Emulator/Device.
- Steps to Reproduce (numbered).
- Expected Result.
- Actual Result.
- Evidence (screenshots, UI dumps, logcat).
- Environment Details (Android API, backend URL, network state, clock state).
- Additional Context.

### QA Manual Bug Report (summary)

Title format: `[QA-MANUAL] <brief description>`

Fields:
- Description, Steps to Reproduce, Expected/Actual Results.
- Priority, Severity.
- Evidence, Environment.

---

## Labels (provisioned on repo)

All labels below are already created on `grabakar/grabakar-frontend`. To verify:

```bash
gh label list --repo grabakar/grabakar-frontend
```

### Bug type

| Label | Color | Description |
|-------|-------|-------------|
| `bug` | #D73A4A | Something isn't working |
| `qa-auto` | #0E8A16 | Found by automated QA agent |
| `qa-manual` | #1D76DB | Found by manual testing |

### Severity

| Label | Color | Description |
|-------|-------|-------------|
| `severity-S1` | #D73A4A | Critical: crash, data loss, auth bypass |
| `severity-S2` | #FF9800 | Major: feature broken, wrong data |
| `severity-S3` | #FFC107 | Minor: cosmetic, UX rough edge |
| `severity-S4` | #4CAF50 | Trivial: enhancement, suggestion |

### Priority

| Label | Color | Description |
|-------|-------|-------------|
| `priority-P0` | #D73A4A | Blocker: fix before any release |
| `priority-P1` | #FF9800 | Critical: fix within 48h |
| `priority-P2` | #FFC107 | Major: fix before next sprint |
| `priority-P3` | #C5DEF5 | Minor: backlog |

### Module (Frontend & Backend)

| Label | Color | Description |
|-------|-------|-------------|
| `module-login` | #BFD4F2 | Login / Auth / Session |
| `module-grabado` | #BFD4F2 | Grabado creation |
| `module-multi-vidrio` | #BFD4F2 | Multi-glass flow |
| `module-sync` | #BFD4F2 | Offline sync |
| `module-reportes` | #BFD4F2 | Reports (Front+Back) |
| `module-branding` | #BFD4F2 | White-label / branding |
| `module-security` | #BFD4F2 | Security / access control |
| `module-performance` | #BFD4F2 | Performance |
| `module-ux` | #BFD4F2 | Usability / UX |
| `module-api` | #BFD4F2 | Backend API endpoints |
| `module-db` | #BFD4F2 | Database constraints & migrations |
| `module-celery` | #BFD4F2 | Background tasks & scheduling |

### Release blocking

| Label | Color | Description |
|-------|-------|-------------|
| `blocker` | #B60205 | Blocks release |

### Re-provisioning labels (if needed)

```bash
gh label create "qa-auto" --color "0E8A16" --description "Found by automated QA agent" --force
gh label create "qa-manual" --color "1D76DB" --description "Found by manual testing" --force
gh label create "severity-S1" --color "D73A4A" --description "Critical: crash, data loss, auth bypass" --force
gh label create "severity-S2" --color "FF9800" --description "Major: feature broken, wrong data" --force
gh label create "severity-S3" --color "FFC107" --description "Minor: cosmetic, UX rough edge" --force
gh label create "severity-S4" --color "4CAF50" --description "Trivial: enhancement, suggestion" --force
gh label create "priority-P0" --color "D73A4A" --description "Blocker: fix before any release" --force
gh label create "priority-P1" --color "FF9800" --description "Critical: fix within 48h" --force
gh label create "priority-P2" --color "FFC107" --description "Major: fix before next sprint" --force
gh label create "priority-P3" --color "C5DEF5" --description "Minor: backlog" --force
gh label create "module-login" --color "BFD4F2" --description "Login / Auth / Session" --force
gh label create "module-grabado" --color "BFD4F2" --description "Grabado creation" --force
gh label create "module-multi-vidrio" --color "BFD4F2" --description "Multi-glass flow" --force
gh label create "module-sync" --color "BFD4F2" --description "Offline sync" --force
gh label create "module-reportes" --color "BFD4F2" --description "Reports" --force
gh label create "module-branding" --color "BFD4F2" --description "White-label / branding" --force
gh label create "module-security" --color "BFD4F2" --description "Security / access control" --force
gh label create "module-performance" --color "BFD4F2" --description "Performance" --force
gh label create "module-ux" --color "BFD4F2" --description "Usability / UX" --force
gh label create "module-api" --color "BFD4F2" --description "Backend API endpoints" --force
gh label create "module-db" --color "BFD4F2" --description "Database constraints & migrations" --force
gh label create "module-celery" --color "BFD4F2" --description "Background tasks & scheduling" --force
gh label create "blocker" --color "B60205" --description "Blocks release" --force
```

---

## Workflow: From Finding to Resolution

```
                              QA Agent
                              finds bug
                                  |
                                  v
                         Check for duplicate
                      (gh issue list --search)
                          /             \
                     Exists?           New?
                      /                   \
                     v                     v
              Comment on              Create issue
              existing issue          (gh issue create)
                      \                   /
                       \                 /
                        v               v
                     Issue is Open
                          |
                          v
                    Triage by QA Lead
                   (verify severity/priority)
                          |
                          v
                  Developer assigns self
                          |
                          v
                     Fix + PR created
                          |
                          v
                   QA verifies fix
                   on new build
                      /        \
                     v          v
                  PASS        FAIL
                    |           |
                    v           v
              Close issue   Reopen issue
              with comment  with comment
```

### Issue States

| State | Description | Who acts |
|-------|-------------|----------|
| **Open** | Bug found, not yet being worked on | QA Agent creates |
| **In Progress** | Developer assigned and working on fix | Developer |
| **In Review** | PR submitted, awaiting review | Developer |
| **QA Verify** | Fix merged, awaiting QA re-test | QA Agent re-runs TC |
| **Closed** | Fix verified by QA, issue resolved | QA Agent closes |
| **Reopened** | Fix failed QA verification | QA Agent reopens |

---

## Deduplication

Before creating a new issue, the QA agent checks for duplicates:

```bash
EXISTING=$(gh issue list --repo grabakar/grabakar-frontend \
  --state open --label "qa-auto" --search "TC-LOGIN-020" \
  --json number --jq '.[0].number')

if [ -n "$EXISTING" ]; then
  gh issue comment $EXISTING --repo grabakar/grabakar-frontend \
    --body "Confirmed: still failing in build v1.2.1 (commit def456) -- 2026-03-16"
else
  # Create new issue (see full example below)
fi
```

---

## Severity -> SLA Mapping

| Severity | Label | SLA | Release Blocking? |
|----------|-------|-----|-------------------|
| S1 -- Critical | `severity-S1`, `blocker` | Fix before ANY release | Yes |
| S2 -- Major | `severity-S2` | Fix within 48 hours | Yes, if P0/P1 |
| S3 -- Minor | `severity-S3` | Fix before next sprint | No |
| S4 -- Trivial | `severity-S4` | Backlog | No |

---

## Agent Issue Creation: Full Example

```bash
# Agent detects TC-SYNC-022 failure

# 1. Capture screenshot
adb shell screencap /sdcard/screen.png
adb pull /sdcard/screen.png ./reports/2026-03-16/screenshots/TC-SYNC-022_fail.png

# 2. Check for duplicate
EXISTING=$(gh issue list --repo grabakar/grabakar-frontend \
  --state open --label "qa-auto" --search "TC-SYNC-022" \
  --json number --jq '.[0].number')

if [ -n "$EXISTING" ]; then
  # 3a. Comment on existing
  gh issue comment $EXISTING --repo grabakar/grabakar-frontend \
    --body "Still failing on build v1.2.0 (abc123) -- 2026-03-16"
else
  # 3b. Create new issue
  gh issue create \
    --repo grabakar/grabakar-frontend \
    --title "[QA-AUTO] TC-SYNC-022: Errores parciales en batch no aislados" \
    --body "$(cat <<'BODY'
## Test Case Failed

| Field | Value |
|-------|-------|
| **Test ID** | TC-SYNC-022 |
| **Module** | Sincronizacion Offline |
| **Priority** | P0 |
| **Severity** | S2 -- Major |
| **Build** | v1.2.0 (abc123) |
| **Emulator** | GrabaKar_Test -- API 34 |
| **Date** | 2026-03-16 |
| **Backend** | staging |

## Steps to Reproduce
1. Created 5 grabados offline (1 with invalid ley_caso_id)
2. Restored connection
3. Sync triggered

## Expected
- 4 records synced successfully
- 1 record marked error

## Actual
- Entire batch failed
- All 5 records remain pendiente

## Evidence
Screenshot: `reports/2026-03-16/screenshots/TC-SYNC-022_fail.png`

## Environment Details
- Android API: 34
- App version: 1.2.0
- Backend URL: https://staging.grabakar.app/api/v1
- Network state: online (restored from airplane mode)
- Device clock: normal
BODY
)" \
    --label "bug,qa-auto,severity-S2,priority-P0,module-sync,blocker"
fi

# 4. Log the issue locally
echo "| TC-SYNC-022 | Sync | S2 | #<number> | Errores parciales en batch no aislados |" \
  >> ./reports/2026-03-16/issues_created.md
```

---

## Closing Issues After Fix Verification

```bash
# Re-run the failed TC on the new build
# If PASS:
gh issue close <NUMBER> --repo grabakar/grabakar-frontend \
  --comment "Verified fixed in build v1.2.1 (def456). TC-SYNC-022 now passes."

# If still FAIL:
gh issue reopen <NUMBER> --repo grabakar/grabakar-frontend \
  --comment "Still failing in build v1.2.1 (def456). Same behavior observed."
```

---

## Release Sign-Off: Open Issue Audit

Before signing off on a release, the agent produces a snapshot of all open QA issues:

```bash
gh issue list --repo grabakar/grabakar-frontend \
  --state open --label "qa-auto" \
  --json number,title,labels \
  > reports/<timestamp>/open_issues_at_signoff.json

# Check for blockers
gh issue list --repo grabakar/grabakar-frontend \
  --state open --label "blocker" \
  --json number,title
```

A release is **blocked** if any `blocker`-labeled issue is open. See `QA_CHECKLIST_RELEASE.md` for the full release criteria.
