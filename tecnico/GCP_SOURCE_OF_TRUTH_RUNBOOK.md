# GCP as Source of Truth — Remaining Work Runbook

Goal: make **GCP staging** (Cloud Run + VM Postgres/Redis + GCS) the real source of truth, with minimal/no reliance on your local machine for operational work.

This runbook lists the remaining work after the CI/CD and frontend printing fixes.

---

## 1. GitHub -> GCP wiring (one-time)
1. Add the GitHub encrypted secret `GCP_SA_KEY` to:
   - `grabakar-backend`
   - `grabakar-frontend`
   - `grabakar-admin`
2. Ensure the service account behind `GCP_SA_KEY` has permissions for:
   - Artifact Registry: push images to `us-central1-docker.pkg.dev/grabakar-staging/...`
   - Cloud Run: deploy `grabakar-backend`
   - Secret Manager: read `django-secret-key` and `db-password`
   - VM access: `gcloud compute ssh` (and permission to run Docker commands on the VM)
   - GCS buckets: upload `grabakar-frontend-staging` and `grabakar-admin-staging`

---

## 2. Bootstrap staging DB users from GitHub Actions (no local machine)
### What was added
1. Seeder flags: `seed_dev_data` is now re-runnable and supports:
   - `--skip-fixtures` (does not reload `initial.json`)
   - `--reset-passwords` (if users already exist, sets password back to `DEFAULT_PASSWORD`)
2. Workflow: `grabakar-backend/.github/workflows/bootstrap-staging.yml`
   - Builds and pushes the backend image
   - SSHes into `grabakar-state-vm`
   - Runs `python manage.py seed_dev_data ...` against the VM Postgres

### What you do
1. Go to `grabakar-backend` -> **Actions** -> **“Bootstrap Staging (Seed Users)”**
2. Use safe defaults first:
   - `load_fixtures = false`
   - `reset_passwords = false`
3. If staging is completely empty (no tenant/reference rows yet), run once with:
   - `load_fixtures = true`
   - `reset_passwords = false` (or `true` only if you want to force known passwords)

### Expected seeded users (created or aligned)
All with password `grabakar123` (the seeder default):
- `admin` — role `admin` (**is_staff=True**; required for admin panel access)
- `supervisor` — role `supervisor`
- `operador` — role `operador`
- `test` — role `operador`

### Verify after the workflow finishes
1. Operator login: test with `operador / grabakar123` and `supervisor / grabakar123`
2. Admin login: test with `admin / grabakar123` and confirm admin panel access.

---

## 3. VM operations without drift (health + backups)
### What was added to the repo
- `grabakar-infra/scripts/vm_health_check.sh`
- `grabakar-infra/scripts/vm_backup_postgres.sh`

### What remains on the VM (you do once)
1. Copy/symlink scripts to the VM:
   - Place at: `/home/$USER/grabakar-infra/scripts/...` (recommended)
2. Ensure they are executable:
   - `chmod +x /home/$USER/grabakar-infra/scripts/vm_health_check.sh`
   - `chmod +x /home/$USER/grabakar-infra/scripts/vm_backup_postgres.sh`
3. Configure cron:
   - Health (every 5 minutes):
     - `*/5 * * * * /home/$USER/grabakar-infra/scripts/vm_health_check.sh >> /home/$USER/grabakar-vm-health.log 2>&1`
   - Backup (daily 03:00):
     - `0 3 * * * /home/$USER/grabakar-infra/scripts/vm_backup_postgres.sh >> /home/$USER/grabakar-vm-backup.log 2>&1`
4. Ensure VM `gcloud auth` works for the backup script:
   - The backup script calls `gcloud storage cp/ls/rm`
   - It needs a service account that can write to:
     - `gs://grabakar-media-staging/backups/`

---

## 4. Monitoring (so staging is verifiable from GCP)
From `tecnico/DEVOPS_CICD_STRATEGY.md`, you still need to create:
- a Cloud Monitoring uptime check for:
  - `/api/v1/health/` on Cloud Run service `grabakar-backend`

This gives confidence that deployments succeeded even without manual inspection.

---

## 5. Close remaining functional issues (printing regression)
There was an open GitHub bug:
- `grabakar/grabakar-frontend#1` (`TC-MV-010`)

You must still re-test on emulator/device and confirm it’s resolved:
1. Run the same TC-MV-010 steps
2. Confirm the “Impresiones” counter increments correctly on each tap
3. Then close/reopen the issue on GitHub based on QA verification.

---

## Recommended next step (to make this operationally complete)
Answer these so I can produce the exact “runbook order” with zero backtracking:
1. Is staging already seeded with tenant/reference data? (`yes/no`)
2. Do you want to reset all seeded passwords during bootstrap? (`yes/no`)
3. Does the VM already have `gcloud` installed and authenticated? (`yes/no`)

