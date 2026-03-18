# Backlog de Implementación

**Convención de IDs:** `P<fase>-<secuencia>` (ej: P0-01). Dependencias se expresan como lista de IDs que deben estar completos antes de iniciar la tarea.

**Estados:** `pendiente` | `en progreso` | `completo` | `bloqueado`

---

## Fase 0: Infraestructura y Setup

### P0-01: Crear esqueleto del repositorio backend
- **Repo:** `grabakar-backend`
- **Descripción:** Proyecto Django con estructura base, Docker Compose (Django + PostgreSQL 16 + Redis), Dockerfile, `requirements.txt`, settings split (base/local/production), app `api` vacía.
- **Entregable:** `docker compose up` levanta Django + PostgreSQL + Redis. El endpoint `/api/health/` responde 200.
- **Dependencias:** Ninguna
- **Archivos relevantes:** `tecnico/DEPLOYMENT.md`, `prompts/BACKEND_AGENT.md`
- **Criterio de aceptación:**
  - `docker compose up -d` funciona sin errores
  - Django admin accesible en `localhost:8000/admin/`
  - PostgreSQL accesible en `localhost:5432`
  - `.cursorrules` creado en la raíz del repo
  - `.env.example` con todas las variables necesarias
  - `pytest` ejecuta sin errores (0 tests por ahora)
- **Estado:** completo

### P0-02: Crear esqueleto del repositorio frontend
- **Repo:** `grabakar-frontend`
- **Descripción:** Proyecto React 18 + TypeScript + Vite. Configurar ESLint, Prettier, Vitest, estructura de carpetas (`src/pages/`, `src/components/`, `src/services/`, `src/utils/`, `src/hooks/`, `src/types/`). React Router configurado con rutas placeholder.
- **Entregable:** `npm run dev` levanta la app. `npm test` ejecuta Vitest. Rutas `/login`, `/home`, `/grabado/nuevo` muestran placeholders.
- **Dependencias:** Ninguna
- **Archivos relevantes:** `tecnico/ARQUITECTURA.md`, `prompts/FRONTEND_AGENT.md`
- **Criterio de aceptación:**
  - `npm run dev` funciona
  - `npm run lint` pasa sin errores
  - `npm test` ejecuta sin errores (0 tests por ahora)
  - `.cursorrules` creado en la raíz del repo
  - Service worker básico registrado (PWA shell)
  - Theme provider configurado con CSS variables para white-labeling
- **Estado:** completo

### P0-03: Crear repositorio de documentación

- **Repo:** `grabakar-docs`
- **Descripción:** Mover todo el contenido de `grabakar-docs/` del workspace actual al repositorio dedicado. Agregar `README.md` como índice de navegación.
- **Entregable:** Repo con toda la documentación organizada y README funcional.
- **Dependencias:** Ninguna (los docs ya existen en el workspace)
- **Criterio de aceptación:**
  - Todas las carpetas y archivos del workspace `grabakar-docs/` presentes
  - `README.md` con links a cada sección
- **Estado:** completo

### P0-04: Configurar CI/CD base para backend
- **Repo:** `grabakar-backend`
- **Descripción:** GitHub Actions workflow: lint (ruff), tests (pytest), build Docker image. Se ejecuta en cada PR y push a `main`.
- **Entregable:** `.github/workflows/ci.yml` funcional.
- **Dependencias:** P0-01
- **Criterio de aceptación:**
  - Workflow ejecuta lint + tests en cada PR
  - PR no se puede mergear si CI falla
- **Estado:** completo

### P0-05: Configurar CI/CD base para frontend
- **Repo:** `grabakar-frontend`
- **Descripción:** GitHub Actions workflow: lint (ESLint), tests (Vitest), build. Se ejecuta en cada PR y push a `main`.
- **Entregable:** `.github/workflows/ci.yml` funcional.
- **Dependencias:** P0-02
- **Criterio de aceptación:**
  - Workflow ejecuta lint + tests + build en cada PR
  - PR no se puede mergear si CI falla
- **Estado:** pendiente

---

## Fase 1: MVP Core

### Bloque Backend - Modelos y Auth

#### P1-01: Modelos de datos base
- **Repo:** `grabakar-backend`
- **Descripción:** Crear modelos Django: `Tenant`, `Usuario` (extiende AbstractUser), `LeyCaso`, `Grabado`, `ImpresionVidrio`. Migraciones. Admin registrado para todos los modelos. Fixtures con datos iniciales (1 tenant, 1 ley, 1 usuario admin).
- **Dependencias:** P0-01
- **Archivos relevantes:** `tecnico/MODELO_DATOS.md`
- **Criterio de aceptación:**
  - `python manage.py migrate` ejecuta sin errores
  - Todos los modelos visibles y editables en Django admin
  - Fixtures cargables con `python manage.py loaddata initial`
  - Tests: modelo Grabado requiere patente >= 6 chars, tipo_movimiento válido, etc.
- **Estado:** completo

#### P1-02: Autenticación (JWT + token offline)
- **Repo:** `grabakar-backend`
- **Descripción:** Endpoints de auth usando `djangorestframework-simplejwt`. Login retorna access token, refresh token, y un offline_token (JWT de larga vida, 72h, con claims limitados). Endpoint de refresh. El login también retorna la configuración del tenant (branding) y leyes activas.
- **Dependencias:** P1-01
- **Archivos relevantes:** `tecnico/API_CONTRACTS.md` (sección Auth), `features/LOGIN_SESION.md`
- **Criterio de aceptación:**
  - `POST /api/auth/login/` retorna tokens + config
  - `POST /api/auth/refresh/` retorna nuevo access token
  - Token offline tiene expiración de 72h
  - Login con credenciales inválidas retorna 401 con mensaje en español
  - Tests: login exitoso, login fallido, refresh, token expirado
- **Estado:** completo

#### P1-03: API CRUD de Grabados
- **Repo:** `grabakar-backend`
- **Descripción:** Endpoints REST para Grabado: crear (con vidrios nested), listar (paginado, filtrable por operador/fecha/tipo), detalle. Permisos: operador ve solo los suyos; supervisor ve todos los del tenant.
- **Dependencias:** P1-01, P1-02
- **Archivos relevantes:** `tecnico/API_CONTRACTS.md` (sección Grabados)
- **Criterio de aceptación:**
  - `POST /api/grabados/` crea grabado con vidrios
  - `GET /api/grabados/` lista paginada con filtros
  - `GET /api/grabados/{uuid}/` detalle con vidrios
  - Permisos por rol verificados
  - Tests: CRUD completo, permisos, validaciones
- **Estado:** completo

#### P1-04: API de sincronización batch
- **Repo:** `grabakar-backend`
- **Descripción:** Endpoint `POST /api/sync/upload/` que recibe un batch de grabados desde un dispositivo. Procesa cada registro: crea si no existe, actualiza si UUID ya existe y fecha_creacion_local es más reciente. Setea `fecha_sincronizacion` al timestamp del servidor. Retorna resumen (exitosos, fallidos, errores por UUID).
- **Dependencias:** P1-01, P1-02
- **Archivos relevantes:** `tecnico/API_CONTRACTS.md` (sección Sync), `tecnico/SYNC_STRATEGY.md`
- **Criterio de aceptación:**
  - Batch de 10+ registros se procesa correctamente
  - Registros duplicados (mismo UUID) se manejan sin error
  - Response incluye conteo y detalle de errores
  - Tests: batch válido, batch con duplicados, batch con datos inválidos
- **Estado:** completo

#### P1-05: API de configuración
- **Repo:** `grabakar-backend`
- **Descripción:** Endpoint `GET /api/config/` que retorna la configuración del tenant del usuario autenticado: branding (nombre, logo, colores), leyes activas, lista de vidrios por tipo de vehículo.
- **Dependencias:** P1-01, P1-02
- **Archivos relevantes:** `tecnico/API_CONTRACTS.md` (sección Config)
- **Criterio de aceptación:**
  - Retorna branding del tenant del usuario
  - Retorna leyes activas
  - Retorna configuración de vidrios
  - Tests: respuesta correcta, usuario sin tenant (error)
- **Estado:** completo

### Bloque Frontend - Infraestructura

#### P1-06: Capa de almacenamiento local (IndexedDB)
- **Repo:** `grabakar-frontend`
- **Descripción:** Implementar capa de persistencia con Dexie.js. Tablas: `grabados`, `impresiones_vidrio`, `sync_queue`, `config_cache`, `auth_cache`. CRUD completo para cada tabla. Lógica de purga de registros >30 días ya sincronizados.
- **Dependencias:** P0-02
- **Archivos relevantes:** `features/SINCRONIZACION_OFFLINE.md`, `tecnico/SYNC_STRATEGY.md`
- **Criterio de aceptación:**
  - CRUD funcional para todas las tablas
  - Purga automática ejecuta correctamente (test con datos mock)
  - Registros no sincronizados nunca se purgan
  - Tests unitarios para cada operación
- **Estado:** completo

#### P1-07: Servicio de autenticación frontend
- **Repo:** `grabakar-frontend`
- **Descripción:** `AuthService`: login (online), almacenar tokens en IndexedDB, refresh automático, validación de token offline, logout. `AuthContext` con React Context para estado global de auth. `ProtectedRoute` wrapper. Interceptor de axios/fetch para agregar token a requests.
- **Dependencias:** P0-02, P1-06
- **Archivos relevantes:** `features/LOGIN_SESION.md`, `tecnico/API_CONTRACTS.md` (sección Auth)
- **Criterio de aceptación:**
  - Login guarda tokens en IndexedDB
  - Token expirado trigger refresh automático
  - Sin conexión + token offline válido = acceso permitido
  - Sin conexión + token offline expirado = redirige a login con mensaje
  - `ProtectedRoute` redirige a `/login` si no autenticado
  - Tests: flujo completo con mocks
- **Estado:** completo

#### P1-08: SyncManager
- **Repo:** `grabakar-frontend`
- **Descripción:** Servicio que gestiona la cola de sincronización. Detecta conectividad (`navigator.onLine` + ping periódico). Cuando online, envía batch al endpoint `/api/sync/upload/`. Reintenta con backoff exponencial en caso de error. Expone estado reactivo (pendientes, sincronizando, error).
- **Dependencias:** P1-06, P1-07
- **Archivos relevantes:** `features/SINCRONIZACION_OFFLINE.md`, `tecnico/SYNC_STRATEGY.md`
- **Criterio de aceptación:**
  - Detecta cambio online/offline
  - Envía batch cuando detecta conectividad
  - Reintenta con backoff en caso de error
  - Estado reactivo disponible via hook (`useSyncStatus`)
  - Tests: mock de red, verificar reintentos
- **Estado:** completo

### Bloque Frontend - Páginas

#### P1-09: Página de Login
- **Repo:** `grabakar-frontend`
- **Descripción:** UI de login: campos usuario y contraseña, botón de ingreso, logo/branding del tenant (desde config cacheada o default), mensajes de error en español, indicador de estado de conexión. Responsive para tablet y móvil.
- **Dependencias:** P1-07
- **Archivos relevantes:** `features/LOGIN_SESION.md`
- **Criterio de aceptación:**
  - Login exitoso redirige a `/home`
  - Error de credenciales muestra mensaje claro en español
  - Sin conexión muestra estado y usa token offline si disponible
  - UI responsive (tablet landscape como target primario)
  - Tests de componente: render, submit, errores
- **Estado:** completo

#### P1-10: Página Home (Lista de Grabados)
- **Repo:** `grabakar-frontend`
- **Descripción:** Lista de grabados del dispositivo (últimos 30 días). Cada item muestra: patente, fecha, tipo_movimiento, estado_sync (ícono). Botón "+" flotante para crear nuevo. Header con indicador de conexión y estado de sync. Menú hamburguesa con: config impresión, tipo vehículo, cola sync, cerrar sesión.
- **Dependencias:** P1-06, P1-07, P1-09
- **Archivos relevantes:** `features/GRABADO_PATENTE.md`
- **Criterio de aceptación:**
  - Lista carga desde IndexedDB
  - Botón "+" navega a `/grabado/nuevo`
  - Menú hamburguesa funcional
  - Indicadores visuales de sync correctos
  - Tests de componente
- **Estado:** completo

#### P1-11: Formulario de Creación de Grabado
- **Repo:** `grabakar-frontend`
- **Descripción:** Formulario con todos los campos de Grabado. Validaciones: patente >= 6 chars, confirmación coincide, campos obligatorios. Alerta de duplicado si patente ya existe en IndexedDB. Al guardar: persiste en IndexedDB, encola para sync, navega a flujo multi-vidrio. El botón cambia de "Guardar" a "Imprimir" post-guardado; campos patente y VIN se bloquean post-guardado.
- **Dependencias:** P1-06, P1-07, P1-10
- **Archivos relevantes:** `features/GRABADO_PATENTE.md`
- **Criterio de aceptación:**
  - Todas las validaciones funcionan
  - Duplicado detectado y alertado
  - Post-guardado: campos bloqueados, botón cambia
  - Registro persiste en IndexedDB
  - Registro se encola para sync
  - Tests: validaciones, duplicado, guardado
- **Estado:** completo

#### P1-12: Flujo Multi-Vidrio
- **Repo:** `grabakar-frontend`
- **Descripción:** Después de guardar un grabado, la UI itera por cada vidrio a grabar. Pantalla por vidrio: patente en tipografía grande (Poka-Yoke), nombre del vidrio, progreso ("Vidrio 3 de 6"), botón "Imprimir" (incrementa contador en IndexedDB), botón "Siguiente"/"Finalizar". La lista de vidrios viene de la configuración del tenant/tipo de vehículo.
- **Dependencias:** P1-06, P1-11
- **Archivos relevantes:** `features/FLUJO_MULTI_VIDRIO.md`
- **Criterio de aceptación:**
  - Itera por todos los vidrios configurados
  - Patente visible en tipografía grande (>48px mínimo)
  - Contador de impresiones incrementa por vidrio
  - "Finalizar" retorna a Home
  - Tests de componente: flujo completo
- **Estado:** completo

#### P1-13: Theming y White-Label
- **Repo:** `grabakar-frontend`
- **Descripción:** Theme provider que inyecta CSS variables desde configuración del tenant (cacheada en IndexedDB). Variables: `--color-primary`, `--color-secondary`, `--logo-url`, `--app-name`. Default: colores GrabaKar. Todos los componentes usan las variables, cero valores hardcodeados.
- **Dependencias:** P0-02, P1-06
- **Archivos relevantes:** `features/WHITE_LABELING.md`
- **Criterio de aceptación:**
  - Cambiar config de tenant cambia colores y logo en toda la app
  - Sin config: defaults de GrabaKar
  - Cero referencias hardcodeadas a "GrabaKar" en componentes
  - Test: verificar que CSS variables se aplican
- **Estado:** completo

---

## Fase 2: Sync Automático + Reportes XLSX

### Bloque Backend - Modelo y Reportes

#### P2-01: Modelo Sucursal + campos nuevos + migración
- **Repo:** `grabakar-backend`
- **Descripción:** Crear modelo `Sucursal` (tenant FK, nombre, activa). Agregar `Tenant.tipo_cliente`, `Usuario.sucursal` FK, `Grabado.sucursal` FK, `Grabado.responsable_texto`. Generar y aplicar migración. Actualizar admin, serializers, SyncService.
- **Dependencias:** P1-01
- **Archivos relevantes:** `tecnico/MODELO_DATOS.md`
- **Criterio de aceptación:**
  - Migración aplica sin errores
  - Sucursal visible en admin con filtros por tenant
  - Sync upload acepta `responsable_texto` y `sucursal_id`
  - Tests: modelo, admin, sync con nuevos campos
- **Estado:** completo

#### P2-02: ReporteXLSXService + endpoint plataforma
- **Repo:** `grabakar-backend`
- **Descripción:** Crear `api/services/reporte_xlsx_service.py` con `ReporteXLSXService.generar_xlsx(report_date, tenant_id=None)`. Genera XLSX con 3 tabs (Resumen, Resumen por dia, Patentes). Agregar `openpyxl` a requirements. Crear vista `ReportePlataformaView` en `api/views/reportes.py` y registrar en `api/urls.py`. Admin-only.
- **Dependencias:** P2-01
- **Archivos relevantes:** Ninguno (ya implementado)
- **Criterio de aceptación:**
  - `GET /api/v1/reportes/plataforma/?fecha=2026-03-03` devuelve XLSX descargable
  - XLSX tiene 3 tabs con datos correctos
  - Tab 2 agrupa por `fecha_sincronizacion` (verificado)
  - Solo admin puede acceder (403 para operador/supervisor)
  - Tests: servicio unitarios + endpoint integración
- **Estado:** completo

#### P2-03: Celery task XLSX diaria
- **Repo:** `grabakar-backend`
- **Descripción:** Task `generar_reporte_xlsx_diario()` que genera XLSX month-to-date diariamente. Configurar en celery-beat (01:00 AM America/Santiago).
- **Dependencias:** P2-02
- **Archivos relevantes:** Ninguno (ya implementado)
- **Criterio de aceptación:**
  - Task ejecuta y genera XLSX correcto
  - Configurada en celery-beat scheduler
- **Estado:** completo

#### P2-04: Seed data sucursales
- **Repo:** `grabakar-backend`
- **Descripción:** Actualizar fixtures con sucursales de ejemplo. Opcionalmente, script de backfill para asignar sucursales a grabados existentes basado en datos conocidos.
- **Dependencias:** P2-01
- **Criterio de aceptación:**
  - `loaddata initial` carga sucursales de ejemplo
- **Estado:** completo

---

## Grafo de Dependencias

```
P0-01 (BE skeleton) ──┬── P0-04 (BE CI/CD)
                       ├── P1-01 (Modelos) ──┬── P1-02 (Auth BE) ──┬── P1-03 (CRUD Grabados)
                       │                     │                     ├── P1-04 (Sync batch)
                       │                     │                     └── P1-05 (Config API)
                       │                     └────────────────────────────┘

P0-02 (FE skeleton) ──┬── P0-05 (FE CI/CD)
                       ├── P1-06 (IndexedDB) ──┬── P1-07 (Auth FE) ──┬── P1-08 (SyncManager)
                       │                       │                     └── P1-09 (Login page)
                       │                       ├── P1-10 (Home) ──── P1-11 (Form Grabado) ── P1-12 (Multi-Vidrio)
                       │                       └── P1-13 (Theming)
                       └── P1-13 (Theming)

P0-03 (Docs repo) ── sin dependientes técnicos
```

### Tareas Paralelizables (Max Concurrencia)

**Ronda 1** (0 dependencias): P0-01, P0-02, P0-03 → 3 agentes simultáneos
**Ronda 2** (post P0-01 y P0-02): P0-04, P0-05, P1-01, P1-06, P1-13 → 5 tareas, 3-4 agentes recomendados
**Ronda 3** (post P1-01): P1-02, P1-07 → 2 agentes
**Ronda 4** (post P1-02, P1-07): P1-03, P1-04, P1-05, P1-08, P1-09 → 5 tareas, 3-4 agentes
**Ronda 5** (post P1-09): P1-10, P1-11 → 2 agentes
**Ronda 6** (post P1-11): P1-12 → 1 agente

---

## Fase 5: DevOps & CI/CD

### P5-01: Pipeline CI/CD Automático
- **Repo:** `grabakar-infra` (orquestrador)
- **Descripción:** Implementar estrategia DevOps robusta con GitHub Actions. Configurar pipelines para compilar y desplegar automáticamente el backend y ambos frontends a GCP cuando hay cambios en las ramas principales.
- **Entregable:** Workflows en GitHub que detecten cambios en los submodulos o repositorios individuales y desplieguen automáticamente a staging/production sin intervención manual rutinaria.
- **Dependencias:** Repositorios separados completos (Fase 0)
- **Archivos relevantes:** `.github/workflows/deploy.yml`
- **Criterio de aceptación:**
  - Un commit en `grabakar-backend` dispara actualización en Cloud Run
  - Un commit en los frontends regenera el build `.js/.css` y actualiza los buckets en Google Cloud Storage automáticamente.
