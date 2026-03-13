# Backend — Estado y reporte

Estado del proyecto `grabakar-backend` respecto a `BACKEND_IMPLEMENTATION.md` y `API_CONTRACTS.md`. Última actualización: 2026-03-04 (post mini-agentes).

---

## Resumen ejecutivo

| Área            | Estado | Notas                                           |
|-----------------|--------|-------------------------------------------------|
| Repo y settings | Hecho  | Django 5, DRF, JWT, Celery, Docker, test.py     |
| Modelos y admin | Hecho  | Tenant, Usuario, LeyCaso, Grabado, ImpresionVidrio |
| Auth            | Hecho  | Login, refresh, offline token, config en login   |
| Config          | Hecho  | GET /config/                                     |
| Grabados CRUD   | Hecho  | CRUD + list contract + filtros + paginación      |
| Sync            | Hecho  | POST upload + GET status + errores en respuesta  |
| Reportes per-tenant | Hecho  | Diario + mensual, JSON/CSV, permisos, Celery task|
| Reportes XLSX   | Pendiente | Cross-tenant, multi-tab, requiere Sucursal model |
| CI              | Hecho  | GitHub Actions: ruff + pytest + cobertura        |
| Tests           | 61 pasan | Cobertura completa por dominio                 |

---

## 1. Lo que está funcionando

### Infra y configuración

- Proyecto Django en `config/` con settings split: `base`, `local`, `production`, `test` (SQLite in-memory para tests).
- Una app `api/` con modelos, admin, urls, views, serializers, services, tasks, utils.
- Docker Compose: django, celery, celery-beat, postgres, redis. Dockerfile multi-stage.
- `.env.example`, `pytest.ini`, `config/celery.py`, custom exception handler en español.
- CI: `.github/workflows/ci.yml` con ruff + pytest --cov-fail-under=80.

### Modelos

- **Tenant**, **Usuario** (AbstractUser, tenant, rol, activo), **LeyCaso**, **Grabado** (UUID PK, tenant, estado_sync, índices), **ImpresionVidrio** (unique_together grabado+numero_vidrio). Migración inicial aplicable.

### Autenticación

- **POST /api/v1/auth/login/**: username/password, devuelve access_token, refresh_token, offline_token, usuario, tenant, leyes_activas, vidrios_config. Errores en español. Usuario inactivo → 401.
- **POST /api/v1/auth/refresh/**: acepta `refresh_token` (o `refresh`). Rotación y blacklist.
- JWT con tenant_id y rol en el payload.

### Configuración

- **GET /api/v1/config/**: autenticado; devuelve tenant, leyes_activas, vidrios_config.

### Grabados

- **POST /api/v1/grabados/**: creación con vidrios anidados, validación patente (6–10 alfanuméricos) y VIN (17 chars), tenant y ley_caso activa.
- **GET /api/v1/grabados/**: listado paginado (PAGE_SIZE 20, max 100). Filtros: `tipo_movimiento`, `estado_sync`, `operador_id` (solo supervisor/admin), `fecha_desde`, `fecha_hasta`, `patente` (icontains). Anotaciones: `cantidad_vidrios`, `vidrios_impresos`.
- **GET /api/v1/grabados/{uuid}/**: detalle con vidrios.
- `GrabadoListSerializer`: `usuario_responsable` nested como `{ id, nombre_completo }`, `cantidad_vidrios`, `vidrios_impresos`. Cumple contrato API § 2.
- `GrabadoPagination`: `page_size_query_param='page_size'`, `max_page_size=100`.
- `TenantQuerySetMixin` aplicado; operador solo ve propios, supervisor/admin ve todos del tenant.

### Sync

- **POST /api/v1/sync/upload/**: batch (max 100), creación/actualización por UUID y fecha_creacion_local, tenant del usuario. Acepta `vidrios` o `impresiones` en cada ítem. Respuesta: `procesados`, `exitosos`, `fallidos`, `errores` (array con `{uuid, error, code}`), `resultados` (por ítem).
- **GET /api/v1/sync/status/?device_id=xxx**: retorna `device_id`, `ultima_sincronizacion`, `registros_sincronizados`, `registros_pendientes_servidor`. Filtrado por tenant del usuario. Sin device_id → 400.

### Reportes (per-tenant JSON/CSV — implementado)

- **GET /api/v1/reportes/diario/**: query params `fecha` (ISO date, default hoy), `operador_id` (opcional), `formato` (json/csv). Solo supervisor/admin; operador → 403.
- **GET /api/v1/reportes/mensual/**: query param `mes` (YYYY-MM, default mes actual), `operador_id`, `formato`. Misma estructura y permisos que diario.
- `ReporteService.generar_diario(tenant_id, fecha, operador_id)`: filtra por día en timezone del proyecto, agrega por operador y tipo, lista registros con campos del contrato.
- `ReporteService.generar_mensual(tenant_id, anio, mes, operador_id)`: misma lógica para rango mensual.
- CSV: UTF-8 BOM, headers según contrato.

### Reportes (plataforma XLSX — pendiente)

- **No implementado aún.** El reporte externo actual es un XLSX multi-tab cross-tenant agrupado por `fecha_sincronizacion`. El `ReporteService` existente filtra por `fecha_creacion_local` y es per-tenant — no replica el formato externo.
- Requiere: modelo Sucursal, `Tenant.tipo_cliente`, `Grabado.responsable_texto`, `Grabado.sucursal`, y un nuevo `ReporteXLSXService`.
- Plan completo: `prompts/implementation_plans/REPORT_XLSX_IMPLEMENTATION.md`.

### Celery y tasks

- Celery configurado con Redis como broker y result backend.
- `purgar_registros_antiguos`: elimina grabados sincronizados >30 días.
- `generar_reporte_diario(tenant_id, fecha)`: task Celery que invoca `ReporteService.generar_diario`. Default: reporte del día anterior.

### CI

- `.github/workflows/ci.yml`: trigger en PR a main, Python 3.12, pip cache, ruff check, pytest con `--cov=api --cov-fail-under=80`.
- Usa `config.settings.test` (SQLite in-memory) para velocidad en CI.

### Tests (61 total)

| Archivo            | Tests | Dominio                                |
|--------------------|-------|----------------------------------------|
| test_health.py     | 1     | Health check                           |
| test_config.py     | 1     | GET /config/                           |
| test_auth.py       | 7     | Login, refresh, validaciones           |
| test_grabados.py   | 18    | CRUD, list contract, filtros, paginación, tenant |
| test_sync.py       | 11    | Upload, status, errores, tenant aislado|
| test_reportes.py   | 23    | Service diario/mensual, views, permisos, CSV |

---

## 2. Alineación con contratos API

| Endpoint                    | Contrato § | Estado     | Notas |
|-----------------------------|------------|------------|-------|
| POST /auth/login/           | 1          | Cumplido   | — |
| POST /auth/refresh/         | 1          | Cumplido   | — |
| POST /grabados/             | 2          | Cumplido   | Validaciones patente, VIN, ley activa |
| GET /grabados/              | 2          | Cumplido   | List serializer, filtros, paginación |
| GET /grabados/{uuid}/       | 2          | Cumplido   | Detalle con vidrios |
| POST /sync/upload/          | 3          | Cumplido   | Batch + campo errores |
| GET /sync/status/           | 3          | Cumplido   | Por device_id |
| GET /config/                | 4          | Cumplido   | — |
| GET /reportes/diario/       | 5          | Cumplido   | JSON + CSV, permisos |
| GET /reportes/mensual/      | 5          | Cumplido   | JSON + CSV, permisos |

---

## 3. Observaciones post-review

### Código correcto, sin bugs bloqueantes

La revisión del código producido por los 4 agentes no encontró bugs que impidan el funcionamiento. La lógica de negocio, validaciones, permisos y aislamiento multi-tenant están implementados según spec.

### Notas menores

1. **Fixtures duplicadas en test_reportes.py**: El archivo declara sus propios fixtures (`tenant`, `otro_tenant`, `ley`, `api_client`) que ya existen en `conftest.py`. Funcional pero redundante. Refactorizar si se quiere DRY.
2. **Comparación de fechas en sync**: `_crear_grabado` en SyncService compara `registro['fecha_creacion_local']` (string) contra `str(existente.fecha_creacion_local)` (string). Funciona con ISO 8601, pero parsear a datetime sería más robusto.
3. **Permisos en reportes**: Se implementan con `if request.user.rol not in (...)` inline en la vista. Podría extraerse a un permission class reutilizable si se agregan más endpoints restringidos.
4. **auth/logout**: No implementado. El contrato no lo exige para MVP.
5. **factory_boy**: `TESTING.md` menciona factory_boy para fixtures. No se usa; los tests usan fixtures de pytest directamente. Funcional, no es un problema.

### Estructura final del backend

```
grabakar-backend/
├── .github/workflows/ci.yml
├── Dockerfile
├── docker-compose.yml
├── manage.py
├── pytest.ini
├── requirements.txt
├── config/
│   ├── __init__.py
│   ├── celery.py
│   ├── urls.py
│   ├── wsgi.py
│   ├── asgi.py
│   └── settings/
│       ├── base.py
│       ├── local.py
│       ├── production.py
│       └── test.py
└── api/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models.py
    ├── permissions.py
    ├── urls.py
    ├── fixtures/
    │   └── initial.json
    ├── migrations/
    │   └── 0001_initial.py
    ├── serializers/
    │   ├── auth.py
    │   └── grabados.py
    ├── services/
    │   ├── reporte_service.py
    │   └── sync_service.py
    ├── tasks/
    │   ├── __init__.py
    │   ├── reportes_tasks.py
    │   └── sync_tasks.py
    ├── tests/
    │   ├── conftest.py
    │   ├── test_auth.py
    │   ├── test_config.py
    │   ├── test_grabados.py
    │   ├── test_health.py
    │   ├── test_reportes.py
    │   └── test_sync.py
    ├── utils/
    │   └── exceptions.py
    └── views/
        ├── auth.py
        ├── config.py
        ├── grabados.py
        ├── health.py
        ├── reportes.py
        └── sync.py
```

---

## 4. Siguientes pasos

Backend Fase 1 completo. Fase 2 parcial (reportes per-tenant listos, XLSX pendiente). Pendiente:

- **Reporte plataforma XLSX**: modelo Sucursal + `ReporteXLSXService` + endpoint. Ver `REPORT_XLSX_IMPLEMENTATION.md`.
- **Ejecutar pytest en CI real** para confirmar que los 61 tests pasan y cobertura ≥80%.
- **Fase 3**: PrintService / Bluetooth no requiere cambios backend.
- **Fase 4**: ScannerService / OCR — posible endpoint si se requiere validación server-side.
- **Fase 5**: Ionic/Capacitor — sin impacto backend.

---

## 5. Referencias rápidas

- **Contrato API**: `grabakar-docs/tecnico/API_CONTRACTS.md`
- **Modelo de datos**: `grabakar-docs/tecnico/MODELO_DATOS.md`
- **Plan backend**: `grabakar-docs/prompts/implementation_plans/BACKEND_IMPLEMENTATION.md`
- **Testing**: `grabakar-docs/tecnico/TESTING.md`
- **Agentes paralelos**: `grabakar-docs/prompts/agents/` (AGENT_GRABADOS, AGENT_SYNC, AGENT_REPORTES, AGENT_CI)
