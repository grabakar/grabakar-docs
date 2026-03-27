# GCP Staging Deployment — Reference

Deployment notes for the `grabakar-staging` project (project number `1089044937741`).

## Authentication: Workload Identity Federation

The GCP Organization has `iam.disableServiceAccountKeyCreation` enforced, so JSON keys **cannot** be created. All GitHub Actions authenticate via OIDC tokens using Workload Identity Federation.

### Resources

| Resource | Value |
|---|---|
| Service Account | `github-actions@grabakar-staging.iam.gserviceaccount.com` |
| WIF Pool | `github-actions-pool` (global) |
| WIF Provider | `github-actions-provider-2` |
| Full Provider URI | `projects/1089044937741/locations/global/workloadIdentityPools/github-actions-pool/providers/github-actions-provider-2` |
| Attribute Condition | `assertion.repository_owner == 'grabakar'` |

### IAM Roles

**Service Account `github-actions`:**

| Role | Purpose |
|---|---|
| `roles/run.admin` | Deploy to Cloud Run |
| `roles/storage.admin` | Upload to GCS buckets |
| `roles/iam.serviceAccountUser` | Act as SA in Cloud Run |
| `roles/artifactregistry.admin` | Push Docker images |
| `roles/compute.instanceAdmin.v1` | SSH to VM for migrations/Celery |

**Compute Engine default SA (`1089044937741-compute@developer.gserviceaccount.com`):**

| Role | Purpose |
|---|---|
| `roles/artifactregistry.reader` | Pull Docker images on the VM |

## Repositories & Workflows

| Repo | Workflow | What it deploys |
|---|---|---|
| `grabakar-backend` | `deploy.yml` | Docker → Artifact Registry → Cloud Run + VM migrations + Celery restart |
| `grabakar-frontend` | `deploy.yml` | `npm run build` → `gs://grabakar-frontend-staging` |
| `grabakar-admin` | `deploy.yml` | `npm run build` → `gs://grabakar-admin-staging` |
| `grabakar-backend` | `bootstrap-staging.yml` | Manual: seeds staging DB with test users |
| `grabakar-frontend` | `build-apks.yml` | Manual: builds and uploads `apk-local` + `apk-gcp` artifacts |

All deployment workflows trigger on `push` to `main` and support manual `workflow_dispatch`.

## Known Gotchas

### 1. Docker auth inside VM SSH sessions
When GitHub Actions SSHes into the VM to run migrations or restart Celery, the `docker pull` command fails unless Docker is explicitly authenticated against Artifact Registry **within that SSH session**:
```bash
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
```
This is already included in the workflow YAML.

### 2. WIF Provider creation requires attribute conditions
Google's "Secure by Default" policies require an `--attribute-condition` when creating OIDC providers. Without it, `create-oidc` returns `INVALID_ARGUMENT`. The condition we use:
```
assertion.repository_owner == 'grabakar'
```

### 3. GCS bucket error page flag
The correct flag for the 404 error page in `gcloud storage buckets update` is:
```
--web-error-page=index.html     ✅ correct
--web-not-found-page=index.html ❌ does not exist
```

### 4. WIF propagation delay
After first creating a WIF pool/provider, expect up to 10 minutes of `invalid_target` errors before the OIDC tokens propagate globally.

### 5. Android build requires Java 21
`grabakar-frontend` APK build fails with `invalid source release: 21` when the environment uses Java 17 or lower.

Use Java 21 in local builds and CI runners:

```bash
export JAVA_HOME="/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
```

### 6. APK backend target must be validated explicitly
An APK can compile successfully but still point to the wrong backend (`10.0.2.2` vs Cloud Run).

Always validate `dist/assets/index-*.js` after build:

- GCP variant must include `grabakar-backend-1089044937741.us-central1.run.app`
- GCP variant must not include `10.0.2.2:8000`

## Setup Script

The file `setup_wif.sh` (at the repo root) automates:
1. Creating the `github-actions` service account
2. Granting all required IAM roles
3. Creating the WIF pool and OIDC provider
4. Binding all three GitHub repositories

Run it after `gcloud auth login` with an Organization Admin account.
