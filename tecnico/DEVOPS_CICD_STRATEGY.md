# DevOps & CI/CD — GrabaKar (GCP Free Tier)

Implementable strategy for building, testing, and deploying GrabaKar to our $0/month GCP staging infrastructure. Every command, path, and service name in this document maps to real resources that already exist.

> **Scope:** This document covers the **staging** environment only. Production ($100-200/mo with managed services) will get its own addendum when the time comes.

**Related docs:**

| Document | Purpose |
|----------|---------|
| `tecnico/GCP_ARCHITECTURE.md` | What is deployed today (service names, IPs, buckets) |
| `tecnico/DEPLOYMENT.md` | Local Docker Compose, Dockerfile, env vars reference |
| `tecnico/SEGURIDAD.md` | Secrets management, OWASP considerations |
| `tecnico/GCP_DEPLOYMENT_PLAN.md` | Original GCP evaluation and infra setup plan |

---

## 1. Current State of the World

Before automating anything, here is exactly what exists in GCP project `grabakar-staging`:

| Resource | Type | Identifier |
|----------|------|------------|
| Backend API | Cloud Run | `grabakar-backend` (us-central1), URL: `https://grabakar-backend-1089044937741.us-central1.run.app` |
| Operator SPA | GCS bucket | `gs://grabakar-frontend-staging` |
| Admin SPA | GCS bucket | `gs://grabakar-admin-staging` |
| Media uploads | GCS bucket | `gs://grabakar-media-staging` |
| Docker images | Artifact Registry | `us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend` |
| Secrets | Secret Manager | `django-secret-key`, `db-password` |
| VM (Postgres/Redis/Celery) | Compute Engine | `grabakar-state-vm` (e2-micro), internal IP `10.128.0.2` |
| Networking | VPC connector | Cloud Run → default VPC → VM internal IP |

**What does NOT exist yet:**

- GitHub Actions workflows (all deploys are manual)
- Workload Identity Federation (or any CI ↔ GCP auth)
- Automated migrations
- Uptime monitoring / alerts
- Celery worker auto-update on deploy
- Frontend CI pipelines

---

## 2. Design Principles

| Principle | What it means for us |
|-----------|---------------------|
| **Build once, deploy many** | One Docker image per commit SHA. Same image runs on Cloud Run (API) and on the VM (Celery worker). Never rebuild for different targets. |
| **Immutable artifacts** | Backend: image tagged `sha-<7chars>`. Frontend: `dist/` bundles. No `:latest` in automation — only in manual dev workflows. |
| **Secrets never in code** | Local: `.env` (gitignored). GCP: Secret Manager. CI: GitHub encrypted secrets/variables. Zero JSON key files committed anywhere. |
| **Migrations block deploy** | New code never hits Cloud Run until migrations succeed against the staging DB. If migration fails, deploy aborts. |
| **Free tier awareness** | No Cloud SQL, no Memorystore, no Cloud Build. We use GitHub Actions runners (free for public repos, 2000 min/mo for private), `docker build` on the runner, and the e2-micro VM for all stateful services. |

---

## 3. Repository Map

```
grabakar-infra/           ← Local dev orchestration (Docker Compose)
├── repos/
│   ├── grabakar-backend/ ← Django API (has its own .git)
│   ├── grabakar-frontend/← Operator PWA (has its own .git)
│   └── grabakar-docs/    ← Documentation (has its own .git)
└── docker-compose.yml

grabakar-admin/           ← Admin SPA (separate repo outside grabakar-infra)
```

Each app repo owns its own CI/CD workflows:

| Repo | CI (on PR) | CD (on merge to `main`) |
|------|------------|------------------------|
| `grabakar-backend` | ruff lint + pytest | Build image → push AR → migrate on VM → deploy Cloud Run → update Celery on VM → health check |
| `grabakar-frontend` | tsc + build | Build dist → upload to GCS bucket |
| `grabakar-admin` | tsc + build | Build dist → upload to GCS bucket |

---

## 4. Branch Strategy

```
feature/xyz  ──PR──→  main  ──auto──→  staging (GCP)
                       │
                    tag v*.*.*  ──manual──→  production (future)
```

- **Feature branches**: All development happens here. PRs trigger CI checks.
- **`main`**: Protected branch. Merge triggers auto-deploy to staging.
- **Tags** (`v1.0.0`): Reserved for future production deploys via `workflow_dispatch`.

---

## 5. CI Pipelines (PR Checks)

These run on every pull request. Their only job is to catch problems before merge. They do NOT deploy anything.

### 5.1 Backend CI

```yaml
# grabakar-backend/.github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

concurrency:
  group: ci-backend-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install ruff
        run: pip install ruff==0.15.4
      - name: Lint
        run: ruff check .
      - name: Format check
        run: ruff format --check .

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: grabakar_test
          POSTGRES_USER: grabakar
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
    env:
      DJANGO_SETTINGS_MODULE: config.settings.test
      DJANGO_SECRET_KEY: ci-test-secret-key-not-real
      DB_NAME: grabakar_test
      DB_USER: grabakar
      DB_PASSWORD: test
      DB_HOST: localhost
      DB_PORT: 5432
      CELERY_BROKER_URL: redis://localhost:6379/0
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('requirements.txt') }}
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest --tb=short -q

  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image (verify Dockerfile is valid)
        run: docker build --target runtime -t grabakar-backend:ci-test .
```

### 5.2 Frontend CI

```yaml
# grabakar-frontend/.github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

concurrency:
  group: ci-frontend-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Type check
        run: npx tsc --noEmit
      - name: Lint
        run: npm run lint
      - name: Build (verifies production bundle works)
        run: npm run build
        env:
          VITE_API_URL: https://grabakar-backend-1089044937741.us-central1.run.app
```

### 5.3 Admin CI

```yaml
# grabakar-admin/.github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

concurrency:
  group: ci-admin-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Type check
        run: npx tsc --noEmit
      - name: Lint
        run: npm run lint
      - name: Build
        run: npm run build
        env:
          VITE_API_URL: https://grabakar-backend-1089044937741.us-central1.run.app
```

---

## 6. CD Pipeline — Backend

This is the most complex pipeline because it orchestrates: image build, push, database migration, Cloud Run deploy, Celery worker update, and health check.

### 6.1 GCP Authentication from GitHub Actions

We have two options. Start with **Option A** (simpler), migrate to **Option B** (more secure) when ready.

#### Option A — Service Account Key (quick start)

Create a deploy service account and export its JSON key:

```bash
# One-time setup on your machine
gcloud iam service-accounts create github-deploy \
  --display-name="GitHub Actions Deploy"

# Grant the minimum roles it needs
SA=github-deploy@grabakar-staging.iam.gserviceaccount.com

gcloud projects add-iam-policy-binding grabakar-staging \
  --member="serviceAccount:$SA" --role="roles/run.admin"
gcloud projects add-iam-policy-binding grabakar-staging \
  --member="serviceAccount:$SA" --role="roles/artifactregistry.writer"
gcloud projects add-iam-policy-binding grabakar-staging \
  --member="serviceAccount:$SA" --role="roles/storage.objectAdmin"
gcloud projects add-iam-policy-binding grabakar-staging \
  --member="serviceAccount:$SA" --role="roles/secretmanager.secretAccessor"
gcloud projects add-iam-policy-binding grabakar-staging \
  --member="serviceAccount:$SA" --role="roles/compute.instanceAdmin.v1"
gcloud projects add-iam-policy-binding grabakar-staging \
  --member="serviceAccount:$SA" --role="roles/iam.serviceAccountUser"

# Export key JSON — store this as a GitHub secret
gcloud iam service-accounts keys create /tmp/github-deploy-key.json \
  --iam-account=$SA

# IMPORTANT: after uploading to GitHub Secrets, delete the local file
rm /tmp/github-deploy-key.json
```

In GitHub repo settings → Secrets → `GCP_SA_KEY`: paste the JSON contents.

#### Option B — Workload Identity Federation (recommended upgrade)

Eliminates long-lived JSON keys entirely. GitHub Actions gets short-lived OIDC tokens.

```bash
PROJECT_ID=grabakar-staging
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
SA=github-deploy@${PROJECT_ID}.iam.gserviceaccount.com

# Create workload identity pool
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool"

# Create OIDC provider
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --workload-identity-pool=github-pool \
  --location=global \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

# Allow the GitHub repo to impersonate the SA
# Replace GITHUB_ORG/GITHUB_REPO with your actual values (e.g. grabakar/grabakar-backend)
gcloud iam service-accounts add-iam-policy-binding $SA \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/attribute.repository/GITHUB_ORG/GITHUB_REPO"
```

### 6.2 Backend Deploy Workflow

```yaml
# grabakar-backend/.github/workflows/deploy.yml
name: Deploy Backend

on:
  push:
    branches: [main]
  workflow_dispatch: {}

concurrency:
  group: deploy-backend-staging
  cancel-in-progress: false

env:
  GCP_PROJECT: grabakar-staging
  REGION: us-central1
  AR_REGISTRY: us-central1-docker.pkg.dev
  AR_IMAGE: us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend
  VM_NAME: grabakar-state-vm
  VM_ZONE: us-central1-a

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      # ── 1. Checkout ──
      - uses: actions/checkout@v4

      - name: Set image tag from git SHA
        id: meta
        run: echo "tag=sha-$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

      # ── 2. Authenticate to GCP ──
      # Option A: service account key
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      # Option B (WIF): uncomment these lines and remove the above
      #   workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
      #   service_account: ${{ vars.GCP_SA_EMAIL }}

      - uses: google-github-actions/setup-gcloud@v2

      # ── 3. Build and push Docker image ──
      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ env.AR_REGISTRY }} --quiet

      - name: Build image (linux/amd64)
        run: |
          docker build \
            --platform linux/amd64 \
            -t ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }} \
            .

      - name: Push image
        run: docker push ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }}

      # ── 4. Run database migrations on the VM ──
      - name: Run migrations via VM
        run: |
          gcloud compute ssh ${{ env.VM_NAME }} \
            --zone=${{ env.VM_ZONE }} \
            --command="
              docker pull ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }} && \
              docker run --rm --network host \
                -e DJANGO_SETTINGS_MODULE=config.settings.production \
                -e DB_HOST=127.0.0.1 \
                -e DB_PORT=5432 \
                -e DB_NAME=grabakar \
                -e DB_USER=grabakar \
                -e DB_PASSWORD=\$(docker exec grabakar-postgres printenv POSTGRES_PASSWORD) \
                -e DJANGO_SECRET_KEY=migration-runner \
                -e CELERY_BROKER_URL=redis://127.0.0.1:6379/0 \
                -e REDIS_URL=redis://127.0.0.1:6379/0 \
                ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }} \
                python manage.py migrate --noinput
            "

      # ── 5. Deploy to Cloud Run ──
      - name: Deploy Cloud Run service
        run: |
          gcloud run deploy grabakar-backend \
            --image ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }} \
            --region ${{ env.REGION }} \
            --platform managed \
            --allow-unauthenticated \
            --network default \
            --subnet default \
            --vpc-egress private-ranges-only \
            --min-instances 0 \
            --max-instances 2 \
            --memory 512Mi \
            --cpu 1

      # ── 6. Update Celery worker on VM ──
      - name: Update Celery worker on VM
        run: |
          gcloud compute ssh ${{ env.VM_NAME }} \
            --zone=${{ env.VM_ZONE }} \
            --command="
              docker stop grabakar-celery-worker 2>/dev/null || true && \
              docker rm grabakar-celery-worker 2>/dev/null || true && \
              docker run -d --name grabakar-celery-worker --restart unless-stopped \
                --network host \
                -e DJANGO_SETTINGS_MODULE=config.settings.production \
                -e DB_HOST=127.0.0.1 \
                -e DB_PORT=5432 \
                -e DB_NAME=grabakar \
                -e DB_USER=grabakar \
                -e DB_PASSWORD=\$(docker exec grabakar-postgres printenv POSTGRES_PASSWORD) \
                -e DJANGO_SECRET_KEY=\$(gcloud secrets versions access latest --secret=django-secret-key) \
                -e CELERY_BROKER_URL=redis://127.0.0.1:6379/0 \
                -e REDIS_URL=redis://127.0.0.1:6379/0 \
                ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }} \
                celery -A config worker -l info
            "

      # ── 7. Health check ──
      - name: Verify deployment
        run: |
          sleep 10
          URL=$(gcloud run services describe grabakar-backend \
            --region ${{ env.REGION }} \
            --format 'value(status.url)')
          echo "Checking $URL/api/v1/health/"
          curl -sf --retry 3 --retry-delay 5 "$URL/api/v1/health/"
          echo ""
          echo "Deploy successful: ${{ env.AR_IMAGE }}:${{ steps.meta.outputs.tag }}"
```

### 6.3 How This Workflow Interacts With the VM

The deploy pipeline SSHs into `grabakar-state-vm` twice:

1. **Migration step:** Pulls the new image, runs `manage.py migrate` as a one-off container connected to the local Postgres via `--network host`.
2. **Celery update step:** Stops the old worker container, starts a new one from the same image with the same env vars.

This works because:
- The VM is in the same VPC as Cloud Run.
- `gcloud compute ssh` uses IAM — no SSH keys to manage.
- The migration container is ephemeral (`--rm`), so it doesn't consume the e2-micro's limited RAM after it finishes.
- The Celery worker is long-running (`-d`, `--restart unless-stopped`).

---

## 7. CD Pipeline — Frontend SPAs

Both frontends follow the same pattern: build static files, sync to a GCS bucket.

### 7.1 Operator Frontend Deploy

```yaml
# grabakar-frontend/.github/workflows/deploy.yml
name: Deploy Frontend

on:
  push:
    branches: [main]
  workflow_dispatch: {}

concurrency:
  group: deploy-frontend-staging
  cancel-in-progress: false

env:
  GCS_BUCKET: grabakar-frontend-staging
  VITE_API_URL: https://grabakar-backend-1089044937741.us-central1.run.app

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build production bundle
        run: npm run build
        env:
          VITE_API_URL: ${{ env.VITE_API_URL }}

      # Authenticate to GCP
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - uses: google-github-actions/setup-gcloud@v2

      # Sync dist/ to bucket (deletes stale files)
      - name: Upload to GCS
        run: |
          gcloud storage rsync dist/ gs://${{ env.GCS_BUCKET }}/ \
            --recursive \
            --delete-unmatched-destination-objects

      # Set cache headers: HTML=no-cache, assets=long-cache
      - name: Set cache metadata
        run: |
          gcloud storage objects update "gs://${{ env.GCS_BUCKET }}/index.html" \
            --cache-control="no-cache, max-age=0"
          gcloud storage objects update "gs://${{ env.GCS_BUCKET }}/assets/**" \
            --cache-control="public, max-age=31536000, immutable" 2>/dev/null || true

      - name: Verify deployment
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            "https://storage.googleapis.com/${{ env.GCS_BUCKET }}/index.html")
          if [ "$STATUS" != "200" ]; then
            echo "ERROR: Frontend returned HTTP $STATUS"
            exit 1
          fi
          echo "Frontend deploy verified (HTTP 200)"
```

### 7.2 Admin Panel Deploy

```yaml
# grabakar-admin/.github/workflows/deploy.yml
name: Deploy Admin

on:
  push:
    branches: [main]
  workflow_dispatch: {}

concurrency:
  group: deploy-admin-staging
  cancel-in-progress: false

env:
  GCS_BUCKET: grabakar-admin-staging
  VITE_API_URL: https://grabakar-backend-1089044937741.us-central1.run.app

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build production bundle
        run: npm run build
        env:
          VITE_API_URL: ${{ env.VITE_API_URL }}

      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - uses: google-github-actions/setup-gcloud@v2

      - name: Upload to GCS
        run: |
          gcloud storage rsync dist/ gs://${{ env.GCS_BUCKET }}/ \
            --recursive \
            --delete-unmatched-destination-objects

      - name: Set cache metadata
        run: |
          gcloud storage objects update "gs://${{ env.GCS_BUCKET }}/index.html" \
            --cache-control="no-cache, max-age=0"
          gcloud storage objects update "gs://${{ env.GCS_BUCKET }}/assets/**" \
            --cache-control="public, max-age=31536000, immutable" 2>/dev/null || true

      - name: Verify deployment
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            "https://storage.googleapis.com/${{ env.GCS_BUCKET }}/index.html")
          if [ "$STATUS" != "200" ]; then
            echo "ERROR: Admin panel returned HTTP $STATUS"
            exit 1
          fi
          echo "Admin deploy verified (HTTP 200)"
```

---

## 8. Database Migrations

### 8.1 Strategy

Migrations are a **first-class deploy step**. They run on the VM (where Postgres lives) before Cloud Run gets the new image.

```
Developer          GitHub Actions              VM (e2-micro)            Cloud Run
    │                    │                          │                      │
    │  makemigrations    │                          │                      │
    │  commit .py file   │                          │                      │
    │──── push to main ──│                          │                      │
    │                    │── build & push image ────│                      │
    │                    │── SSH: pull image ───────│                      │
    │                    │── SSH: migrate ──────────│── runs against ──────│
    │                    │                          │   local Postgres     │
    │                    │                          │                      │
    │                    │   (migration succeeds?)  │                      │
    │                    │── YES: deploy Cloud Run ─│──────────────────────│
    │                    │── NO: pipeline FAILS ────│                      │
```

### 8.2 Safety Rules

1. **Migration files are always committed.** Never auto-generate in CI. `makemigrations` runs on your machine, you review the `.py`, you commit it.
2. **Forward-only.** Rollback = write a new migration that reverses the change.
3. **Two-phase schema changes.** To remove a column:
   - Deploy 1: Remove all code that references the column. Keep the column in the DB.
   - Deploy 2: Add a migration that drops the column.
4. **No data migrations in the same PR as schema changes.** Separate them for clarity.

### 8.3 Manual Migration (escape hatch)

If you need to run a migration outside of CI (e.g., emergency fix):

```bash
# SSH into the VM
gcloud compute ssh grabakar-state-vm --zone=us-central1-a

# Pull the latest image
docker pull us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:latest

# Run migration as a one-off container
docker run --rm --network host \
  -e DJANGO_SETTINGS_MODULE=config.settings.production \
  -e DB_HOST=127.0.0.1 -e DB_PORT=5432 \
  -e DB_NAME=grabakar -e DB_USER=grabakar \
  -e DB_PASSWORD=$(docker exec grabakar-postgres printenv POSTGRES_PASSWORD) \
  -e DJANGO_SECRET_KEY=manual-migration \
  -e CELERY_BROKER_URL=redis://127.0.0.1:6379/0 \
  -e REDIS_URL=redis://127.0.0.1:6379/0 \
  us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:latest \
  python manage.py migrate --noinput
```

---

## 9. Secrets & Configuration

### 9.1 Three Layers

| Context | Tool | Example |
|---------|------|---------|
| Local dev | `.env` file (gitignored) | `DB_PASSWORD=grabakar_dev` |
| GitHub Actions | Repository secrets + env vars | `secrets.GCP_SA_KEY` |
| GCP runtime | Secret Manager + env vars in `env.yaml` | `django-secret-key`, `db-password` |

### 9.2 What's in Secret Manager Today

| Secret name | Used by | How it's mounted |
|-------------|---------|-----------------|
| `django-secret-key` | Cloud Run | `--set-secrets DJANGO_SECRET_KEY=django-secret-key:latest` |
| `db-password` | Cloud Run | `--set-secrets DB_PASSWORD=db-password:latest` |

### 9.3 What Goes in GitHub Secrets

| GitHub Secret | Purpose |
|---------------|---------|
| `GCP_SA_KEY` | Service account JSON for `google-github-actions/auth@v2` |

No other secrets are needed. The deploy workflow reads the DB password from the VM's Postgres container at migration time (via `docker exec`), and Cloud Run reads secrets from Secret Manager at runtime.

### 9.4 What's in Cloud Run's `env.yaml`

The backend deploys with `--env-vars-file env.yaml`. This file is NOT committed (it contains the VM internal IP and other infra details). Keep it in the GCP Console or store it locally for manual deploys:

```yaml
# env.yaml (used only for gcloud run deploy, NOT committed)
DJANGO_SETTINGS_MODULE: config.settings.production
DJANGO_DEBUG: "False"
DJANGO_ALLOWED_HOSTS: "*"
DB_NAME: grabakar
DB_USER: grabakar
DB_HOST: "10.128.0.2"
DB_PORT: "5432"
CELERY_BROKER_URL: "redis://10.128.0.2:6379/0"
REDIS_URL: "redis://10.128.0.2:6379/0"
CORS_ALLOWED_ORIGINS: "https://storage.googleapis.com"
GS_BUCKET_NAME: grabakar-media-staging
DEFAULT_FILE_STORAGE: "storages.backends.gcloud.GoogleCloudStorage"
```

> **Note:** For the automated deploy workflow (section 6.2), the env vars are already set on the Cloud Run service from the initial manual deploy. The `gcloud run deploy` command in CI only updates the image — existing env vars and secrets persist across revisions unless you explicitly change them.

### 9.5 Rotation Schedule

| Secret | Rotate every | How |
|--------|-------------|-----|
| `django-secret-key` | 90 days | `gcloud secrets versions add django-secret-key --data-file=-` then redeploy |
| `db-password` | 90 days | Update in Postgres, update in Secret Manager, redeploy |
| `GCP_SA_KEY` | 180 days (or migrate to WIF) | Regenerate key, update GitHub secret |

---

## 10. VM Management

The e2-micro VM (`grabakar-state-vm`) is the backbone of the free-tier architecture. It runs three long-lived Docker containers.

### 10.1 Container Inventory

| Container name | Image | Ports | Restart policy |
|---------------|-------|-------|---------------|
| `grabakar-postgres` | `postgres:16` | `5432` | `unless-stopped` |
| `grabakar-redis` | `redis:7` | `6379` | `unless-stopped` |
| `grabakar-celery-worker` | Same as Cloud Run backend image | N/A (connects to localhost) | `unless-stopped` |

### 10.2 VM Health Check Script (Repo-backed)

The script lives in this repo at: `grabakar-infra/scripts/vm_health_check.sh`.

Copy it to the VM (or symlink) and run via cron:

```bash
# Example: every 5 minutes
*/5 * * * * /home/$USER/grabakar-infra/scripts/vm_health_check.sh >> /home/$USER/grabakar-vm-health.log 2>&1
```

Script content (keep as-is):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Basic health checks for the free-tier state VM containers.
# Intended for cron, e.g.:
#   */5 * * * * /home/<user>/grabakar-infra/scripts/vm_health_check.sh >> /var/log/grabakar-vm-health.log 2>&1

log() {
  echo "[$(date -Iseconds)] $*"
}

restart_if_down() {
  local container_name="$1"
  local check_cmd="$2"

  if ! docker exec "$container_name" bash -lc "$check_cmd" >/dev/null 2>&1; then
    log "UNHEALTHY: $container_name -> restarting"
    docker restart "$container_name" >/dev/null 2>&1 || true
  else
    log "OK: $container_name"
  fi
}

log "VM health check starting"

# Postgres
restart_if_down "grabakar-postgres" "pg_isready -U grabakar"

# Redis
restart_if_down "grabakar-redis" "redis-cli ping"

# Celery worker (container may not be present on first boot)
CELERY_RUNNING_IDS="$(docker ps -q --filter "name=grabakar-celery-worker" --filter "status=running" || true)"
if [ -z "$CELERY_RUNNING_IDS" ]; then
  log "UNHEALTHY: grabakar-celery-worker not running -> restart/initial start required"
  # Restart only if the container exists; otherwise create/start should be handled by CD.
  if docker ps -a -q --filter "name=grabakar-celery-worker" >/dev/null 2>&1; then
    docker restart grabakar-celery-worker >/dev/null 2>&1 || true
  fi
else
  log "OK: grabakar-celery-worker running"
fi

log "VM health check finished"
```

### 10.3 VM Backup (Postgres) (Repo-backed)

The script lives in this repo at: `grabakar-infra/scripts/vm_backup_postgres.sh`.

Example cron (daily 03:00):

```bash
0 3 * * * /home/$USER/grabakar-infra/scripts/vm_backup_postgres.sh >> /home/$USER/grabakar-vm-backup.log 2>&1
```

Script content:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Free-tier VM backup: dump Postgres from the running container and upload to GCS.
#
# Intended usage (cron on the VM):
#   0 3 * * * /home/<user>/grabakar-infra/scripts/vm_backup_postgres.sh
#
# Required on the VM:
# - docker running `grabakar-postgres`
# - gcloud auth configured for a service account that can write to the media bucket

log() {
  echo "[$(date -Iseconds)] $*"
}

POSTGRES_CONTAINER="${POSTGRES_CONTAINER:-grabakar-postgres}"
DB_NAME="${DB_NAME:-grabakar}"
DB_USER="${DB_USER:-grabakar}"

# Example: gs://grabakar-media-staging/backups
GCS_BACKUP_PREFIX="${GCS_BACKUP_PREFIX:-gs://grabakar-media-staging/backups}"

BACKUP_KEEP_DAYS="${BACKUP_KEEP_DAYS:-7}"

TMP_FILE="/tmp/${DB_NAME}-backup-$(date +%Y%m%d).sql.gz"
OBJECT_NAME="$(basename "$TMP_FILE")"
DEST_URI="${GCS_BACKUP_PREFIX}/${OBJECT_NAME}"

log "Starting Postgres backup (container=${POSTGRES_CONTAINER}, db=${DB_NAME})"

# Dump directly from the container. The container already has POSTGRES_PASSWORD in its env.
docker exec "$POSTGRES_CONTAINER" bash -lc "pg_dump -U '$DB_USER' '$DB_NAME'" | gzip -c > "$TMP_FILE"

log "Uploading backup to: $DEST_URI"
gcloud storage cp "$TMP_FILE" "$DEST_URI"

rm -f "$TMP_FILE"

log "Applying retention policy (keep last ${BACKUP_KEEP_DAYS} days)"

# Delete backups older than KEEP_DAYS (based on filename YYYYMMDD).
NOW_EPOCH="$(date +%s)"

gcloud storage ls "${GCS_BACKUP_PREFIX}/" --format="value(name)" | while read -r full; do
  base="$(basename "$full")"
  # Expect: <db>-backup-YYYYMMDD.sql.gz
  date_part="$(echo "$base" | sed -E 's/.*-backup-([0-9]{8})\\.sql\\.gz/\\1/')"

  # Skip unexpected names
  if [[ ! "$date_part" =~ ^[0-9]{8}$ ]]; then
    continue
  fi

  backup_iso="${date_part:0:4}-${date_part:4:2}-${date_part:6:2}"
  backup_epoch="$(date -d "$backup_iso" +%s || true)"
  if [ -z "$backup_epoch" ]; then
    continue
  fi

  age_days="$(( (NOW_EPOCH - backup_epoch) / 86400 ))"
  if [ "$age_days" -gt "$BACKUP_KEEP_DAYS" ]; then
    log "Deleting old backup: $full (age=${age_days}d)"
    gcloud storage rm -q "$full"
  fi
done

log "Backup finished"
```

### 10.4 VM Disaster Recovery

If the VM dies or gets corrupted:

```bash
# 1. Recreate VM
gcloud compute instances create grabakar-state-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --boot-disk-size=30GB \
  --boot-disk-type=pd-standard

# 2. SSH in and install Docker
gcloud compute ssh grabakar-state-vm --zone=us-central1-a
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 3. Start Postgres
docker run -d --name grabakar-postgres --restart unless-stopped \
  -e POSTGRES_DB=grabakar \
  -e POSTGRES_USER=grabakar \
  -e POSTGRES_PASSWORD=<from-secret-manager> \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# 4. Restore from backup
gcloud storage cp gs://grabakar-media-staging/backups/grabakar-LATEST.sql.gz /tmp/
gunzip /tmp/grabakar-LATEST.sql.gz
docker exec -i grabakar-postgres psql -U grabakar grabakar < /tmp/grabakar-LATEST.sql

# 5. Start Redis
docker run -d --name grabakar-redis --restart unless-stopped \
  -p 6379:6379 \
  redis:7

# 6. Start Celery worker
docker run -d --name grabakar-celery-worker --restart unless-stopped \
  --network host \
  -e DJANGO_SETTINGS_MODULE=config.settings.production \
  -e DB_HOST=127.0.0.1 -e DB_PORT=5432 \
  -e DB_NAME=grabakar -e DB_USER=grabakar \
  -e DB_PASSWORD=<from-secret-manager> \
  -e DJANGO_SECRET_KEY=$(gcloud secrets versions access latest --secret=django-secret-key) \
  -e CELERY_BROKER_URL=redis://127.0.0.1:6379/0 \
  -e REDIS_URL=redis://127.0.0.1:6379/0 \
  us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:latest \
  celery -A config worker -l info

# 7. Note the new internal IP and update Cloud Run env vars if it changed
gcloud compute instances describe grabakar-state-vm \
  --zone=us-central1-a \
  --format='get(networkInterfaces[0].networkIP)'
```

---

## 11. Monitoring & Observability ($0)

### 11.1 Free GCP Monitoring

| What | Service | How to access |
|------|---------|---------------|
| API request logs | Cloud Logging | `gcloud logging read 'resource.type="cloud_run_revision"'` or GCP Console → Logging |
| Error traces | Cloud Error Reporting | Auto-captured from Cloud Run stderr. GCP Console → Error Reporting |
| Latency / request count | Cloud Run metrics | GCP Console → Cloud Run → `grabakar-backend` → Metrics tab |
| Uptime | Cloud Monitoring uptime check | See setup below |

### 11.2 Uptime Check Setup

```bash
# Free: up to 10 uptime checks
gcloud monitoring uptime-check-configs create http \
  grabakar-api-health \
  --display-name="GrabaKar API Health" \
  --resource-type=cloud-run-revision \
  --cloud-run-service-name=grabakar-backend \
  --cloud-run-service-region=us-central1 \
  --http-check-request-method=GET \
  --http-check-path="/api/v1/health/" \
  --period=300
```

### 11.3 Application Logging

Django logs to stdout → Cloud Logging picks it up automatically (via `production.py` LOGGING config).

To query logs:

```bash
# Recent errors
gcloud logging read \
  'resource.type="cloud_run_revision" AND severity>=ERROR' \
  --limit=20 --format=json

# Specific request
gcloud logging read \
  'resource.type="cloud_run_revision" AND httpRequest.requestUrl="/api/v1/sync/upload/"' \
  --limit=10
```

### 11.4 VM Monitoring

The e2-micro VM gets basic Compute Engine metrics for free: CPU, disk, network. Accessible in GCP Console → Compute Engine → `grabakar-state-vm` → Monitoring tab.

For Postgres/Redis container health, rely on the cron health check script (section 10.2).

---

## 12. Rollback Procedures

### 12.1 Backend Rollback (Cloud Run)

Cloud Run keeps every revision. Rollback is instant — just shift traffic.

```bash
# See recent revisions
gcloud run revisions list --service grabakar-backend --region us-central1 --limit=5

# Route 100% traffic to a previous revision
gcloud run services update-traffic grabakar-backend \
  --to-revisions grabakar-backend-XXXXX=100 \
  --region us-central1
```

> **Warning:** If the new code came with a database migration, rolling back the code without rolling back the migration can cause errors. Only do this if the migration was additive (new columns, new tables) — never if it was destructive.

Also rollback the Celery worker on the VM:

```bash
gcloud compute ssh grabakar-state-vm --zone=us-central1-a --command="
  docker stop grabakar-celery-worker && \
  docker rm grabakar-celery-worker && \
  docker run -d --name grabakar-celery-worker --restart unless-stopped \
    --network host \
    -e DJANGO_SETTINGS_MODULE=config.settings.production \
    -e DB_HOST=127.0.0.1 -e DB_PORT=5432 \
    -e DB_NAME=grabakar -e DB_USER=grabakar \
    -e DB_PASSWORD=\$(docker exec grabakar-postgres printenv POSTGRES_PASSWORD) \
    -e DJANGO_SECRET_KEY=\$(gcloud secrets versions access latest --secret=django-secret-key) \
    -e CELERY_BROKER_URL=redis://127.0.0.1:6379/0 \
    -e REDIS_URL=redis://127.0.0.1:6379/0 \
    us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:<PREVIOUS_TAG> \
    celery -A config worker -l info
"
```

### 12.2 Frontend Rollback (GCS)

GCS doesn't keep version history by default. Two options:

**Option A: Re-run the deploy workflow for a previous commit.**

```bash
# In GitHub Actions → Deploy Frontend → Run workflow → pick a branch/commit
```

**Option B: Enable GCS object versioning (recommended).**

```bash
# One-time setup
gcloud storage buckets update gs://grabakar-frontend-staging --versioning
gcloud storage buckets update gs://grabakar-admin-staging --versioning

# To rollback: list versions of index.html and restore
gcloud storage ls -la gs://grabakar-frontend-staging/index.html
gcloud storage cp gs://grabakar-frontend-staging/index.html#GENERATION gs://grabakar-frontend-staging/index.html
```

---

## 13. Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DEVELOPER WORKFLOW                              │
│                                                                         │
│  1. Code on feature branch                                              │
│  2. Push → PR to main                                                   │
│  3. CI checks run (lint, test, build)                                   │
│  4. Code review + approve                                               │
│  5. Merge to main → CD triggers automatically                           │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        GITHUB ACTIONS (CD)                              │
│                                                                         │
│  Backend:                          Frontend / Admin:                     │
│  ┌──────────────────────────┐      ┌───────────────────────────┐        │
│  │ 1. docker build          │      │ 1. npm ci                 │        │
│  │ 2. docker push → AR      │      │ 2. npm run build          │        │
│  │ 3. SSH VM: migrate       │      │ 3. gcloud storage rsync   │        │
│  │ 4. gcloud run deploy     │      │ 4. curl verify            │        │
│  │ 5. SSH VM: restart celery│      └──────────────┬────────────┘        │
│  │ 6. curl health check     │                     │                     │
│  └────────────┬─────────────┘                     │                     │
│               │                                   │                     │
└───────────────┼───────────────────────────────────┼─────────────────────┘
                │                                   │
                ▼                                   ▼
┌───────────────────────────┐  ┌──────────────────────────────────────────┐
│      Cloud Run            │  │        Google Cloud Storage              │
│  grabakar-backend         │  │                                          │
│  (scales 0 → 2)           │  │  gs://grabakar-frontend-staging          │
│  sha-abc1234              │  │  gs://grabakar-admin-staging             │
│                           │  │  (public, SPA index.html routing)        │
└─────────────┬─────────────┘  └──────────────────────────────────────────┘
              │
              │  VPC (private IP)
              ▼
┌─────────────────────────────────┐
│  grabakar-state-vm (e2-micro)   │
│  10.128.0.2                     │
│                                 │
│  ┌─────────────┐ ┌───────────┐ │
│  │ postgres:16 │ │ redis:7   │ │
│  │ port 5432   │ │ port 6379 │ │
│  └─────────────┘ └───────────┘ │
│  ┌─────────────────────────────┐│
│  │ grabakar-celery-worker      ││
│  │ (same image as Cloud Run)   ││
│  └─────────────────────────────┘│
└─────────────────────────────────┘
```

---

## 14. Manual Deploy Cheat Sheet

For when you need to deploy without CI (e.g., CI is broken, or first-time setup):

### Backend

```bash
cd grabakar-backend

# 1. Build for Cloud Run's architecture
docker build --platform linux/amd64 \
  -t us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:manual-$(date +%Y%m%d) .

# 2. Push to Artifact Registry
docker push us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:manual-$(date +%Y%m%d)

# 3. Run migrations (SSH into VM)
gcloud compute ssh grabakar-state-vm --zone=us-central1-a
# (then run the docker run --rm migrate command from section 8.3)

# 4. Deploy to Cloud Run
gcloud run deploy grabakar-backend \
  --image us-central1-docker.pkg.dev/grabakar-staging/grabakar/backend:manual-$(date +%Y%m%d) \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --network default --subnet default --vpc-egress private-ranges-only

# 5. Update Celery on VM (SSH in, stop old, start new)
```

### Frontends

```bash
# Operator
cd grabakar-frontend
VITE_API_URL=https://grabakar-backend-1089044937741.us-central1.run.app npm run build
gcloud storage rsync dist/ gs://grabakar-frontend-staging/ --recursive --delete-unmatched-destination-objects

# Admin
cd grabakar-admin
VITE_API_URL=https://grabakar-backend-1089044937741.us-central1.run.app npm run build
gcloud storage rsync dist/ gs://grabakar-admin-staging/ --recursive --delete-unmatched-destination-objects
```

---

## 15. GitHub Repository Settings

For each repo (`grabakar-backend`, `grabakar-frontend`, `grabakar-admin`):

### Required Settings

| Setting | Value | Why |
|---------|-------|-----|
| Default branch | `main` | All CD workflows trigger on push to `main` |
| Branch protection on `main` | Require PR, require CI to pass | Prevents broken code from reaching staging |
| Repository secrets | `GCP_SA_KEY` | Auth for deploy workflows |

### Recommended Settings

| Setting | Value |
|---------|-------|
| Require linear history | Squash merges only (cleaner deploy log) |
| Auto-delete head branches | On (keeps repo clean) |
| Require up-to-date branches | On (ensures CI ran against latest main) |

---

## 16. Cost Breakdown

Everything described in this document costs $0/month:

| Resource | Free tier limit | Our usage |
|----------|----------------|-----------|
| **Cloud Run** | 2M requests/mo, 360k vCPU-seconds | Well under (scales to zero) |
| **Compute Engine** | 1 e2-micro in US region, 30GB disk | Exactly 1 e2-micro |
| **Cloud Storage** | 5GB, 50k reads, 5k writes/mo | ~100MB across 3 buckets |
| **Artifact Registry** | 500MB | ~300MB (one backend image) |
| **Secret Manager** | 6 active versions, 10k accesses | 2 secrets |
| **Cloud Logging** | 50GB/mo | Negligible |
| **Cloud Monitoring** | 10 uptime checks | 1 uptime check |
| **GitHub Actions** | 2000 min/mo (private), unlimited (public) | ~10-15 min/deploy |

---

## 17. Implementation Checklist

### Phase 1 — GCP Auth Setup (30 min)

- [ ] Create `github-deploy` service account with required roles
- [ ] Export key JSON and add as `GCP_SA_KEY` in all three GitHub repos
- [ ] Delete local copy of key JSON
- [ ] Verify: `gcloud auth activate-service-account --key-file=...` works

### Phase 2 — CI Pipelines (1 hour)

- [ ] Create `grabakar-backend/.github/workflows/ci.yml` (section 5.1)
- [ ] Create `grabakar-frontend/.github/workflows/ci.yml` (section 5.2)
- [ ] Create `grabakar-admin/.github/workflows/ci.yml` (section 5.3)
- [ ] Open a test PR in each repo to verify CI passes
- [ ] Enable branch protection on `main`: require CI to pass

### Phase 3 — CD Pipelines (2 hours)

- [ ] Create `grabakar-backend/.github/workflows/deploy.yml` (section 6.2)
- [ ] Create `grabakar-frontend/.github/workflows/deploy.yml` (section 7.1)
- [ ] Create `grabakar-admin/.github/workflows/deploy.yml` (section 7.2)
- [ ] Test: merge a trivial change to `main` in each repo, verify deploy succeeds
- [ ] Verify: health check passes, frontends load in browser

### Phase 4 — Staging Bootstrap (30 min)

- [ ] Create `grabakar-backend/.github/workflows/bootstrap-staging.yml`
- [ ] Run it once from the GitHub UI to ensure seeded users exist in staging
- [ ] Prefer `load_fixtures=false` on repeated runs
- [ ] Only use `reset_passwords=true` if you want to force known passwords

### Phase 5 — VM Hardening (1 hour)

- [ ] SSH into VM, install health check script (section 10.2)
- [ ] Configure health check cron: `*/5 * * * * /home/$USER/grabakar-infra/scripts/vm_health_check.sh`
- [ ] Install DB backup script (section 10.3)
- [ ] Configure backup cron: `0 3 * * * /home/$USER/grabakar-infra/scripts/vm_backup_postgres.sh`
- [ ] Verify: backup appears in `gs://grabakar-media-staging/backups/`

### Phase 6 — Monitoring (30 min)

- [ ] Create uptime check for `/api/v1/health/` (section 11.2)
- [ ] Enable GCS object versioning on frontend and admin buckets
- [ ] Optional: set up email alert notification channel for uptime failures

### Estimated Total: ~5 hours

---

## 18. Future Improvements (When Leaving Free Tier)

| Improvement | Trigger | Effort |
|------------|---------|--------|
| Workload Identity Federation | Security audit / team grows | 1 hour |
| Cloud SQL (managed Postgres) | Production readiness | 2 hours + $50-80/mo |
| Memorystore (managed Redis) | Production readiness | 1 hour + $30/mo |
| Cloud CDN + custom domains | Customer-facing URLs needed | 2 hours + $5/mo |
| Cloud Run min-instances=1 | Cold start latency unacceptable | Config change + $20/mo |
| Separate production GCP project | Live customers | 3 hours |
| Canary deploys (Cloud Run traffic splitting) | Risk-averse deploys | 1 hour |
| Cloud Run Jobs for migrations | Cleaner than SSH | 1 hour |
