# Agente paralelo: CI Backend (P0-04)

Asignación para añadir el pipeline de CI del backend. No modificar lógica de negocio ni endpoints.

## Contexto

- **Repo**: `grabakar-backend/` (raíz del backend)
- **Referencia**: `grabakar-docs/tecnico/TESTING.md` § CI — GitHub Actions (backend.yml)
- **Plan**: `BACKEND_IMPLEMENTATION.md` § CI/CD (P0-04)

## Estado actual

- No existe `.github/workflows/` en el backend. Si el backend vive en un repo separado, crear `.github/workflows/ci.yml` en la raíz de ese repo.

## Tarea única

### Crear `.github/workflows/ci.yml`

Contenido base (adaptar paths si el backend está en un monorepo):

```yaml
name: Backend CI
on:
  pull_request:
    branches: [main]
    paths:
      - '**'
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: grabakar_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Lint
        run: ruff check .
      - name: Tests
        env:
          DJANGO_SETTINGS_MODULE: config.settings.test
          DB_ENGINE: django.db.backends.postgresql
          DB_NAME: grabakar_test
          DB_USER: postgres
          DB_PASSWORD: postgres
          DB_HOST: localhost
          DB_PORT: 5432
          SECRET_KEY: test-secret
          CELERY_BROKER_URL: redis://localhost:6379/0
        run: pytest --cov=api --cov-fail-under=80 -q
```

**Notas**:

- Si `config.settings.test` usa SQLite en memoria, los tests pueden no necesitar Postgres en CI; en ese caso se puede quitar el servicio `postgres` y usar solo SQLite para velocidad. Si se quiere paridad con producción, dejar Postgres y asegurar que `config/settings/test.py` en CI use las env vars de base de datos.
- Ajustar `paths` si el backend está en un subdirectorio (ej. `grabakar-backend/**`).
- `pytest` debe ejecutarse desde la raíz del backend donde está `manage.py`.

## Verificación

- Push a una branch y abrir PR; el workflow debe ejecutarse.
- Lint (ruff) y tests deben pasar.

## Branch y commits

- Branch: `feature/ci-backend`
- Commit: `ci(backend): add GitHub Actions workflow for lint and pytest`
