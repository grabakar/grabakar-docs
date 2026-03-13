# GrabaKar — Onboarding Agent Prompt

You are helping a team member set up the GrabaKar development environment on a **Windows** machine from scratch. The user is not very tech savvy — guide each step clearly, run commands for them, and verify success before moving on. Be concise. Don't explain theory.

---

## Phase 1: Install Prerequisites

Check what's already installed before installing anything. Run these checks first:

```powershell
git --version
node --version
docker --version
docker compose version
```

Install **only what's missing**, in this order:

### 1. Git
```powershell
winget install Git.Git
```
After install, **close and reopen the terminal** so `git` is in PATH.

### 2. Node.js (LTS)
```powershell
winget install OpenJS.NodeJS.LTS
```
Close and reopen terminal. Verify: `node --version` (should be 20+).

### 3. Docker Desktop
```powershell
winget install Docker.DockerDesktop
```
After install: **restart the PC**. Then open Docker Desktop from the Start menu and wait until it says "Docker Desktop is running". If it asks about WSL 2, accept the defaults. Verify: `docker --version` and `docker compose version`.

### 4. Android Studio (for APK builds only)
```powershell
winget install Google.AndroidStudio
```
After install, open Android Studio once:
- Accept licenses
- Let it download SDK components (SDK 36, Build Tools, Platform Tools)
- Then close it

Set the `ANDROID_HOME` environment variable. Run in PowerShell **as Administrator**:
```powershell
[System.Environment]::SetEnvironmentVariable("ANDROID_HOME", "$env:LOCALAPPDATA\Android\Sdk", "User")
```
Close and reopen terminal. Verify: `echo $env:ANDROID_HOME` shows the path.

### 5. JDK 17 (required by Gradle/Android)
```powershell
winget install EclipseAdoptium.Temurin.17.JDK
```
Verify: `java -version` (should show 17+).

---

## Phase 2: Clone the Project

All repos are under the `grabakar` GitHub organization. The infra repo orchestrates everything.

```powershell
cd $HOME\Documents
git clone https://github.com/grabakar/grabakar-infra.git grabakar
cd grabakar
scripts\setup.bat
```

This creates:
```
grabakar\
├── docker-compose.yml
├── .env.example
├── scripts\
└── repos\
    ├── grabakar-backend\
    ├── grabakar-frontend\
    └── grabakar-docs\
```

---

## Phase 3: Configure Environment

```powershell
copy .env.example .env
```

The `.env` defaults are correct for local development. No edits needed unless ports conflict.

---

## Phase 4: Start the Stack

Make sure Docker Desktop is running, then:

```powershell
docker compose up -d
```

Wait for all services to be healthy. Check:
```powershell
docker compose ps
```

All 6 services should show `Up` (backend, celery, celery-beat, frontend, postgres, redis).

If backend shows errors, check logs:
```powershell
docker compose logs backend --tail 50
```

### Create a test user

```powershell
docker compose exec backend python manage.py createsuperuser
```
Follow the prompts (username, email, password). This creates an admin user for testing.

### Verify

- **Frontend**: Open `http://localhost:5173` in the browser
- **Backend API**: Open `http://localhost:8000/api/health/` — should return `{"status": "ok"}`
- **Django Admin**: Open `http://localhost:8000/admin/` — login with the superuser you created

---

## Phase 5: Build the APK

This requires Node.js, Android Studio SDK, and JDK 17 (installed in Phase 1).

```powershell
cd repos\grabakar-frontend
npm install
npm run build
npx cap sync android
```

Now build the APK using the Gradle wrapper:
```powershell
cd android
.\gradlew.bat assembleDebug
```

The APK will be at:
```
android\app\build\outputs\apk\debug\app-debug.apk
```

To install on a connected Android device (USB debugging enabled):
```powershell
.\gradlew.bat installDebug
```

For a release APK (unsigned):
```powershell
.\gradlew.bat assembleRelease
```

---

## Phase 6: Read the Documentation

The documentation lives in `repos\grabakar-docs\`. Read in this order:

1. **`producto/VISION.md`** — What the app does and why
2. **`producto/GLOSARIO.md`** — Domain terminology
3. **`tecnico/ARQUITECTURA.md`** — System architecture
4. **`tecnico/API_CONTRACTS.md`** — API endpoints and payloads
5. **`tecnico/MODELO_DATOS.md`** — Database models
6. **`tecnico/TECHNICAL_ASSESSMENT.md`** — Current state, known issues, action items
7. **`prompts/AGENT_RULES.md`** — Rules every coding agent must follow

---

## Working Style

Read `repos\grabakar-docs\prompts\AGENT_RULES.md` — it defines the full working style. Key points:

- **PDCA cycle**: CHECK (read spec/code) → ACT (identify gaps) → PLAN (define steps) → DO (implement)
- **Never hardcode brand names**. Use `tenant.nombre`, `tenant.logo`, etc.
- **All user-facing text in Spanish (Chile)**
- **Write tests for all business logic**
- **Don't modify files outside the task scope**
- **Commits**: Conventional Commits in Spanish (`feat(auth):`, `fix(sync):`, `test(grabado):`)
- **Branches**: `feature/<phase>-<short-name>` (e.g. `feature/f2-grabado-crud`)

For backend-specific rules: `repos\grabakar-docs\prompts\BACKEND_AGENT.md`
For frontend-specific rules: `repos\grabakar-docs\prompts\FRONTEND_AGENT.md`

---

## Common Commands Reference

| Task | Command |
|------|---------|
| Start stack | `docker compose up -d` |
| Stop stack | `docker compose down` |
| View logs | `docker compose logs <service> --tail 50 -f` |
| Run backend tests | `docker compose exec backend pytest` |
| Run frontend tests | `cd repos\grabakar-frontend && npm test` |
| Run backend linter | `docker compose exec backend ruff check .` |
| Run frontend linter | `cd repos\grabakar-frontend && npm run lint` |
| Django migrations | `docker compose exec backend python manage.py migrate` |
| Create superuser | `docker compose exec backend python manage.py createsuperuser` |
| Django shell | `docker compose exec backend python manage.py shell` |
| Rebuild containers | `docker compose up -d --build` |
| Update all repos | `scripts\update.bat` |
| Build debug APK | `cd repos\grabakar-frontend\android && .\gradlew.bat assembleDebug` |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `docker compose` not recognized | Open Docker Desktop first, wait until it's running |
| Port 5432/6379 already in use | Stop local PostgreSQL/Redis, or change ports in `.env` |
| `npm install` fails on node-gyp | Run `npm install --global windows-build-tools` in admin PowerShell |
| Android build fails "SDK not found" | Set `ANDROID_HOME` env var (see Phase 1.4) |
| Android build fails "JDK not found" | Install JDK 17 (see Phase 1.5), set `JAVA_HOME` if needed |
| `gradlew.bat` permission denied | Run from `repos\grabakar-frontend\android\` directory directly |
| Backend 500 errors | Check `docker compose logs backend --tail 100` for tracebacks |
| Frontend blank page | Check browser console (F12), verify backend is healthy |
