# Arquitectura del Sistema

## Diagrama de Alto Nivel

```
┌────────────────────────────────────────────┐
│          DISPOSITIVO (Tablet/Móvil)        │
│                                            │
│  React 18+ PWA (TypeScript / Vite)         │
│  ├─ Context API (estado global)            │
│  ├─ Dexie.js → IndexedDB                  │
│  │   ├─ grabados (cola offline)            │
│  │   ├─ impresiones_vidrio                 │
│  │   └─ configuracion (cache)              │
│  ├─ Service Worker (cache, offline shell)  │
│  ├─ SyncManager (cola + retry)             │
│  ├─ AuthService (JWT + offline token)      │
│  ├─ PrintService (Bluetooth, fase 2+)      │
│  └─ ScannerService (OCR, fase 4+)          │
│                                            │
│  Conectividad: navigator.onLine + ping     │
│  ├─ Online  → sync automático en background│
│  └─ Offline → opera con datos locales      │
└──────────────────┬─────────────────────────┘
                   │ HTTPS REST JSON
                   │ (cuando hay red)
                   ▼
┌────────────────────────────────────────────┐
│           BACKEND (Django 5+)              │
│                                            │
│  Gunicorn + Nginx                          │
│  ├─ API REST (DRF)                         │
│  │   ├─ /api/v1/auth/                      │
│  │   ├─ /api/v1/sync/upload/               │
│  │   ├─ /api/v1/config/                    │
│  │   ├─ /api/v1/grabados/                  │
│  │   └─ /api/v1/reportes/                  │
│  ├─ Single app: api/                       │
│  └─ Multi-tenant queryset filtering        │
│                                            │
│  Celery Workers (Redis broker)             │
│  ├─ Procesamiento de sync batches          │
│  ├─ Generación de reportes CSV             │
│  └─ Purga periódica de datos               │
│                                            │
├────────────────┬───────────────────────────┤
│  PostgreSQL 16+│       Redis               │
│  ├─ Datos      │  ├─ Celery broker         │
│  ├─ Usuarios   │  ├─ Celery result backend │
│  ├─ Tenants    │  └─ Cache (opcional)       │
│  └─ JSONB cfg  │                           │
└────────────────┴───────────────────────────┘
```

## Diseño Offline-First

El dispositivo es la fuente de verdad para datos no sincronizados.

1. Todo registro se crea y persiste primero en IndexedDB local.
2. El backend no es requerido para la operación diaria (crear grabados, imprimir).
3. La sincronización ocurre en background cuando hay red.
4. Sin red, la app opera normalmente por días. Solo el login inicial requiere conexión.
5. Conflictos se resuelven a favor del dispositivo (`fecha_creacion_local` más reciente gana).

## Repositorios

| Repositorio | Contenido | Stack |
|---|---|---|
| `grabakar-frontend` | PWA React, exportable a Ionic | React 18+, TypeScript, Vite, Dexie.js, Capacitor |
| `grabakar-backend` | API REST, lógica de negocio, sync | Django 5+, DRF, PostgreSQL 16+, Celery, Redis |
| `grabakar-docs` | Documentación completa del proyecto | Markdown |
| `grabakar-infra` | IaC, Docker Compose prod, CI/CD | Terraform, Docker, scripts — solo si se justifica |

Los repositorios de frontend y backend son independientes para permitir trabajo paralelo con múltiples agentes/desarrolladores.

## Arquitectura Frontend

### Estado y datos

- **Context API** para estado global (auth, tenant config, connectivity status). Sin Redux ni Zustand — la complejidad no lo justifica.
- **Dexie.js** como wrapper sobre IndexedDB. Fuente de verdad local para grabados, impresiones y configuración cacheada.
- **react-hook-form** para formularios con validación en `onBlur`/`onSubmit`.

### Servicios modulares

```typescript
// Servicios como singletons, importados donde se necesiten
AuthService       // Login, refresh, offline token, session validation
SyncManager       // Cola de sync, batch upload, retry con backoff
PrintService      // Bluetooth printing (fase 2+), interfaz definida desde fase 1
ScannerService    // OCR cámara (fase 4+), interfaz definida desde fase 1
```

### PWA / Service Worker

- Precache del app shell y assets estáticos via Workbox (plugin de Vite).
- Runtime cache para requests de config (`/api/v1/config/`).
- El SW no intercepta requests de sync — eso lo maneja SyncManager directamente.

### Ruta a Ionic

Fase 1 es webapp PWA pura. En fase 5 se exporta a Ionic/Capacitor para acceso nativo (Bluetooth, cámara, Play Store). React se mantiene como motor de UI — Ionic solo agrega la capa nativa.

## Arquitectura Backend

### Monolito Django

Una sola app `api/` dentro del proyecto Django. No hay microservicios. La estructura interna:

```
grabakar-backend/
├── .github/workflows/ci.yml   # ruff + pytest
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── local.py
│   │   ├── production.py
│   │   └── test.py             # SQLite in-memory para tests
│   ├── celery.py
│   ├── urls.py
│   └── wsgi.py / asgi.py
├── api/
│   ├── models.py               # Tenant, Usuario, LeyCaso, Grabado, ImpresionVidrio
│   ├── serializers/            # auth.py, grabados.py
│   ├── views/                  # auth, config, grabados, health, reportes, sync
│   ├── services/               # SyncService, ReporteService
│   ├── tasks/                  # sync_tasks, reportes_tasks
│   ├── tests/                  # conftest + tests por dominio
│   ├── utils/                  # exceptions.py (custom handler español)
│   ├── permissions.py          # TenantQuerySetMixin
│   └── urls.py
├── manage.py
├── Dockerfile
├── requirements.txt
├── pytest.ini
└── docker-compose.yml
```

### Celery + Redis

- Redis como message broker y result backend.
- Tasks: procesamiento de sync batches, generación de reportes CSV, purga de datos antiguos.
- Celery beat para tareas periódicas (reporte diario a las 23:59 CLT, purga semanal).

## Comunicación

- **Protocolo**: HTTPS REST JSON exclusivamente.
- **Sin WebSockets**: la naturaleza offline-first hace que WebSockets sean poco fiables (el dispositivo puede estar sin red por días). Toda comunicación es request-response iniciada por el dispositivo.
- **Versionamiento**: `/api/v1/` como prefijo. Versión nueva cuando hay breaking changes.

## Multi-Tenant

- Modelo `Tenant` con branding (logo, colores) y `configuracion_json` (JSONB) para overrides por cliente.
- Todas las queries filtradas por `tenant_id` del usuario autenticado. Un mixin o manager custom garantiza esto:

```python
class TenantQuerySetMixin:
    def get_queryset(self):
        return super().get_queryset().filter(tenant=self.request.user.tenant)
```

- El branding se entrega al frontend via `GET /api/v1/config/` y se cachea en IndexedDB.
- CSS variables en frontend para aplicar colores dinámicamente.
- Cero valores hardcodeados: "GrabaKar" es el nombre default del tenant, no una constante.

## Log de Decisiones Arquitectónicas

| Decisión | Elegido | Alternativa | Razón |
|---|---|---|---|
| Framework frontend | React 18+ | Flutter | Web-first con ruta directa a PWA. Flutter web tiene peor SEO y bundle size. React → Ionic es un path probado para exportar a nativo. |
| Framework backend | Django 5+ | Node.js/Express | Familiaridad del equipo con Python. Admin panel gratis. ORM maduro con migrations. DRF es estándar para APIs REST. |
| Base de datos | PostgreSQL 16+ | MySQL, SQLite | JSONB para config flexible por tenant. Fiabilidad probada. Soporte nativo en GCP Cloud SQL. Full-text search si se necesita. |
| Storage local | Dexie.js | localForage, raw IndexedDB | Mejor wrapper de IndexedDB: API fluida, reactive queries (`liveQuery`), transactions con rollback, compound indexes. |
| Estado global | Context API | Redux, Zustand | La app tiene poco estado compartido (auth, config, connectivity). Context es suficiente. No se justifica una librería externa. |
| Async tasks | Celery + Redis | Django Q, Huey | Celery es el estándar en Django. Redis cumple doble rol (broker + cache). Mayor ecosistema y documentación. |
| Comunicación | REST JSON | GraphQL, WebSockets | REST es simple y predecible para offline-first. GraphQL agrega complejidad innecesaria para esta cantidad de endpoints. WebSockets no funcionan sin red. |
| Deploy target | GCP | AWS, Azure | Stack nativo: Cloud Run (Django API), Cloud SQL (PostgreSQL), Memorystore (Redis), Cloud Scheduler (CRON), Cloud Storage (Media). GitHub Actions automatiza el despliegue a GCP. |
