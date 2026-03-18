# Casos de Prueba: Data Consistency e Integridad del API

## Módulo: Base de Datos & Validaciones Modelos
**Nivel**: API Testing y Database Integration
**Prioridad Base**: Alta (P0)

### TC-DATA-001: Aislamiento Multi-Tenant Estricto
**Prioridad**: P0 | **Severidad**: S1 (Crítico)
**Precondiciones**:
- Tenants A y B en la base de datos con `Grabados`, `Sucursales` y `Usuarios` en cada uno.
- Token JWT obtenido para Operador de Tenant A.
**Pasos**:
1. Intentar listar sucursales: GET `/api/v1/sucursales/`.
2. Recuperar ID de un Grabado del Tenant B e intentar un GET `/api/v1/grabados/<id_tenant_b>/`.
3. Intentar crear un Grabado asignándolo forzosamente al Tenant B mediante payload (`tenant_id: id_B`)
**Resultado Esperado**:
- Paso 1 retorna las sucursales del Tenant A. No aparecen las del B.
- Paso 2 retorna `404 Not Found`. El queryset no expone recursos cruzados.
- Paso 3 el registro se guarda, pero sobreescribe internamente y asocia el guardado al Tenant A con el que el usuario se ha logueado. (Cualquier intento de suplantar credenciales desde el payload se obvia o se lanza `403/400`).

### TC-DATA-002: Lógica Unique-Push con UUIDs desde Sync
**Prioridad**: P1 | **Severidad**: S2
**Precondiciones**:
- Operador autenticado enviando un Batch.
**Pasos**:
1. Hacer un POST `/api/v1/sync/upload/` incluyendo un UUID "x000-..." con un grabado V1 en `estado_sync: pendiente`.
2. Validar Base de datos que Grabado existe.
3. Hacer otro POST idéntico con el UUID "x000-...", pero con `fecha_creacion_local` idéntica.
4. Hacer otro POST idéntico pero la `fecha_creacion_local` es en un string UTC que denota algo **MÁS RECIENTE**, como una corrección.
**Resultado Esperado**:
- El envío inicial guarda e inserta el Grabado en DB.
- Envío 2 es marcado como *ignorado*, de la misma manera que está en la respuesta de la API de sync.
- El 3 se entiende como Update, actualizando el registro original con los nuevos campos de metadata (si los hay) debido a resolución de conflictos por timestamp (latest-win).

### TC-DATA-003: Restricciones de Modelo (Patentes & VIN)
**Prioridad**: P1 | **Severidad**: S2
**Precondiciones**:
- POST a `/api/v1/grabados/`
**Pasos**:
1. Crear un registro donde la `patente` contenga un guión (`AB-CD-12`) o menos de 6 caracteres.
2. Crear un registro donde el valor de `VIN` tenga 16 o 18 caracteres.
3. Crear un registro dejando un `tipo_vehiculo` o `ley_caso_id` fuera de los Enums válidos permitidos (ej: `"SUV_VOLADOR"`).
**Resultado Esperado**:
- Todos los pasos resultan en error 400 Bad Request.
- Los mensajes de validación explican claramente los constraints alfanuméricos esperados por DRF (es decir, DRF Serializer Validations evitan que Database levante IntegrityErrors duros y en su lugar emite `ValidationError` JSON format).

### TC-DATA-004: Transacciones Atómicas (Impresiones)
**Prioridad**: P1 | **Severidad**: S2
**Setup**: Backend API (Create)
**Precondiciones**:
- Autenticado como operador del sistema de Grabado.
**Pasos**:
1. Someter un payload anidado (`grabado` + 6 `vidrios_impresos` y nested records).
2. Se introduce maliciosamente una estructura corrupta en el último de los objetos de los `vidrios_impresos` (ej un constraint roto que haga fallar a DRF / DB justo al querer insertar el vidrio No.6).
**Resultado Esperado**:
- Transacción Atómica entra en Rollback.
- El Grabado contenedor general no se inserta ni quema secuencia de IDs (no queda grabado huérfano con solo 5 vidrios).
- Se retorna Error al dispositivo para ser procesado/marcado.
