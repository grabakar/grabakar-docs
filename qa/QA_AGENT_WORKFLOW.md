# QA Agent Workflow — Android Emulator Testing

This document defines the complete workflow for the **QA Tester Agent**, an autonomous agent that boots an Android emulator, installs the GrabaKar APK, executes test scenarios, captures evidence, and logs findings as GitHub issues.

---

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      QA TESTER AGENT                          │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐  │
│  │ ADB Control │  │ API Runner  │  │ GitHub Issue Logger  │  │
│  │ (Frontend)  │  │ (Backend)   │  │                      │  │
│  │ • boot emu  │  │ • HTTP GET/ │  │ • create issues     │  │
│  │ • install   │  │   POST reqs │  │ • attach evidence    │  │
│  │ • tap/swipe │  │ • run tasks │  │ • label severity     │  │
│  │ • screenshot│  │ • chk model │  │ • assign to proj     │  │
│  └─────────────┘  └─────────────┘  └──────────────────────┘  │
│                                                               │
│  Output: reports/<timestamp>/                                 │
│  ├── summary.md          # Overall results                    │
│  ├── screenshots/        # UI Evidence per test case          │
│  ├── api_logs/           # JSON payloads / Celery tracebacks  │
│  └── issues_created.md   # Log of GitHub issues filed         │
└───────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

Before the agent can run, the environment must be set up per [QA_AGENT_SETUP.md](QA_AGENT_SETUP.md):
- Android SDK + emulator + ADB installed
- AVD created (e.g., `GrabaKar_Test`)
- `gh` CLI authenticated with write access to the repo
- APK file available at a known path
- Backend running (Docker Compose local or staging URL)

---

## Phase 1: Environment Boot

```bash
# 1. Start emulator (headless for CI, or with display for interactive)
emulator -avd GrabaKar_Test -no-audio -no-boot-anim &

# 2. Wait for boot to complete
adb wait-for-device
adb shell getprop sys.boot_completed  # wait until returns "1"

# 3. Verify connectivity
adb shell ping -c 1 google.com

# 4. Install APK
adb install -r /path/to/grabakar.apk

# 5. Verify installation
adb shell pm list packages | grep grabakar
```

**Agent decision point**: If any step fails, log setup failure and abort with error.

---

## Phase 2: Test Data Setup

```bash
# Seed backend with test data (if local)
cd /path/to/grabakar-backend
python manage.py loaddata api/fixtures/initial.json

# Or verify against staging — ensure test users exist:
# - operador1 / password123 (operador role)
# - supervisor1 / password123 (supervisor role)  
# - admin1 / password123 (admin role)
# - All belong to test tenant "GrabaKar Test"
```

---

## Phase 3: Test Execution

The agent executes test cases from the TC documents. For each test case:

### A. Frontend Execution Loop (Emulator)

```
FOR each test case in frontend test suite:
    1. Reset app state if needed
       adb shell pm clear com.grabakar.app
       
    ... [Steps 2-7 detailed UI interactions]
```

### B. Backend API Execution Loop (cURL/HTTP)

```
FOR each test case in backend test suite (TC_BACKEND_*):
    1. Authenticate & Obtain Tokens
       - POST /api/v1/auth/login/ -> save Access Token
       
    2. Setup Data State
       - Verify DB seeding or send pre-requisite POSTs
       
    3. Execute API Calls
       - Send GET/POST with predefined payloads
       - Capture HTTP status codes and JSON response body
       
    4. Trigger Background Tasks (Celery)
       - Using CLI or internal test endpoints if debugging
       - E.g., docker-compose exec django python manage.py trigger_report
       
    5. Validate Expected Results
       - Assert HTTP Status == 200/400/403
       - Parse JSON for expected keys/values
       - Check Content-Type for XLSX downloads
       - Check Backend Logs / DB records (via Admin or shell)
       
    6. Record Result
       - PASS: log and continue
       - FAIL: log, capture request/response JSON, create GitHub issue on grabakar-backend repo
       - BLOCKED: log reason, continue
```

### Priority Execution Order

1. **Smoke tests first** (P0 cases from each module):
   - TC-LOGIN-001, TC-LOGIN-020, TC-LOGIN-021
   - TC-GRAB-001, TC-GRAB-004, TC-GRAB-013, TC-GRAB-016
   - TC-MV-001, TC-MV-002
   - TC-SYNC-001, TC-SYNC-022
   - TC-SEC-001, TC-SEC-010
   - TC-DATA-001, TC-REPORTES-B-002, TC-REPORTES-B-003

2. **Critical path** (P1):
   - All P1 cases across frontend and backend modules

3. **Regression** (P2-P3):
   - Remaining cases

---

## Phase 4: Issue Logging

When a test **FAILS**, the agent creates a GitHub issue:

```bash
```bash
# Frontend UI failures -> grabakar-frontend
gh issue create \
  --repo "grabakar/grabakar-frontend" \
  --title "[QA-AUTO] TC-LOGIN-020: Login sin conexión no muestra error" \
  ...

# Backend API/Data failures -> grabakar-backend
gh issue create \
  --repo "grabakar/grabakar-backend" \
  --title "[QA-AUTO] TC-REPORTES-B-003: Operador puede generar XLSX de plataforma" \
  --body "$(cat issue_body.md)" \
  --label "bug,qa-auto,severity-S1,module-api,blocker"
```

### Issue Body Template

```markdown
## Test Case Failed

| Field | Value |
|-------|-------|
| **Test ID** | TC-LOGIN-020 |
| **Module** | Login y Gestión de Sesión |
| **Priority** | P0 |
| **Severity** | S2 — Major |
| **Build** | APK v1.2.0 (commit abc123) |
| **Emulator** | Pixel_6_API_34 |
| **Date** | 2026-03-16 |

## Steps Reproduced
1. Enabled airplane mode on emulator
2. Opened app
3. Entered valid credentials
4. Tapped "Iniciar Sesión"

## Expected
- Error message: "Se requiere conexión a internet para iniciar sesión."

## Actual
- App showed loading spinner indefinitely
- No error message displayed after 30 seconds

## Evidence
![Screenshot at failure](screenshots/TC-LOGIN-020_fail.png)

## Environment
- Android API: 34
- App version: 1.2.0
- Backend: staging
- Network: airplane mode (offline)
```

### Label Mapping

| Condition | Labels |
|-----------|--------|
| P0 test failure | `bug`, `qa-auto`, `severity-S1`, `blocker` |
| P1 test failure | `bug`, `qa-auto`, `severity-S2` |
| P2 test failure | `bug`, `qa-auto`, `severity-S3` |
| P3 test failure | `bug`, `qa-auto`, `severity-S4` |

---

## Phase 5: Report Generation

At the end of a test run, generate a summary report:

```
reports/
└── 2026-03-16T08-44-00/
    ├── summary.md          # Full results table
    ├── screenshots/        # All captured screenshots
    │   ├── TC-LOGIN-001_pass.png
    │   ├── TC-LOGIN-020_fail.png
    │   └── ...
    └── issues_created.md   # Links to created GitHub issues
```

### Summary Report Format

```markdown
# QA Test Run Report — 2026-03-16

| Metric | Value |
|--------|-------|
| **Total tests** | 238 |
| **Passed** | 225 |
| **Failed** | 8 |
| **Blocked** | 5 |
| **Pass rate** | 94.5% |
| **Duration** | 45 min |
| **Build** | v1.2.0 (abc123) |

## Failed Tests

| TC ID | Module | Priority | Issue |
|-------|--------|----------|-------|
| TC-LOGIN-020 | Login | P0 | #142 |
| TC-SYNC-022 | Sync | P0 | #143 |
| ... | ... | ... | ... |

## Blocked Tests

| TC ID | Reason |
|-------|--------|
| TC-REP-020 | XLSX feature not implemented |
| TC-DATA-004 | DB Constraints currently disabled |
| ... | ... |
```

---

## Agent Commands Reference

### ADB Quick Reference

```bash
# Device management
adb devices                               # List connected devices
adb shell getprop ro.build.version.sdk    # Get API level

# App control
adb install -r app.apk                   # Install/reinstall
adb shell pm clear com.grabakar.app      # Clear app data
adb shell am start -n com.grabakar.app/.MainActivity  # Launch
adb shell am force-stop com.grabakar.app  # Force stop

# UI interaction
adb shell input tap 540 960              # Tap at coordinates
adb shell input text "BBDF12"            # Type text
adb shell input keyevent 66              # Press Enter
adb shell input keyevent 4               # Press Back
adb shell input swipe 540 1600 540 400   # Swipe up

# Network control
adb shell settings put global airplane_mode_on 1   # Airplane on
adb shell settings put global airplane_mode_on 0   # Airplane off
adb shell svc wifi enable                           # WiFi on
adb shell svc wifi disable                          # WiFi off

# Screenshot & UI dump
adb shell screencap /sdcard/screen.png
adb pull /sdcard/screen.png
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml

# Clock manipulation (for token tests)
adb shell date -s "20260319.120000"       # Set date
```

### Backend / API Quick Reference

```bash
# HTTP Requests
curl -X POST "http://localhost:8000/api/v1/auth/login/" -d '{"username":"admin", "password":"123"}' -H "Content-Type: application/json"
curl -X GET "http://localhost:8000/api/v1/reportes/plataforma/" -H "Authorization: Bearer <TOKEN>" -o reporte.xlsx

# DB & Background Verification
docker-compose exec django python manage.py shell -c "from api.models import Grabado; print(Grabado.objects.count())"
docker-compose logs celery_worker | grep "Generacion XLSX"
```

### gh CLI Quick Reference

```bash
# Issues
gh issue create --title "..." --body "..." --label "bug"
gh issue list --label "qa-auto"
gh issue close <number>

# Labels (one-time setup)
gh label create "qa-auto" --color "0E8A16" --description "Automated QA finding"
gh label create "severity-S1" --color "D73A4A" --description "Critical"
gh label create "severity-S2" --color "FF9800" --description "Major"
gh label create "severity-S3" --color "FFC107" --description "Minor"
gh label create "severity-S4" --color "4CAF50" --description "Trivial"
```

---

## Run Frequency

| Trigger | Scope | Environment |
|---------|-------|-------------|
| Per PR merge to `main` | Smoke (P0 only) | Local emulator |
| Pre-release | Full suite | Staging |
| Weekly | Full suite | Staging |
| On demand | Selected modules | Any |
