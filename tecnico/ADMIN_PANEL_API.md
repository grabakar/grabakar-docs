# Contratos de API — Admin Panel

Base URL: `/api/v1/admin/`
Formato: JSON
Autenticación: JWT Bearer token (header `Authorization: Bearer <token>`)
Permiso requerido: `is_staff=True` en el usuario autenticado
Idioma de mensajes de error: Español

---

## Permiso: IsPlatformAdmin

Todos los endpoints bajo `/api/v1/admin/` usan esta permission class:

```python
from rest_framework.permissions import BasePermission

class IsPlatformAdmin(BasePermission):
    """Acceso restringido a usuarios de plataforma (is_staff=True)."""
    message = "No tienes acceso al panel de administración."

    def has_permission(self, request, view):
        return (
            request.user
            and request.user.is_authenticated
            and request.user.is_staff
        )
```

Response 403 estándar para usuarios sin `is_staff`:

```json
{
  "error": "No tienes acceso al panel de administración.",
  "code": "PERMISSION_DENIED"
}
```

---

## 1. Dashboard

### GET /admin/dashboard/

Retorna KPIs agregados para la vista principal del admin panel.

**Query params:**
- `fecha` (ISO date, default hoy) — fecha de referencia para "hoy" y "mes actual"

**Response 200:**

```json
{
  "fecha": "2026-03-16",
  "grabados_hoy": 142,
  "grabados_mes": 3850,
  "grabados_total": 28500,
  "tenants_activos": 15,
  "usuarios_activos": 87,
  "sync_pendientes": 23,
  "sync_errores": 2,
  "sync_tasa_exito": 99.1
}
```

**Queries backend (aproximadas):**

```python
from django.utils import timezone
from django.db.models import Count, Q

hoy = timezone.now().date()
mes_inicio = hoy.replace(day=1)

dashboard = {
    "grabados_hoy": Grabado.objects.filter(fecha_creacion_local__date=hoy).count(),
    "grabados_mes": Grabado.objects.filter(fecha_creacion_local__date__gte=mes_inicio).count(),
    "grabados_total": Grabado.objects.count(),
    "tenants_activos": Tenant.objects.count(),
    "usuarios_activos": Usuario.objects.filter(activo=True).count(),
    "sync_pendientes": Grabado.objects.filter(estado_sync="pendiente").count(),
    "sync_errores": Grabado.objects.filter(estado_sync="error").count(),
}
```

---

## 2. Analytics

### GET /admin/analytics/grabados/

Serie temporal de grabados para gráficos. Agrupa por día.

**Query params:**
- `dias` (int, default 30) — cantidad de días hacia atrás desde hoy
- `tenant_id` (int, opcional) — filtrar por tenant específico
- `agrupacion` (string, default `dia`) — `dia` | `semana` | `mes`

**Response 200:**

```json
{
  "periodo": { "desde": "2026-02-14", "hasta": "2026-03-16" },
  "serie": [
    { "fecha": "2026-02-14", "total": 45 },
    { "fecha": "2026-02-15", "total": 52 },
    { "fecha": "2026-02-16", "total": 38 }
  ],
  "por_tenant": [
    { "tenant_id": 1, "tenant_nombre": "AutoGrab Chile", "total": 1200 },
    { "tenant_id": 2, "tenant_nombre": "VidrioPro", "total": 890 },
    { "tenant_id": 3, "tenant_nombre": "GrabMax", "total": 650 }
  ],
  "por_tipo_movimiento": [
    { "tipo": "venta", "total": 2100 },
    { "tipo": "demo", "total": 450 },
    { "tipo": "capacitacion", "total": 190 }
  ]
}
```

---

### GET /admin/analytics/sync/

Estado de sincronización agregado por día.

**Query params:**
- `dias` (int, default 30)
- `tenant_id` (int, opcional)

**Response 200:**

```json
{
  "resumen": {
    "sincronizado": 27500,
    "pendiente": 23,
    "error": 2
  },
  "serie": [
    {
      "fecha": "2026-03-15",
      "sincronizado": 140,
      "pendiente": 3,
      "error": 0
    },
    {
      "fecha": "2026-03-16",
      "sincronizado": 135,
      "pendiente": 5,
      "error": 1
    }
  ]
}
```

---

## 3. Tenants

### GET /admin/tenants/

Lista paginada de todos los tenants con conteos anotados.

**Query params:**
- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `search` (string, búsqueda parcial en `nombre`)
- `ordering` (string, default `-id`) — campos: `nombre`, `id`, `tipo_cliente`

**Response 200:**

```json
{
  "count": 15,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "nombre": "AutoGrab Chile",
      "tipo_cliente": "CONCESIÓN",
      "logo_url": "https://storage.example.com/logos/autograb.png",
      "color_primario": "#1A73E8",
      "color_secundario": "#FFFFFF",
      "configuracion_json": {},
      "sucursales_count": 3,
      "usuarios_count": 12,
      "grabados_mes_count": 450
    }
  ]
}
```

**Query backend:**

```python
Tenant.objects.annotate(
    sucursales_count=Count('sucursales'),
    usuarios_count=Count('usuarios', filter=Q(usuarios__activo=True)),
    grabados_mes_count=Count(
        'grabado',
        filter=Q(grabado__fecha_creacion_local__date__gte=mes_inicio)
    ),
)
```

---

### POST /admin/tenants/

Crea un nuevo tenant.

**Request:**

```json
{
  "nombre": "NuevaConcesionaria",
  "tipo_cliente": "CONCESIÓN",
  "logo_url": null,
  "color_primario": "#1976D2",
  "color_secundario": "#424242",
  "configuracion_json": {}
}
```

**Response 201:**

```json
{
  "id": 16,
  "nombre": "NuevaConcesionaria",
  "tipo_cliente": "CONCESIÓN",
  "logo_url": null,
  "color_primario": "#1976D2",
  "color_secundario": "#424242",
  "configuracion_json": {},
  "sucursales_count": 0,
  "usuarios_count": 0,
  "grabados_mes_count": 0
}
```

**Validaciones:**
- `nombre`: obligatorio, max 100 chars, único
- `color_primario`: formato hex 7 chars (#RRGGBB)
- `color_secundario`: formato hex 7 chars

---

### GET /admin/tenants/{id}/

Detalle de un tenant.

**Response 200:**

```json
{
  "id": 1,
  "nombre": "AutoGrab Chile",
  "tipo_cliente": "CONCESIÓN",
  "logo_url": "https://storage.example.com/logos/autograb.png",
  "color_primario": "#1A73E8",
  "color_secundario": "#FFFFFF",
  "configuracion_json": {
    "vidrios": {
      "auto": [
        { "numero": 1, "nombre": "Parabrisas" },
        { "numero": 2, "nombre": "Luneta trasera" }
      ]
    }
  },
  "sucursales_count": 3,
  "usuarios_count": 12,
  "grabados_mes_count": 450,
  "sucursales": [
    { "id": 1, "nombre": "Sede Santiago Centro", "activa": true },
    { "id": 2, "nombre": "Sede Providencia", "activa": true },
    { "id": 3, "nombre": "Sede Las Condes", "activa": false }
  ],
  "usuarios_recientes": [
    { "id": 42, "nombre_completo": "Juan Pérez", "rol": "operador", "activo": true, "last_login": "2026-03-16T08:30:00Z" }
  ]
}
```

---

### PUT /admin/tenants/{id}/

Actualiza un tenant. Mismos campos que POST. Retorna el tenant actualizado con conteos.

---

## 4. Sucursales

### GET /admin/sucursales/

Lista paginada de todas las sucursales.

**Query params:**
- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `tenant_id` (int, opcional) — filtrar por tenant
- `activa` (boolean, opcional) — filtrar por estado
- `search` (string, búsqueda en `nombre`)
- `ordering` (string, default `tenant__nombre,nombre`)

**Response 200:**

```json
{
  "count": 45,
  "next": "/api/v1/admin/sucursales/?page=2",
  "previous": null,
  "results": [
    {
      "id": 1,
      "nombre": "Sede Santiago Centro",
      "tenant": { "id": 1, "nombre": "AutoGrab Chile" },
      "activa": true,
      "grabados_mes_count": 180
    }
  ]
}
```

---

### POST /admin/sucursales/

**Request:**

```json
{
  "nombre": "Sede Viña del Mar",
  "tenant_id": 1,
  "activa": true
}
```

**Response 201:**

```json
{
  "id": 46,
  "nombre": "Sede Viña del Mar",
  "tenant": { "id": 1, "nombre": "AutoGrab Chile" },
  "activa": true,
  "grabados_mes_count": 0
}
```

**Validaciones:**
- `nombre`: obligatorio, max 150 chars
- `tenant_id`: obligatorio, debe existir
- `(tenant_id, nombre)`: combinación única

---

### GET /admin/sucursales/{id}/

Detalle de sucursal. Misma estructura que en listado, más campos adicionales.

### PUT /admin/sucursales/{id}/

Actualiza sucursal. Campos: `nombre`, `activa`. No se puede cambiar el tenant.

---

## 5. Usuarios

### GET /admin/usuarios/

Lista paginada de todos los usuarios.

**Query params:**
- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `tenant_id` (int, opcional)
- `rol` (string, opcional) — `operador` | `supervisor` | `admin`
- `activo` (boolean, opcional)
- `search` (string, búsqueda en `nombre_completo`, `username`)
- `ordering` (string, default `-last_login`)

**Response 200:**

```json
{
  "count": 87,
  "next": "/api/v1/admin/usuarios/?page=2",
  "previous": null,
  "results": [
    {
      "id": 42,
      "username": "jperez",
      "nombre_completo": "Juan Pérez",
      "email": "jperez@autograb.cl",
      "rol": "operador",
      "tenant": { "id": 1, "nombre": "AutoGrab Chile" },
      "sucursal": { "id": 1, "nombre": "Sede Santiago Centro" },
      "activo": true,
      "is_staff": false,
      "last_login": "2026-03-16T08:30:00Z",
      "date_joined": "2025-11-15T10:00:00Z"
    }
  ]
}
```

---

### POST /admin/usuarios/

Crea un nuevo usuario.

**Request:**

```json
{
  "username": "mlopez",
  "password": "temp_password_123",
  "nombre_completo": "María López",
  "email": "mlopez@autograb.cl",
  "rol": "operador",
  "tenant_id": 1,
  "sucursal_id": 1,
  "activo": true,
  "is_staff": false
}
```

**Response 201:**

```json
{
  "id": 88,
  "username": "mlopez",
  "nombre_completo": "María López",
  "email": "mlopez@autograb.cl",
  "rol": "operador",
  "tenant": { "id": 1, "nombre": "AutoGrab Chile" },
  "sucursal": { "id": 1, "nombre": "Sede Santiago Centro" },
  "activo": true,
  "is_staff": false,
  "last_login": null,
  "date_joined": "2026-03-16T14:00:00Z"
}
```

**Validaciones:**
- `username`: obligatorio, único, alfanumérico + guiones/puntos
- `password`: obligatorio en creación, min 8 chars
- `nombre_completo`: obligatorio
- `rol`: obligatorio, enum `[operador, supervisor, admin]`
- `tenant_id`: obligatorio, debe existir
- `sucursal_id`: opcional, debe pertenecer al tenant indicado

---

### GET /admin/usuarios/{id}/

Detalle de usuario. Misma estructura que en listado.

### PATCH /admin/usuarios/{id}/

Actualización parcial. Campos editables: `nombre_completo`, `email`, `rol`, `sucursal_id`, `activo`, `is_staff`. Password se maneja por separado.

### POST /admin/usuarios/{id}/reset-password/

Reset de contraseña desde el admin panel.

**Request:**

```json
{
  "new_password": "nueva_password_456"
}
```

**Response 200:**

```json
{
  "message": "Contraseña actualizada exitosamente."
}
```

---

## 6. Grabados

### GET /admin/grabados/

Lista paginada cross-tenant de todos los grabados.

**Query params:**
- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `tenant_id` (int, opcional)
- `sucursal_id` (int, opcional)
- `operador_id` (int, opcional)
- `estado_sync` (string, opcional) — `pendiente` | `sincronizado` | `error`
- `tipo_movimiento` (string, opcional)
- `tipo_vehiculo` (string, opcional)
- `patente` (string, búsqueda parcial)
- `fecha_desde` (ISO date, opcional)
- `fecha_hasta` (ISO date, opcional)
- `es_duplicado` (boolean, opcional)
- `ordering` (string, default `-fecha_creacion_local`)

**Response 200:**

```json
{
  "count": 28500,
  "next": "/api/v1/admin/grabados/?page=2",
  "previous": null,
  "results": [
    {
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
      "patente": "BBCL99",
      "vin_chasis": "1HGCM82633A004352",
      "tipo_movimiento": "venta",
      "tipo_vehiculo": "auto",
      "usuario_responsable": {
        "id": 42,
        "nombre_completo": "Juan Pérez"
      },
      "tenant": { "id": 1, "nombre": "AutoGrab Chile" },
      "sucursal": { "id": 1, "nombre": "Sede Santiago Centro" },
      "fecha_creacion_local": "2026-03-16T10:30:00-03:00",
      "fecha_sincronizacion": "2026-03-16T14:30:05Z",
      "estado_sync": "sincronizado",
      "es_duplicado": false,
      "cantidad_vidrios": 6,
      "vidrios_impresos": 6
    }
  ]
}
```

---

### GET /admin/grabados/{uuid}/

Detalle completo de un grabado con sus impresiones de vidrio.

**Response 200:**

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "patente": "BBCL99",
  "vin_chasis": "1HGCM82633A004352",
  "orden_trabajo": "OC-2026-001",
  "responsable_texto": "Carlos Muñoz",
  "usuario_responsable": {
    "id": 42,
    "nombre_completo": "Juan Pérez"
  },
  "ley_caso": {
    "id": 1,
    "nombre": "Ley 20.580"
  },
  "tenant": { "id": 1, "nombre": "AutoGrab Chile" },
  "sucursal": { "id": 1, "nombre": "Sede Santiago Centro" },
  "tipo_movimiento": "venta",
  "tipo_vehiculo": "auto",
  "formato_impresion": "horizontal",
  "fecha_creacion_local": "2026-03-16T10:30:00-03:00",
  "fecha_sincronizacion": "2026-03-16T14:30:05Z",
  "device_id": "device-abc-123",
  "estado_sync": "sincronizado",
  "es_duplicado": false,
  "vidrios": [
    {
      "uuid": "660e8400-e29b-41d4-a716-446655440001",
      "numero_vidrio": 1,
      "nombre_vidrio": "Parabrisas",
      "fecha_impresion": "2026-03-16T10:32:00-03:00",
      "impreso": true,
      "cantidad_impresiones": 1
    },
    {
      "uuid": "660e8400-e29b-41d4-a716-446655440002",
      "numero_vidrio": 2,
      "nombre_vidrio": "Luneta trasera",
      "fecha_impresion": "2026-03-16T10:33:00-03:00",
      "impreso": true,
      "cantidad_impresiones": 1
    }
  ]
}
```

---

## 7. LeyCaso (Configuración)

### GET /admin/leyes/

Lista todas las leyes/normativas.

**Response 200:**

```json
{
  "results": [
    {
      "id": 1,
      "nombre": "Ley 20.580",
      "descripcion": "Grabado obligatorio de patente en vidrios de vehículos",
      "activa": true
    }
  ]
}
```

---

### POST /admin/leyes/

**Request:**

```json
{
  "nombre": "Ley 21.000",
  "descripcion": "Nueva normativa de trazabilidad",
  "activa": true
}
```

**Response 201:** Objeto creado.

### PUT /admin/leyes/{id}/

Actualiza una ley. Campos: `nombre`, `descripcion`, `activa`.

---

## 8. Cambios al Login Existente

### Modificación al serializer de `POST /api/v1/auth/login/`

Agregar `is_staff` al objeto `usuario` en la response. Esto permite al frontend admin verificar acceso sin un endpoint adicional.

**Response actual (ampliada):**

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "offline_token": "eyJ...",
  "usuario": {
    "id": 1,
    "username": "plataforma_admin",
    "nombre_completo": "Admin GrabaKar",
    "rol": "admin",
    "is_staff": true
  },
  "tenant": { "..." : "..." },
  "leyes_activas": [ "..." ],
  "vidrios_config": { "..." : "..." }
}
```

Impacto: campo nuevo, no rompe la app móvil (simplemente lo ignora).

---

## Resumen de Implementación Backend

### Archivos nuevos

| Archivo | Contenido |
|---------|-----------|
| `api/views/admin.py` | `DashboardView`, `TenantViewSet`, `SucursalAdminViewSet`, `UsuarioAdminViewSet`, `GrabadoAdminViewSet`, `AnalyticsGrabadosView`, `AnalyticsSyncView`, `LeyCasoAdminViewSet` |
| `api/serializers/admin.py` | `DashboardSerializer`, `TenantAdminSerializer`, `SucursalAdminSerializer`, `UsuarioAdminSerializer`, `UsuarioCreateSerializer`, `GrabadoAdminListSerializer`, `GrabadoAdminDetailSerializer`, `LeyCasoAdminSerializer` |

### Archivos modificados

| Archivo | Cambio |
|---------|--------|
| `api/permissions.py` | Agregar `IsPlatformAdmin` |
| `api/urls.py` | Registrar router con rutas `/admin/*` |
| `api/serializers/auth.py` | Agregar `is_staff` al serializer de login |
| `config/settings/base.py` | Agregar CORS origin del admin panel |

### Sin migraciones

Todos los campos usados (`is_staff`, `is_superuser`, `last_login`, `date_joined`) ya existen en `AbstractUser`. No se requiere `makemigrations`.

---

## Códigos de Error

Mismos códigos que la API principal (ver [API_CONTRACTS.md](API_CONTRACTS.md#7-códigos-de-error-estándar)), más:

| Código HTTP | Code | Descripción |
|---|---|---|
| 403 | PLATFORM_ACCESS_DENIED | Usuario autenticado no tiene `is_staff=True` |
| 400 | DUPLICATE_SUCURSAL | Combinación `(tenant_id, nombre)` ya existe |
| 400 | INVALID_SUCURSAL_TENANT | `sucursal_id` no pertenece al `tenant_id` del usuario |

---

## Deployment

### Docker Compose (desarrollo local)

Agregar un servicio para el admin panel:

```yaml
admin:
  build:
    context: ../grabakar-admin
    dockerfile: Dockerfile
  ports:
    - "5174:80"
  environment:
    - VITE_API_URL=http://localhost:8000
  depends_on:
    - api
```

### GCP (producción)

El admin panel es una SPA estática. Opciones:

1. **Cloud Storage + Cloud CDN** — servir como sitio estático (recomendado, más simple)
2. **Cloud Run** — contenedor Nginx sirviendo el build, si se necesita más control

La API es la misma instancia de Cloud Run del backend existente. Solo se agregan rutas.
