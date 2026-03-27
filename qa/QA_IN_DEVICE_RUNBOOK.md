### QA In-Device Runbook — GrabaKar (Android Emulator)

This runbook explains **how** the QA Testing Agent (or a human following the same steps) will execute the **in-device Android testing strategy** for GrabaKar, phase by phase. It operationalizes the existing documents:

- `QA_MASTER_PLAN.md`
- `QA_AGENT_SETUP.md`
- `QA_AGENT_WORKFLOW.md`
- All `TC_*.md` test case specs
- `QA_CHECKLIST_RELEASE.md`
- `QA_GITHUB_ISSUES.md`

The goal is that **any** QA agent can follow this single document to perform a full run on the Android emulator and produce reproducible results, with every defect reported as a GitHub issue.

---

### Bug Reporting via GitHub Issues

Every phase in this runbook follows the same issue-reporting procedure when a test case **FAILS**. This section is the single source of truth; individual phases reference it rather than re-defining the process.

**Repository**: `grabakar/grabakar-frontend`

**Pre-requisites (one-time, already provisioned)**
- `gh` CLI authenticated with write access to the repo.
- All QA labels exist on the repo (run `gh label list` to verify): `qa-auto`, `severity-S1`--`S4`, `priority-P0`--`P3`, `module-*`, `blocker`.
- Issue templates installed at `.github/ISSUE_TEMPLATE/qa-auto-bug.yml` and `qa-manual-bug.yml`.

**Step-by-step procedure on each FAIL**

1. **Capture evidence**
   ```
   adb shell screencap /sdcard/TC-<ID>_fail.png
   adb pull /sdcard/TC-<ID>_fail.png reports/<timestamp>/screenshots/TC-<ID>_fail.png
   ```
   Optionally capture a UI dump:
   ```
   adb shell uiautomator dump /sdcard/ui.xml
   adb pull /sdcard/ui.xml reports/<timestamp>/ui/TC-<ID>.xml
   ```

2. **Check for duplicates**
   ```
   gh issue list --repo grabakar/grabakar-frontend \
     --state open --label "qa-auto" --search "TC-<ID>"
   ```
   - If a matching open issue exists, **add a comment** instead of creating a new issue:
     ```
     gh issue comment <NUMBER> --repo grabakar/grabakar-frontend \
       --body "Still failing on build <VERSION> (<COMMIT>) -- <DATE>"
     ```
     Then skip to step 5.

3. **Determine labels**
   Every issue gets at least: `bug`, `qa-auto`, one `severity-*`, one `priority-*`, one `module-*`.
   Add `blocker` when severity is S1 or priority is P0.

   | Test priority | Severity label | Extra |
   |---------------|----------------|-------|
   | P0 | `severity-S1` or `severity-S2` (use S1 for crashes/data-loss) | `blocker` |
   | P1 | `severity-S2` | -- |
   | P2 | `severity-S3` | -- |
   | P3 | `severity-S4` | -- |

   Module label mapping:

   | TC prefix | Label |
   |-----------|-------|
   | TC-LOGIN | `module-login` |
   | TC-GRAB | `module-grabado` |
   | TC-MV | `module-multi-vidrio` |
   | TC-SYNC | `module-sync` |
   | TC-REP | `module-reportes` |
   | TC-WL | `module-branding` |
   | TC-SEC | `module-security` |
   | TC-PERF | `module-performance` |
   | TC-UX | `module-ux` |

4. **Create the GitHub issue**
   ```
   gh issue create \
     --repo grabakar/grabakar-frontend \
     --title "[QA-AUTO] TC-<ID>: <brief description in Spanish>" \
     --body "$(cat <<'BODY'
   ## Test Case Failed

   | Field | Value |
   |-------|-------|
   | **Test ID** | TC-<ID> |
   | **Module** | <module name> |
   | **Priority** | <P0/P1/P2/P3> |
   | **Severity** | <S1/S2/S3/S4 -- label> |
   | **Build** | <version> (<commit>) |
   | **Emulator** | <AVD name> -- API <level> |
   | **Date** | <YYYY-MM-DD> |
   | **Backend** | <local / staging> |

   ## Steps to Reproduce
   1. <step from TC>
   2. ...

   ## Expected Result
   <from TC spec>

   ## Actual Result
   <observed behavior>

   ## Evidence
   Screenshot: `reports/<timestamp>/screenshots/TC-<ID>_fail.png`

   ## Environment Details
   - Android API: <level>
   - App version: <version>
   - Backend URL: <url>
   - Network state: <online / offline / airplane_mode>
   - Device clock: <normal / manipulated>
   BODY
   )" \
     --label "bug,qa-auto,<severity>,<priority>,<module>[,blocker]"
   ```

5. **Log the issue locally**
   Append a row to `reports/<timestamp>/issues_created.md`:
   ```
   | TC-<ID> | <module> | <severity> | #<issue_number> | <title> |
   ```

**Verification of issue quality (self-check)**
Before moving on, the agent confirms each filed issue has:
- A title starting with `[QA-AUTO] TC-<ID>:`.
- All table fields filled (no placeholders left).
- At least one screenshot or UI dump referenced.
- Correct labels (verify with `gh issue view <NUMBER> --json labels`).

**Closing / re-testing workflow**
- When a fix is merged and a new build is available, re-run the failed TC.
- On PASS: close the issue with a comment:
  ```
  gh issue close <NUMBER> --repo grabakar/grabakar-frontend \
    --comment "Verified fixed in build <VERSION> (<COMMIT>)"
  ```
- On FAIL: add a comment and reopen if it was closed:
  ```
  gh issue reopen <NUMBER> --repo grabakar/grabakar-frontend \
    --comment "Still failing in build <VERSION> (<COMMIT>)"
  ```

---

### Phase 0 -- Environment & Build Readiness

**Objective**: Ensure the Android emulator, APK, backend, and GitHub issue-reporting pipeline are ready before any test case is executed.

**Inputs**
- Target build identifier (version + commit SHA).
- Target backend environment (`local` or `staging`).
- Access to the repo on this machine.

**Steps**
1. **Tooling sanity check**
   - Verify Java:
     - `java -version` -> **JDK 21** (requerido por build Android actual).
   - Verify Android SDK / ADB / Emulator:
     - `adb version`
     - If `adb` is not found (command not available), update your PATH following `QA_AGENT_SETUP.md` (Android SDK platform-tools) and re-open the terminal, then re-run Phase 0.
     - `emulator -version`
   - Verify `node`, `python`, `gh` CLI:
     - `node --version`
     - `python3 --version`
     - `gh --version`
2. **AVD availability**
   - `avdmanager list avd` must contain `GrabaKar_Test`.
3. **Boot emulator**
   - Interactive:
     - `emulator -avd GrabaKar_Test -no-audio -no-boot-anim`
   - Headless (CI-style):
     - `emulator -avd GrabaKar_Test -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect &`
   - Wait for boot:
     - `adb wait-for-device`
     - `adb shell getprop sys.boot_completed` -> must return `1`.
4. **Network check from emulator**
   - `adb shell ping -c 1 google.com` should succeed (for online phases).
5. **Backend health**
   - **Local**: `curl http://localhost:8000/api/v1/health/`
   - **From emulator** (local backend): confirm app config uses `http://10.0.2.2:8000/api/v1`.
   - **Staging**: health endpoint of staging must return OK.
6. **APK build & install**
   - If building from source (debug example):
     - `cd grabakar-frontend`
     - `npm install`
     - Export Java 21:
       - `export JAVA_HOME="/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home"`
       - `export PATH="$JAVA_HOME/bin:$PATH"`
     - Build variant according to target backend:
       - Local: `npm run build:android:local`
       - Staging GCP: `npm run build:android:gcp`
       - Ambas variantes: `npm run build:android:variants`
   - Install on emulator:
     - `adb install -r app/build/outputs/apk/debug/app-debug.apk`
   - Verify installation:
     - `adb shell pm list packages | grep grabakar`
     - Launch:
       - `adb shell am start -n com.grabakar.app/.MainActivity`
7. **Test users / tenants**
   - En staging, asegurar usuarios seeded por `bootstrap-staging.yml`:
     - `admin`, `supervisor`, `operador`, `test`
     - Password default: `grabakar123`
   - Verificar login API antes de iniciar QA in-device:
     - `POST /api/v1/auth/login/` con cada usuario debe retornar `200`.
   - If using local backend and not seeded, run the user-creation script described in `QA_AGENT_SETUP.md`.
8. **GitHub issue reporting readiness**
   - Confirm `gh auth status` shows the agent is authenticated with write scope.
   - Confirm labels exist: `gh label list --repo grabakar/grabakar-frontend | grep qa-auto`.
   - Confirm issue templates exist: `.github/ISSUE_TEMPLATE/qa-auto-bug.yml` is present in the repo.
   - Create the run report directory and initialize `issues_created.md`:
     ```
     mkdir -p reports/<timestamp>/screenshots
     echo "| TC ID | Module | Severity | Issue | Title |" > reports/<timestamp>/issues_created.md
     echo "|-------|--------|----------|-------|-------|" >> reports/<timestamp>/issues_created.md
     ```

**Artifacts to capture**
- `reports/<timestamp>/env.md` containing:
  - Build identifier (version + commit).
  - Backend environment and URL.
  - Emulator AVD name and API level.
  - Tool versions (Java, Node, Python, adb, emulator, gh).
  - `gh auth status` output confirming issue-reporting capability.

**Definition of done (Phase 0)**
- Emulator is booted and responsive via ADB.
- Target APK is installed and launches without crash to initial screen.
- Backend health endpoint returns OK.
- Test users and tenant(s) exist and are accessible.
- `gh auth status` confirms write access; QA labels verified on the repo.
- `env.md` file is stored under the current run folder.
- `issues_created.md` initialized in the run folder.

**Success criteria for QA agent**
- All checks pass without manual intervention beyond starting commands.
- If any check fails, the agent **terminates early** with a clear message and logs under `reports/<timestamp>/env.md` indicating setup failure (no test cases are executed).

---

### Phase 1 -- P0 Smoke Suite on Device

**Objective**: Verify that the app is in a sufficiently healthy state to justify deeper testing by running core P0 smoke tests.

**Scope (examples, see `QA_CHECKLIST_RELEASE.md`)**
- Login & Session: `TC-LOGIN-001`, `TC-LOGIN-013`, `TC-LOGIN-020`, `TC-LOGIN-021`, `TC-LOGIN-022`, `TC-LOGIN-030`, `TC-LOGIN-040`, `TC-LOGIN-076`.
- Grabado creation: `TC-GRAB-001`, `TC-GRAB-004`, `TC-GRAB-010`, `TC-GRAB-013`, `TC-GRAB-016`, `TC-GRAB-030`, `TC-GRAB-040`, `TC-GRAB-041`.
- Multi-vidrio, Sync, Security, Reports, White-label as listed in the checklist.

**Execution pattern (per test case)**
1. Reset app state if necessary:
   - `adb shell pm clear com.grabakar.app`
2. Launch app:
   - `adb shell am start -n com.grabakar.app/.MainActivity`
3. Execute **preconditions** from the TC:
   - Login as required role.
   - Ensure online state (for online scenarios).
4. Execute **steps** using UI automation:
   - `adb shell input tap <x> <y>`
   - `adb shell input text "<VALUE>"`
   - `adb shell input keyevent <CODE>`
   - (Optionally Appium WebDriver/Docker if configured).
5. **Validate expected result**:
   - Capture screenshot:
     - `adb shell screencap /sdcard/TC-XXX.png`
     - `adb pull /sdcard/TC-XXX.png reports/<timestamp>/screenshots/TC-XXX.png`
   - Optionally dump UI tree:
     - `adb shell uiautomator dump /sdcard/ui.xml`
     - `adb pull /sdcard/ui.xml reports/<timestamp>/ui/TC-XXX.xml`
6. Record outcome:
   - Append a line to `reports/<timestamp>/results.csv`:
     - `TC_ID,Module,Priority,Status,Notes,GH_Issue`
   - Status in {PASS, FAIL, BLOCKED}.
7. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section above:
   - Capture evidence, check for duplicates, create or comment, log to `issues_created.md`.
   - For P0 smoke failures, always include the `blocker` label.

**Stop condition**
- If **any P0 smoke test FAILS** with severity S1/S2, mark the smoke phase as failed and **do not continue** to later phases for this build, unless the run is explicitly marked as "exploratory".
- All FAIL issues must be filed **before** the agent stops (even if the run is aborted early).

**Definition of done (Phase 1)**
- Every P0 smoke test listed in `QA_CHECKLIST_RELEASE.md` has a status and minimal notes.
- At least one screenshot exists for each FAIL.
- Every FAIL has a corresponding GitHub issue (new or commented on existing).

**Success criteria for QA agent**
- 100% of P0 smoke cases are actually executed (no silent skips).
- A concise `smoke-summary.md` is generated with:
  - Total P0 tests, passes, fails, blocked.
  - List of GitHub issue numbers filed during this phase.
  - Explicit "GO/NO-GO for deeper testing" recommendation.

---

### Phase 2 -- Core Functional Flows (Login, Grabado, Multi-Vidrio)

**Objective**: Validate that the main business flows work end-to-end on the device beyond the minimal smoke.

**Scope (see `TC_LOGIN_SESION.md`, `TC_GRABADO_PATENTE.md`, `TC_FLUJO_MULTI_VIDRIO.md`)**
- Login lifecycle: fresh login, token refresh, offline token reuse, logout and token cleanup.
- Grabado creation: valid/invalid patente, normalization, duplicate detection, confirmacion de patente.
- Multi-vidrio: Auto (6 vidrios) and Moto (1 vidrio), print and finalize logic, patent display consistency.

**Execution pattern**
1. For each module, load its list of P0 + main P1 test cases.
2. For **login/session**:
   - Use **time manipulation** if required:
     - `adb shell date -s "YYYYMMDD.HHMMSS"` for token expiry tests (only in isolated environments).
3. For **grabado**:
   - Test variety of patente inputs (normal, short, malformed, mixed case) according to cases.
   - Validate IndexedDB/"pending records" via UI indicators (exact technical inspection is covered at lower levels; here the focus is visible behavior).
4. For **multi-vidrio**:
   - Exercise the full 6-step flow for Auto (print/omit, progress bar / step indicator).
   - Confirm Moto path shows 1 glass and direct "Finalizar".
5. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section.

**Artifacts**
- `reports/<timestamp>/functional-login.md`
- `reports/<timestamp>/functional-grabado.md`
- `reports/<timestamp>/functional-multividrio.md`
  - Each with small tables: `TC_ID`, `Status`, `Evidence`, `GH_Issue`.

**Definition of done (Phase 2)**
- All P0 + key P1 test cases in login, grabado and multi-vidrio have outcomes and are reflected in per-module reports.
- Every FAIL has a GitHub issue filed or an existing issue commented on.

**Success criteria for QA agent**
- Any functional break is clearly tied to a **specific test case ID** and a **GitHub issue number**.
- Repro steps and screenshots for each FAIL are sufficient for a developer to reproduce without needing to re-run the whole suite.

---

### Phase 3 -- Offline & Synchronization Scenarios

**Objective**: Verify offline-first behavior and sync reliability with controlled network conditions on the emulator.

**Scope (see `TC_SINCRONIZACION_OFFLINE.md`)**
- Creating grabados offline and persisting them locally.
- Automatic sync when the network is restored.
- Conflict resolution rules and non-purging of unsynced records.

**Execution pattern**
1. Start from clean state:
   - `adb shell pm clear com.grabakar.app`
   - Launch the app.
2. Toggle **airplane mode** and WiFi as per test case:
   - `adb shell settings put global airplane_mode_on 1`
   - `adb shell settings put global airplane_mode_on 0`
   - `adb shell svc wifi disable`
   - `adb shell svc wifi enable`
3. While offline:
   - Perform grabado flows configured as offline in the TCs.
   - Confirm local queues / pending indications via UI.
4. Restore connectivity:
   - Disable airplane mode / enable WiFi.
   - Wait for background sync or trigger manual sync action.
5. Validate:
   - UI sync indicators reflect success/failure.
   - No pending records are silently dropped; failures are visible to the operator/supervisor.
6. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section.
   - Include the exact network state (airplane on/off, WiFi on/off) in both the issue body and evidence.

**Artifacts**
- `reports/<timestamp>/offline-sync.md` summarizing:
  - Number of offline records created.
  - Number successfully synced, conflicts detected, errors surfaced.
  - GitHub issue numbers for any failures.

**Definition of done (Phase 3)**
- All P0 + main P1 offline/sync scenarios are executed and logged.
- Each scenario clearly logs the **network precondition** and resulting sync behavior.
- Every FAIL has a GitHub issue filed.

**Success criteria for QA agent**
- Network conditions are fully scripted (no manual toggling).
- The same offline/sync scenario can be replayed reliably with similar results across runs.
- Every sync defect is tracked in a GitHub issue with `module-sync` label.

---

### Phase 4 -- Security, Roles & Tenant Isolation

**Objective**: Ensure role-based access control and tenant isolation behave correctly when exercised via the app UI on the emulator.

**Scope (see `TC_SEGURIDAD.md`, `TC_LOGIN_SESION.md`)**
- Roles: operador, supervisor, admin.
- Isolation: tenants cannot see each other's data.
- Immutable fields after save (e.g., patente).

**Execution pattern**
1. For each role:
   - Login using the preconfigured test users.
   - Navigate to menus/screens only that role should see.
   - Attempt to access actions that the role **should not** be able to perform.
2. For tenant isolation:
   - Prepare test data for at least 2 tenants (A and B).
   - For a user from tenant A:
     - Confirm that listings, reports and anything aggregating data show only tenant A.
   - Repeat for tenant B if available.
3. Verify immutable fields:
   - After a grabado is saved, try to modify the patente through any available UI; confirm it is impossible or clearly blocked.
4. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section.
   - Security issues default to `severity-S1` unless clearly cosmetic; always include `blocker` for access-control failures.

**Artifacts**
- `reports/<timestamp>/security-matrix.md` with a table:
  - Rows: (role, feature/endpoint), Columns: expected access, actual observed, status, GH issue.

**Definition of done (Phase 4)**
- All P0 security and isolation tests are executed and accounted for in the security matrix.
- Every FAIL has a GitHub issue filed with `module-security` and appropriate severity.

**Success criteria for QA agent**
- A matrix-style summary is produced with **GitHub issue links** for every violation.
- No ambiguous "maybe security" findings: each one is clearly described, reproducible, and tracked.

---

### Phase 5 -- White-Labeling & Reports

**Objective**: Confirm that multi-tenant branding and reporting work correctly on-device.

**Scope (see `TC_WHITE_LABELING.md`, `TC_REPORTES.md`)**
- Branding per tenant and persistence offline.
- No unwanted `"GrabaKar"` hardcoded in white-label contexts.
- Reports content and access control per role and tenant.

**Execution pattern**
1. Configure at least:
   - One tenant with **custom branding**.
   - One **default** tenant.
2. For each tenant:
   - Login with a user for that tenant.
   - Capture a screenshot of:
     - Login screen (if branded).
     - Home/dashboard.
     - Any branded header/footer components.
3. Turn offline and back online to confirm branding elements persist and are not replaced with defaults.
4. For reports:
   - As operador: confirm 403/blocked for report actions.
   - As supervisor/admin: generate/download the main reports.
   - Confirm that report content includes only tenant-specific data.
5. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section.
   - Branding issues use `module-branding`; report issues use `module-reportes`.

**Artifacts**
- `reports/<timestamp>/branding.md` and `reports/<timestamp>/reports-functional.md`.
- Organized screenshots per tenant:
  - `screenshots/tenant-<name>/<context>.png`
- GitHub issue numbers listed in each report for any failures.

**Definition of done (Phase 5)**
- Both branded and default tenant flows have been executed and documented.
- No unexpected branding leaks or cross-tenant data in reports are observed without being recorded as GitHub issues.

**Success criteria for QA agent**
- Visual evidence (screenshots) clearly shows differences between tenants.
- Any branding or report bug includes which tenant, which role, and which screen/report in the GitHub issue description.

---

### Phase 6 -- Performance & Usability

**Objective**: Measure essential performance metrics and validate critical usability aspects on the emulator.

**Scope (see `TC_RENDIMIENTO.md`, `TC_USABILIDAD.md`)**
- Cold start time.
- Sync performance (e.g., 100 records).
- Non-blocking UI during sync.
- Error message clarity, connectivity indicators, patent visual treatment, confirmation on save.

**Execution pattern**
1. **Performance**
   - Cold start:
     - From app not running, measure time from launch command to fully interactive screen using:
       - External stopwatch; or
       - Instrumentation logs if available.
   - Sync:
     - Prepare a known number of pending records (e.g., 100) offline.
     - Go online and measure time until sync completes based on UI indicators/logs.
2. **Usability**
   - Execute representative flows and capture:
     - Error messages when input is invalid or network is down.
     - Visibility of online/offline indicator.
     - Visual feedback after saving a grabado.
     - Legibility of patent text in multi-vidrio (size, weight, font).
3. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section.
   - Performance issues use `module-performance`; usability issues use `module-ux`.
   - Include measured vs expected thresholds in the issue body.

**Artifacts**
- `reports/<timestamp>/performance.md` with:
  - Cold start time(s) vs target threshold.
  - Sync duration(s) vs threshold.
  - GitHub issue numbers for threshold violations.
- `reports/<timestamp>/usability.md` with:
  - List of usability observations tied to TC IDs and GitHub issue numbers.

**Definition of done (Phase 6)**
- All P1 performance and usability cases are executed at least once per reference device configuration.
- Every metric that exceeds the target threshold has a GitHub issue filed.

**Success criteria for QA agent**
- Metrics are clearly reproducible and associated to environment (AVD, API level, backend).
- Usability feedback is actionable, not just subjective, and tracked in GitHub issues.

---

### Phase 7 -- Regression & Edge Cases

**Objective**: Run remaining regression and edge test cases to detect non-obvious failures once core flows are stable.

**Scope (see `TC_EDGE_CASES.md` and remaining non-P0/P1 tests)**
- Large input sizes, special characters, unusual navigation paths.
- App backgrounding/foregrounding during flows (if relevant).
- Date/time changes during session.

**Execution pattern**
1. Build a **regression manifest** for this run:
   - List of modules and test IDs selected (P2-P3 and specific edge cases).
2. Execute each case following the standard pattern:
   - Reset, launch, preconditions, steps, validation, evidence.
3. For any new behavior **not covered by existing TCs**, create a new test case stub:
   - ID, description, steps, expected, and mark as "New edge case" in notes.
4. **On FAIL -> file GitHub issue** following the "Bug Reporting via GitHub Issues" section.
   - Use `severity-S3` or `severity-S4` per the P2/P3 mapping unless the finding is more severe than expected.

**Artifacts**
- `reports/<timestamp>/regression.md` including:
  - Which modules and TCs were in-scope vs out-of-scope for this regression cycle.
  - GitHub issue numbers for any failures.

**Definition of done (Phase 7)**
- All planned regression and edge cases for the run are executed or explicitly marked as skipped with a reason.
- Every FAIL has a GitHub issue.

**Success criteria for QA agent**
- No failures are left "floating": each is tied to an existing TC or a newly defined edge-case stub, and tracked in a GitHub issue.

---

### Phase 8 -- Release Sign-Off & Post-Release Smoke

**Objective**: Aggregate all results into a release decision and verify the distributed APK build (staging/production) with a short smoke suite.

**Execution pattern**
1. **Aggregate results**
   - Use data from all previous phase reports to generate:
     - `reports/<timestamp>/summary.md` following the format from `QA_AGENT_WORKFLOW.md`:
       - Total tests, passed, failed, blocked.
       - Pass rates.
       - Build metadata.
       - Tables of failed and blocked tests with **links to GitHub issues**.
2. **Verify open issue state**
   - Run:
     ```
     gh issue list --repo grabakar/grabakar-frontend \
       --state open --label "qa-auto" --json number,title,labels
     ```
   - Confirm no open `blocker` or `priority-P0` / `priority-P1` issues remain for a GO decision.
   - Include this output in the summary report.
3. **Apply release criteria** (see `QA_CHECKLIST_RELEASE.md` and `QA_MASTER_PLAN.md`):
   - All P0 tests must PASS.
   - >= 95% P1 tests must PASS.
   - No open P0/P1 GitHub issues.
   - Smoke suite passes on the target build.
4. **Post-release smoke**
   - Install the **release APK** (Play Store or distribution channel) on the emulator or a physical device.
   - Run a minimal subset of smoke cases:
     - Basic login.
     - One full grabado + multi-vidrio.
     - One sync scenario.
   - Capture a brief `post-release-smoke.md` report.
   - **On FAIL -> file GitHub issue** (same procedure) with an extra note that this is a **post-release finding**.

**Artifacts**
- `reports/<timestamp>/summary.md` -- includes issue links for every failure.
- `reports/<timestamp>/issues_created.md` -- complete log of all issues filed during the run.
- `reports/<timestamp>/open_issues_at_signoff.md` -- snapshot of open QA issues at sign-off time.
- `reports/<timestamp>/post-release-smoke.md` (for production runs).

**Definition of done (Phase 8)**
- A clear GO/NO-GO recommendation is stated and supported by metrics and links to evidence.
- `issues_created.md` is complete and cross-referenced with the summary.
- Open-issue snapshot is captured for audit.

**Success criteria for QA agent**
- Stakeholders can read `summary.md` plus a handful of linked files and fully understand:
  - What was tested.
  - What failed and where (with clickable GitHub issue links).
  - Whether criteria for release are met.
  - What issues remain open and their severities.

---

### Global Definition of Done & Success for In-Device QA Runs

**Global definition of done**
- Phase 0 checks completed and logged (including GitHub readiness).
- All tests planned for this run have explicit statuses (PASS, FAIL, BLOCKED, SKIPPED) and mapping to TC IDs.
- Every FAIL has:
  - At least one screenshot.
  - Minimal repro steps.
  - A GitHub issue with correct labels and severity (new or commented on existing).
- `issues_created.md` is complete and contains every issue filed or commented on during the run.
- Run artifacts exist under `reports/<timestamp>/` per this runbook.

**Global success for the QA Testing Agent**
- **Coverage**: No P0 tests accidentally omitted; P1+ executed according to this plan's scope.
- **Accuracy**: Low false-positive rate; issues are reproducible and traceable to test cases.
- **Traceability**: Every defect has a GitHub issue; every GitHub issue maps back to a TC ID.
- **Observability**: Reports and folder structure are sufficient to debug and trend issues release over release. The `issues_created.md` log provides a complete audit trail.
- **Repeatability**: This runbook can be reused for future builds with minimal adjustments (mainly build ID, environment, and scope selection).
