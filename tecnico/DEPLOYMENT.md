# Deployment e Infraestructura

## Desarrollo Local

Stack local via Docker Compose. Tres servicios: Django, PostgreSQL, Redis.

| Servicio | Puerto | Imagen |
|---|---|---|
| django | 8000 | Build local (Dockerfile) |
| postgres | 5432 | postgres:16-alpine |
| redis | 6379 | redis:7-alpine |

El frontend se ejecuta fuera de Docker (`npm run dev` en puerto 5173) para hot reload rápido. Comunica con Django via proxy de Vite.

## Docker Compose

```yaml
# docker-compose.yml
services:
  django:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    command: >
      sh -c "python manage.py migrate &&
             gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 2 --reload"
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  celery:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    command: celery -A config worker -l info
    volumes:
      - .:/app
    env_file: .env
    depends_on:
      - django
      - redis

  celery-beat:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    command: celery -A config beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    volumes:
      - .:/app
    env_file: .env
    depends_on:
      - django
      - redis

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME:-grabakar}
      POSTGRES_USER: ${DB_USER:-grabakar}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-grabakar_dev}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-grabakar}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

## Dockerfile (Django, multi-stage)

```dockerfile
# --- Builder stage ---
FROM python:3.12-slim AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends gcc libpq-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# --- Runtime stage ---
FROM python:3.12-slim AS runtime
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /install /usr/local
COPY . .
RUN python manage.py collectstatic --noinput 2>/dev/null || true
EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3"]
```

## Variables de Entorno

```bash
# .env.example — Copiar a .env y completar valores
# Django
DJANGO_SECRET_KEY=change-me-in-production
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DJANGO_SETTINGS_MODULE=config.settings.local

# Database
DB_NAME=grabakar
DB_USER=grabakar
DB_PASSWORD=grabakar_dev
DB_HOST=postgres
DB_PORT=5432

# Redis
REDIS_URL=redis://redis:6379/0

# Celery
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/1

# JWT
JWT_ACCESS_TOKEN_LIFETIME_HOURS=3
JWT_REFRESH_TOKEN_LIFETIME_DAYS=7
JWT_OFFLINE_TOKEN_LIFETIME_HOURS=72

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:5173

# File storage (django-storages)
DEFAULT_FILE_STORAGE=django.core.files.storage.FileSystemStorage
# Para GCP: DEFAULT_FILE_STORAGE=storages.backends.gcloud.GoogleCloudStorage
# GS_BUCKET_NAME=grabakar-media

# Email (reportes)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
# Producción: django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=
EMAIL_PORT=587
EMAIL_HOST_USER=
EMAIL_HOST_PASSWORD=
```

| Variable | Descripción | Requerida |
|---|---|---|
| `DJANGO_SECRET_KEY` | Clave secreta Django. Única por entorno. | Sí |
| `DJANGO_DEBUG` | `True` en local, `False` en staging/prod. | Sí |
| `DJANGO_ALLOWED_HOSTS` | Hosts permitidos, separados por coma. | Sí |
| `DJANGO_SETTINGS_MODULE` | Module path del settings file. | Sí |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` | Credenciales PostgreSQL. | Sí |
| `DB_HOST` / `DB_PORT` | Host y puerto de PostgreSQL. | Sí |
| `REDIS_URL` | URL de conexión a Redis. | Sí |
| `CELERY_BROKER_URL` | URL del broker Celery (Redis). | Sí |
| `JWT_ACCESS_TOKEN_LIFETIME_HOURS` | Duración del access token. Default: 3h. | No |
| `JWT_OFFLINE_TOKEN_LIFETIME_HOURS` | Duración del token offline. Default: 72h. | No |
| `CORS_ALLOWED_ORIGINS` | Origins permitidos para CORS. | Sí |
| `DEFAULT_FILE_STORAGE` | Backend de storage para archivos (django-storages). | Sí |

## GCP Deployment (target)

Diseño platform-agnostic: el backend no importa SDKs de GCP en código de negocio. Toda la adaptación es via env vars y django-storages.

| Componente | Servicio GCP | Alternativa genérica |
|---|---|---|
| Backend API | Cloud Run | Cualquier container runtime (ECS, AKS, VPS) |
| PostgreSQL | Cloud SQL for PostgreSQL | RDS, managed Postgres, self-hosted |
| Redis | Memorystore for Redis | ElastiCache, managed Redis, self-hosted |
| Static/media files | Cloud Storage | S3, Azure Blob, local filesystem |
| Secrets | Secret Manager | Vault, SSM Parameter Store, env files |
| CI/CD | GitHub Actions → Cloud Run deploy | GitHub Actions → cualquier target |
| Frontend hosting | Cloud Storage + CDN (Cloud CDN) | S3+CloudFront, Netlify, Vercel |

### Principios de diseño agnóstico

1. **django-storages** para abstracción de file storage. Cambiar de local a GCS o S3 requiere solo cambiar `DEFAULT_FILE_STORAGE` y credenciales.
2. **Sin imports de `google.cloud.*` en `api/`**. Si se necesita un SDK de GCP, aislarlo en un módulo de infraestructura.
3. **Toda configuración via env vars.** El mismo Docker image corre en local, staging y producción.
4. **DATABASE_URL pattern**: usar `dj-database-url` para parsear la URL de conexión desde una variable de entorno.

### Cloud Run deployment

```bash
# Build y push a Artifact Registry
gcloud builds submit --tag gcr.io/$PROJECT_ID/grabakar-backend

# Deploy a Cloud Run
gcloud run deploy grabakar-backend \
  --image gcr.io/$PROJECT_ID/grabakar-backend \
  --platform managed \
  --region southamerica-east1 \
  --set-env-vars "DJANGO_SETTINGS_MODULE=config.settings.production" \
  --set-secrets "DJANGO_SECRET_KEY=django-secret:latest,DB_PASSWORD=db-password:latest" \
  --allow-unauthenticated
```

## CI/CD — GitHub Actions

### Backend pipeline

```
PR abierto → ruff lint → pytest (80% coverage) → build Docker → [merge] → deploy staging
                                                                           → [manual] deploy prod
```

### Frontend pipeline

```
PR abierto → ESLint + tsc → Vitest (70% coverage) → build Vite → [merge] → deploy staging CDN
                                                                              → [manual] deploy prod
```

### Deploy workflow (backend)

```yaml
# .github/workflows/deploy-backend.yml
name: Deploy Backend
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        default: staging

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment || 'staging' }}
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud builds submit --tag gcr.io/${{ vars.GCP_PROJECT }}/grabakar-backend
      - run: |
          gcloud run deploy grabakar-backend \
            --image gcr.io/${{ vars.GCP_PROJECT }}/grabakar-backend \
            --region ${{ vars.GCP_REGION }} \
            --platform managed
```

## Staging vs Producción

| Aspecto | Staging | Producción |
|---|---|---|
| URL backend | `api-staging.grabakar.cl` | `api.grabakar.cl` |
| URL frontend | `staging.grabakar.cl` | `app.grabakar.cl` |
| Base de datos | Cloud SQL instancia separada | Cloud SQL instancia separada |
| Redis | Memorystore instancia separada | Memorystore instancia separada |
| `DJANGO_DEBUG` | `False` | `False` |
| Secrets | Secret Manager (proyecto staging) | Secret Manager (proyecto prod) |
| Docker image | Mismo build que producción | Mismo build que staging (promoted) |
| Deploy trigger | Auto en merge a `main` | Manual (workflow dispatch) |

El mismo Docker image se promueve de staging a producción. No se re-buildea. Esto garantiza paridad entre ambientes.
