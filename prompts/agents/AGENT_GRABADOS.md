# Agente paralelo: Grabados (P1-03)

Asignación para un agente que trabaje **solo** en el dominio Grabados. No tocar auth, sync, config ni reportes.

## Contexto

- **Repo**: `grabakar-backend/`
- **Reglas**: `AGENT_RULES.md`, `BACKEND_AGENT.md`
- **Contrato**: `grabakar-docs/tecnico/API_CONTRACTS.md` § 2. Grabados
- **Modelo**: `api/models.py` — `Grabado`, `ImpresionVidrio`

## Estado actual

- **Hecho**: ViewSet CRUD, `GrabadoSerializer` / `GrabadoCreateSerializer`, filtro por tenant y rol (operador solo propios), validación patente/VIN, tests de crear/listar/detalle/validaciones/tenant.
- **Falta**: Ver checklist abajo.

## Tareas (orden sugerido)

### 1. List: contrato API vs implementación

**Contrato** (GET /grabados/): cada ítem debe tener:

- `usuario_responsable`: `{ "id", "nombre_completo" }` (no el PK plano)
- `cantidad_vidrios`: número
- `vidrios_impresos`: número

**Actual**: El list usa el mismo `GrabadoSerializer` que el detalle (incluye `ley_caso`, `vidrios` completos). No hay `cantidad_vidrios` ni `vidrios_impresos`.

**Acción**: Añadir un serializer de listado (ej. `GrabadoListSerializer`) con:

- `usuario_responsable` como nested `{ id, nombre_completo }` (o `Serializers.SerializerMethodField` / `PrimaryKeyRelatedField` con custom `to_representation`).
- `cantidad_vidrios`: `Serializers.IntegerField(source='impresiones.count')` o anotación en el queryset.
- `vidrios_impresos`: contar impresiones con `impreso=True` o `cantidad_impresiones > 0` (según definición en contrato).
- En el ViewSet, usar `GrabadoListSerializer` para `action == 'list'` y `GrabadoSerializer` para retrieve.

### 2. Query params GET /grabados/

**Contrato**: `page`, `page_size` (max 100), `operador_id`, `tipo_movimiento`, `fecha_desde`, `fecha_hasta`, `patente` (búsqueda parcial).

**Actual**: Solo `filterset_fields = ['tipo_movimiento', 'estado_sync']`.

**Acción**:

- Añadir filtros: `operador_id` (solo si `request.user.rol` in `['supervisor','admin']`), `fecha_desde`, `fecha_hasta` (sobre `fecha_creacion_local`), `patente` (icontains o similar).
- Respetar `page_size` max 100 (DRF `PAGE_SIZE` o `MaxPageSizePagination`).
- Tests: al menos un test por filtro y uno de paginación > 20 resultados.

### 3. Tests faltantes (plan)

- `test_listar_grabados_supervisor_ve_todos_del_tenant`: supervisor ve grabados de todos los operadores del tenant.
- `test_paginacion`: > 20 resultados devuelve `next` y segunda página correcta.

### 4. No tocar

- `api/views/auth.py`, `api/views/sync.py`, `api/views/config.py`, `api/urls.py` (solo añadir rutas si el plan lo pide; el router ya incluye grabados).
- `api/services/sync_service.py`, `api/models.py` (salvo anotaciones en el queryset si se usan para list).

## Branch y commits

- Branch: `feature/f1-grabados-list-contract`
- Commits: Conventional Commits en español, ej. `feat(grabados): list serializer con cantidad_vidrios y vidrios_impresos`

## Verificación

- `pytest api/tests/test_grabados.py -v`
- `ruff check api/`
- Comparar respuesta GET /grabados/ con `API_CONTRACTS.md` § 2.
