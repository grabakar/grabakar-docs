# Agente paralelo: Sync (P1-04)

Asignación para un agente que trabaje **solo** en sincronización. No tocar auth, grabados CRUD, config ni reportes.

## Contexto

- **Repo**: `grabakar-backend/`
- **Reglas**: `AGENT_RULES.md`, `BACKEND_AGENT.md`
- **Contrato**: `grabakar-docs/tecnico/API_CONTRACTS.md` § 3. Sincronización
- **Código**: `api/views/sync.py`, `api/services/sync_service.py`

## Estado actual

- **Hecho**: POST /sync/upload/ con batch, validación batch vacío y límite 100, tenant en creación, procesamiento por UUID (crear/actualizar/ignorar), tests de batch válido/vacío/límite/401.
- **Falta**: GET /sync/status/ y alinear respuesta de upload con contrato (ver abajo).

## Tareas (orden sugerido)

### 1. GET /sync/status/

**Contrato**: Query param `device_id` (requerido). Response 200:

```json
{
  "device_id": "device-abc-123",
  "ultima_sincronizacion": "2026-03-04T14:30:05Z",
  "registros_sincronizados": 245,
  "registros_pendientes_servidor": 0
}
```

**Acción**:

- Crear vista (ej. `SyncStatusView`) GET, autenticada, que reciba `device_id` por query.
- `ultima_sincronizacion`: max `fecha_sincronizacion` de los Grabados del tenant del usuario con ese `device_id` (o null si no hay).
- `registros_sincronizados`: count de Grabados del tenant con ese `device_id` y `estado_sync='sincronizado'`.
- `registros_pendientes_servidor`: count de Grabados del tenant con `estado_sync='pendiente'` (opcional: mismo device_id o global; aclarar con contrato).
- Registrar en `api/urls.py`: `path('sync/status/', SyncStatusView.as_view())`.
- Tests: con y sin device_id, usuario autenticado, tenant aislado.

### 2. Respuesta POST /sync/upload/ y contrato

**Contrato**: además de `procesados`, `exitosos`, `fallidos`, pide un array `errores`:

```json
"errores": [
  { "uuid": "...", "error": "ley_caso_id 99 no existe", "code": "INVALID_REFERENCE" }
]
```

**Actual**: La respuesta tiene `resultados` (lista con status por ítem: ok, ignorado, error y `mensaje`).

**Acción**: Añadir en la respuesta de `SyncUploadView` un campo `errores` que sea la sublista de `resultados` con `status == 'error'`, mapeada a `{ "uuid", "error": mensaje, "code": "INVALID_REFERENCE" }` (o código según tipo de error). Mantener `resultados` si otros consumidores lo usan, o documentar el cambio.

### 3. Aceptar `vidrios` en el batch

**Contrato**: cada ítem del batch puede traer `vidrios` (array de impresiones).

**Actual**: `SyncService._crear_grabado` ya acepta `impresiones` o `vidrios` (`data.pop('impresiones', data.pop('vidrios', []))`).

**Acción**: Verificar que un batch con `vidrios` en cada ítem se procese bien; si falta algo en el mapeo de campos (uuid, numero_vidrio, etc.), corregir. Añadir test: batch con ítem que trae `vidrios` y verificar que se crean las impresiones.

### 4. No tocar

- `api/views/auth.py`, `api/views/grabados.py`, `api/views/config.py`, `api/serializers/grabados.py`, `api/models.py`.
- Solo tocar `api/urls.py` para registrar sync/status.

## Branch y commits

- Branch: `feature/f1-sync-status`
- Commits: ej. `feat(sync): GET sync/status por device_id`, `fix(sync): respuesta upload con campo errores según contrato`

## Verificación

- `pytest api/tests/test_sync.py -v`
- `ruff check api/`
- Contrato § 3 cumplido para GET /sync/status/ y POST /sync/upload/.
