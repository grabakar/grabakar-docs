# Contratos de API

Base URL: `/api/v1/`  
Formato: JSON  
Autenticación: JWT Bearer token (header `Authorization: Bearer <token>`)  
Idioma de mensajes de error: Español

---

## 1. Autenticación

### POST /auth/login/

Requiere conexión a internet. Retorna tokens y configuración del tenant.

**Request:**
```json
{
  "username": "operador1",
  "password": "secret123"
}
```

**Response 200:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "offline_token": "eyJ...",
  "token_expiry": 10800,
  "offline_token_expiry": 259200,
  "usuario": {
    "id": 42,
    "username": "operador1",
    "nombre_completo": "Juan Pérez",
    "rol": "operador"
  },
  "tenant": {
    "id": 1,
    "nombre": "GrabaKar",
    "logo_url": "https://storage.example.com/logos/grabakar.png",
    "color_primario": "#1A73E8",
    "color_secundario": "#FFFFFF"
  },
  "leyes_activas": [
    { "id": 1, "nombre": "Ley 20.580", "descripcion": "Grabado obligatorio de patente en vidrios" }
  ],
  "vidrios_config": {
    "auto": [
      { "numero": 1, "nombre": "Parabrisas" },
      { "numero": 2, "nombre": "Luneta trasera" },
      { "numero": 3, "nombre": "Lateral delantero izq." },
      { "numero": 4, "nombre": "Lateral delantero der." },
      { "numero": 5, "nombre": "Lateral trasero izq." },
      { "numero": 6, "nombre": "Lateral trasero der." }
    ],
    "moto": [
      { "numero": 1, "nombre": "Parabrisas" }
    ]
  }
}
```

**Response 401:**
```json
{
  "error": "Credenciales inválidas",
  "code": "INVALID_CREDENTIALS"
}
```

**Notas:**
- `access_token`: JWT, expira en 3 horas (10800s). Se usa para todas las requests autenticadas.
- `refresh_token`: JWT, expira en 7 días. Se usa para obtener un nuevo `access_token`.
- `offline_token`: JWT, expira en 72 horas (259200s). Se almacena cifrado en el dispositivo. Permite operar la app sin conexión. No sirve para hacer requests al backend; solo para validación local de sesión.
- `vidrios_config`: Define los vidrios a grabar por tipo de vehículo. El frontend usa esta config para el flujo multi-vidrio.

---

### POST /auth/refresh/

**Request:**
```json
{
  "refresh_token": "eyJ..."
}
```

**Response 200:**
```json
{
  "access_token": "eyJ...",
  "token_expiry": 10800
}
```

**Response 401:**
```json
{
  "error": "Token de refresco expirado o inválido",
  "code": "INVALID_REFRESH_TOKEN"
}
```

---

## 2. Grabados

### POST /grabados/

Crea un grabado con sus vidrios asociados.

**Request:**
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "patente": "BBCL99",
  "vin_chasis": "1HGCM82633A004352",
  "orden_trabajo": "OC-2026-001",
  "usuario_responsable_id": 42,
  "ley_caso_id": 1,
  "tipo_movimiento": "venta",
  "tipo_vehiculo": "auto",
  "formato_impresion": "horizontal",
  "fecha_creacion_local": "2026-03-04T10:30:00-03:00",
  "device_id": "device-abc-123",
  "es_duplicado": false,
  "vidrios": [
    {
      "uuid": "660e8400-e29b-41d4-a716-446655440001",
      "numero_vidrio": 1,
      "nombre_vidrio": "Parabrisas",
      "fecha_impresion": "2026-03-04T10:32:00-03:00",
      "cantidad_impresiones": 1
    }
  ]
}
```

**Response 201:**
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "estado_sync": "sincronizado",
  "fecha_sincronizacion": "2026-03-04T14:30:05Z"
}
```

**Response 400:**
```json
{
  "errores": {
    "patente": ["La patente debe tener al menos 6 caracteres"],
    "tipo_movimiento": ["Valor inválido. Opciones: venta, demo, capacitacion"]
  }
}
```

**Validaciones:**
- `patente`: obligatorio, min 6 chars, alfanumérico
- `tipo_movimiento`: obligatorio, enum `[venta, demo, capacitacion]`
- `tipo_vehiculo`: obligatorio, enum `[auto, moto]`
- `formato_impresion`: obligatorio, enum `[horizontal, vertical]`
- `ley_caso_id`: obligatorio, debe existir y estar activa
- `uuid`: obligatorio, UUID v4, debe ser único

---

### GET /grabados/

Lista paginada. Operador: solo sus registros. Supervisor/Admin: todos los del tenant.

**Query params:**
- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `operador_id` (int, opcional, solo supervisor+)
- `tipo_movimiento` (string, opcional)
- `fecha_desde` (ISO date, opcional)
- `fecha_hasta` (ISO date, opcional)
- `patente` (string, búsqueda parcial, opcional)

**Response 200:**
```json
{
  "count": 150,
  "next": "/api/v1/grabados/?page=2",
  "previous": null,
  "results": [
    {
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
      "patente": "BBCL99",
      "tipo_movimiento": "venta",
      "tipo_vehiculo": "auto",
      "fecha_creacion_local": "2026-03-04T10:30:00-03:00",
      "fecha_sincronizacion": "2026-03-04T14:30:05Z",
      "usuario_responsable": {
        "id": 42,
        "nombre_completo": "Juan Pérez"
      },
      "cantidad_vidrios": 6,
      "vidrios_impresos": 6
    }
  ]
}
```

---

### GET /grabados/{uuid}/

**Response 200:**
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "patente": "BBCL99",
  "vin_chasis": "1HGCM82633A004352",
  "orden_trabajo": "OC-2026-001",
  "usuario_responsable": {
    "id": 42,
    "nombre_completo": "Juan Pérez"
  },
  "ley_caso": {
    "id": 1,
    "nombre": "Ley 20.580"
  },
  "tipo_movimiento": "venta",
  "tipo_vehiculo": "auto",
  "formato_impresion": "horizontal",
  "fecha_creacion_local": "2026-03-04T10:30:00-03:00",
  "fecha_sincronizacion": "2026-03-04T14:30:05Z",
  "device_id": "device-abc-123",
  "es_duplicado": false,
  "vidrios": [
    {
      "uuid": "660e8400-e29b-41d4-a716-446655440001",
      "numero_vidrio": 1,
      "nombre_vidrio": "Parabrisas",
      "fecha_impresion": "2026-03-04T10:32:00-03:00",
      "cantidad_impresiones": 1
    }
  ]
}
```

---

## 3. Sincronización

### POST /sync/upload/

Recibe un batch de grabados desde un dispositivo. Procesa cada uno: crea si UUID no existe, actualiza si UUID existe y `fecha_creacion_local` es más reciente.

**Request:**
```json
{
  "device_id": "device-abc-123",
  "batch": [
    {
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
      "patente": "BBCL99",
      "vin_chasis": "1HGCM82633A004352",
      "orden_trabajo": "OC-2026-001",
      "usuario_responsable_id": 42,
      "ley_caso_id": 1,
      "tipo_movimiento": "venta",
      "tipo_vehiculo": "auto",
      "formato_impresion": "horizontal",
      "fecha_creacion_local": "2026-03-04T10:30:00-03:00",
      "es_duplicado": false,
      "vidrios": [
        {
          "uuid": "660e8400-e29b-41d4-a716-446655440001",
          "numero_vidrio": 1,
          "nombre_vidrio": "Parabrisas",
          "fecha_impresion": "2026-03-04T10:32:00-03:00",
          "cantidad_impresiones": 1
        }
      ]
    }
  ]
}
```

**Response 200:**
```json
{
  "procesados": 10,
  "exitosos": 9,
  "fallidos": 1,
  "errores": [
    {
      "uuid": "770e8400-e29b-41d4-a716-446655440099",
      "error": "ley_caso_id 99 no existe",
      "code": "INVALID_REFERENCE"
    }
  ]
}
```

**Comportamiento:**
- Cada registro se procesa independientemente. Un error en uno no afecta a los demás.
- Si el UUID ya existe en el servidor y `fecha_creacion_local` del batch es posterior, se actualizan los campos (excepto UUID y patente).
- `fecha_sincronizacion` se setea al timestamp del servidor al momento de procesar.
- Tamaño máximo de batch: 100 registros por request.

---

### GET /sync/status/

Retorna estado de sincronización del dispositivo.

**Query params:**
- `device_id` (string, requerido)

**Response 200:**
```json
{
  "device_id": "device-abc-123",
  "ultima_sincronizacion": "2026-03-04T14:30:05Z",
  "registros_sincronizados": 245,
  "registros_pendientes_servidor": 0
}
```

---

## 4. Configuración

### GET /config/

Retorna configuración del tenant del usuario autenticado. Mismo payload que el campo `tenant` + `leyes_activas` + `vidrios_config` del login. Útil para refrescar config sin re-autenticarse.

**Response 200:**
```json
{
  "tenant": { "..." : "..." },
  "leyes_activas": [ "..." ],
  "vidrios_config": { "..." : "..." }
}
```

---

## 5. Reportes per-tenant (Fase 2 — implementado)

### GET /reportes/diario/

Reporte JSON/CSV per-tenant, filtrado por `fecha_creacion_local`. Solo supervisor/admin.

**Query params:**
- `fecha` (ISO date, default hoy)
- `operador_id` (int, opcional, solo supervisor+)
- `formato` (string, `json` | `csv`, default `json`)

**Response 200 (JSON):**
```json
{
  "fecha": "2026-03-04",
  "total_grabados": 25,
  "por_operador": [
    { "operador": "Juan Pérez", "cantidad": 15 },
    { "operador": "María López", "cantidad": 10 }
  ],
  "por_tipo": [
    { "tipo": "venta", "cantidad": 20 },
    { "tipo": "demo", "cantidad": 5 }
  ],
  "registros": [ "..." ]
}
```

**Response 200 (CSV):** Archivo descargable con headers:
```
uuid,patente,operador,fecha_registro,cantidad_impresiones,fecha_impresion,tipo_movimiento,tipo_vehiculo,device_id,estado_sync,fecha_sincronizacion
```

### GET /reportes/mensual/

Misma estructura que diario pero con `mes` (YYYY-MM) como parámetro en vez de `fecha`.

---

## 6. Reporte plataforma XLSX (Fase 2 — implementado)

### GET /reportes/plataforma/

Reporte XLSX multi-tab cross-tenant. Solo admin. Filtra por `fecha_sincronizacion`.

**Query params:**
- `fecha` (ISO date, default hoy). Rango: 1ro del mes de `fecha` hasta `fecha - 1 día`.
- `tenant_id` (int, opcional). Sin él: cross-tenant.
- `formato` (string, default `xlsx`)

**Response 200:** Archivo XLSX descargable con 3 pestañas.

**Content-Type:** `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
**Content-Disposition:** `attachment; filename="Reporte grabakar diario YYYY_Mon_DD.xlsx"`

**Pestaña 1 — Resumen:**

| CLIENTE | SUCURSAL | TOTAL PATENTES |
|---|---|---|

**Pestaña 2 — Resumen por dia:**

| CLIENTE | SUCURSAL | 1 | 2 | ... | N | TOTAL |
|---|---|---|---|---|---|---|

Columnas de día: 1 hasta `fecha.day - 1`. Agrupado por `fecha_sincronizacion`.

**Pestaña 3 — Patentes:**

| PATENTE1 | CANTIDAD | VIN | OT | RESPONSABLE | DISPOSITIVO | FECHA CREADA | FECHA SINCRONIZADA | NOMBRE | EMAIL | CLIENTE | SUCURSAL | TIPO CLIENTE |
|---|---|---|---|---|---|---|---|---|---|---|---|---|

**Response 403:**
```json
{
  "error": "No tienes permiso para esta operación",
  "code": "PERMISSION_DENIED"
}
```

**Requiere modelo Sucursal** y campos adicionales (implementado).

---

## 7. Códigos de Error Estándar

| Código HTTP | Code | Descripción |
|---|---|---|
| 400 | VALIDATION_ERROR | Datos de entrada inválidos (detalle en `errores`) |
| 401 | INVALID_CREDENTIALS | Login fallido |
| 401 | TOKEN_EXPIRED | Access token expirado |
| 401 | INVALID_REFRESH_TOKEN | Refresh token inválido o expirado |
| 403 | PERMISSION_DENIED | Rol insuficiente para la operación |
| 404 | NOT_FOUND | Recurso no encontrado |
| 409 | DUPLICATE_UUID | UUID ya existe (en creación directa, no sync) |
| 429 | RATE_LIMITED | Demasiadas requests |
| 500 | SERVER_ERROR | Error interno |

Todos los errores retornan:
```json
{
  "error": "Mensaje legible en español",
  "code": "ERROR_CODE",
  "detalles": {}
}
```
