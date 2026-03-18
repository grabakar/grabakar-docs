# Estrategia de Documentación y Testing de API

Estándar para el desarrollo, documentación y pruebas interactivas de la API REST de GrabaKar.

---

## Herramienta: drf-spectacular

Usamos **drf-spectacular** para generar automáticamente una especificación OpenAPI 3.0 a partir del código DRF existente. No mantenemos un archivo YAML manual — el código es la fuente de verdad.

### URLs disponibles

| URL | Descripción |
|-----|-------------|
| `/api/v1/docs/` | Swagger UI — interfaz interactiva para explorar y probar endpoints |
| `/api/v1/redoc/` | ReDoc — documentación de lectura |
| `/api/v1/schema/` | Especificación OpenAPI 3.0 (JSON) — para codegen y herramientas externas |

### ¿Por qué no Django-Ninja / YAML manual?

- `drf-spectacular` lee nuestros serializers, viewsets y permisos y genera la spec automáticamente. No hay riesgo de drift entre documentación y código real.
- No se introduce un segundo framework (Ninja) solo para servir docs.
- Los endpoints que usan `ModelSerializer` + `ModelViewSet` se documentan con cero configuración adicional.

---

## Estándar de Desarrollo de Endpoints

### 1. ViewSets basados en serializers (caso ideal)

Los endpoints CRUD que usan `ModelViewSet` + `ModelSerializer` se documentan automáticamente. Ejemplo actual:

- `GrabadoViewSet` → serializers `GrabadoListSerializer`, `GrabadoSerializer`, `GrabadoCreateSerializer`
- `TenantAdminViewSet`, `SucursalAdminViewSet`, `UsuarioAdminViewSet`, etc.

No requieren configuración extra. Swagger UI mostrará correctamente los campos, tipos, validaciones y respuestas.

### 2. Vistas custom (APIView con Response manual)

Las vistas que construyen el response manualmente **deben** usar el decorador `@extend_schema` para que el schema refleje la forma real del request/response.

```python
from drf_spectacular.utils import extend_schema, inline_serializer
from rest_framework import serializers

class SyncUploadView(APIView):
    @extend_schema(
        tags=['Sync'],
        request=inline_serializer('SyncUploadRequest', fields={
            'device_id': serializers.CharField(),
            'batch': serializers.ListField(child=serializers.DictField()),
        }),
        responses={200: inline_serializer('SyncUploadResponse', fields={
            'procesados': serializers.IntegerField(),
            'exitosos': serializers.IntegerField(),
            'fallidos': serializers.IntegerField(),
            'errores': serializers.ListField(child=serializers.DictField()),
        })},
    )
    def post(self, request): ...
```

### 3. Vistas funcionales (@api_view)

Misma regla — usar `@extend_schema`:

```python
@extend_schema(
    tags=['Config'],
    responses={200: inline_serializer('TenantConfigResponse', fields={
        'tenant': serializers.DictField(),
        'leyes_activas': serializers.ListField(child=serializers.DictField()),
        'vidrios_config': serializers.DictField(),
    })},
)
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def tenant_config(request): ...
```

### 4. Tags (agrupación de endpoints)

Cada grupo de endpoints lleva un tag que lo agrupa en Swagger UI:

| Tag | Endpoints |
|-----|-----------|
| `Auth` | `/auth/login/`, `/auth/refresh/` |
| `Grabados` | `/grabados/` (CRUD operador) |
| `Sync` | `/sync/upload/`, `/sync/status/` |
| `Config` | `/config/` |
| `Reportes` | `/reportes/diario/`, `/reportes/mensual/`, `/reportes/plataforma/` |
| `Admin - Dashboard` | `/admin/dashboard/`, `/admin/analytics/*` |
| `Admin - Tenants` | `/admin/tenants/` |
| `Admin - Sucursales` | `/admin/sucursales/` |
| `Admin - Usuarios` | `/admin/usuarios/` |
| `Admin - Grabados` | `/admin/grabados/` |
| `Admin - Leyes` | `/admin/leyes/` |
| `Health` | `/health/` |

Para ViewSets, el tag se asigna vía `SPECTACULAR_SETTINGS['POSTPROCESSING_HOOKS']` o con `@extend_schema_view`. Para vistas custom, se pone en `@extend_schema(tags=[...])`.

---

## Testing de Endpoints vía Web

### Swagger UI (`/api/v1/docs/`)

1. Abrir `http://localhost:8000/api/v1/docs/` en el navegador.
2. Para endpoints autenticados, primero ejecutar `POST /auth/login/` desde la misma UI.
3. Copiar el `access_token` del response.
4. Click en el botón **Authorize** (candado) arriba a la derecha.
5. Ingresar `Bearer <access_token>` en el campo.
6. Ahora todos los endpoints autenticados se pueden probar directamente.

### Desde herramientas externas

El schema en `/api/v1/schema/` es consumible por:

- **Postman**: importar como OpenAPI 3.0 collection.
- **Insomnia**: importar spec.
- **curl**: los ejemplos de request se pueden copiar desde Swagger UI.
- **openapi-typescript**: generar tipos TS para frontend.

```bash
# Generar tipos TypeScript desde la spec
npx openapi-typescript http://localhost:8000/api/v1/schema/ -o src/types/api.d.ts
```

---

## Configuración en el Proyecto

### `requirements.txt`

```
drf-spectacular==0.28.*
```

### `config/settings/base.py`

```python
INSTALLED_APPS = [
    ...
    'drf_spectacular',
]

REST_FRAMEWORK = {
    ...
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'GrabaKar API',
    'DESCRIPTION': 'API REST para el sistema de grabado de patentes vehiculares.',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'SCHEMA_PATH_PREFIX': '/api/v1/',
    'COMPONENT_SPLIT_REQUEST': True,
    'TAGS': [
        {'name': 'Auth', 'description': 'Autenticación JWT'},
        {'name': 'Grabados', 'description': 'CRUD de grabados (operador)'},
        {'name': 'Sync', 'description': 'Sincronización offline'},
        {'name': 'Config', 'description': 'Configuración del tenant'},
        {'name': 'Reportes', 'description': 'Reportes diario, mensual y plataforma'},
        {'name': 'Admin - Dashboard', 'description': 'KPIs y analíticas'},
        {'name': 'Admin - Tenants', 'description': 'Gestión de tenants'},
        {'name': 'Admin - Sucursales', 'description': 'Gestión de sucursales'},
        {'name': 'Admin - Usuarios', 'description': 'Gestión de usuarios'},
        {'name': 'Admin - Grabados', 'description': 'Consulta de grabados (cross-tenant)'},
        {'name': 'Admin - Leyes', 'description': 'Gestión de leyes/casos'},
        {'name': 'Health', 'description': 'Health check'},
    ],
}
```

### `api/urls.py`

```python
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    # ... endpoints existentes ...

    # Documentación API
    path('schema/', SpectacularAPIView.as_view(), name='schema'),
    path('docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

---

## Vistas que Requieren @extend_schema

Las siguientes vistas construyen respuestas manualmente y necesitan anotación:

| Vista | Archivo | Tag |
|-------|---------|-----|
| `LoginView.post` | `api/views/auth.py` | Auth |
| `RefreshView` | `api/views/auth.py` | Auth |
| `tenant_config` | `api/views/config.py` | Config |
| `SyncUploadView.post` | `api/views/sync.py` | Sync |
| `SyncStatusView.get` | `api/views/sync.py` | Sync |
| `health` | `api/views/health.py` | Health |
| `DashboardView.get` | `api/views/admin.py` | Admin - Dashboard |
| `AnalyticsGrabadosView.get` | `api/views/admin.py` | Admin - Dashboard |
| `AnalyticsSyncView.get` | `api/views/admin.py` | Admin - Dashboard |
| `ReporteDiarioView.get` | `api/views/reportes.py` | Reportes |
| `ReporteMensualView.get` | `api/views/reportes.py` | Reportes |
| `ReportePlataformaView.get` | `api/views/reportes.py` | Reportes |

Las siguientes vistas son auto-documentadas (ModelViewSet + serializer):

| Vista | Archivo | Tag |
|-------|---------|-----|
| `GrabadoViewSet` | `api/views/grabados.py` | Grabados |
| `TenantAdminViewSet` | `api/views/admin.py` | Admin - Tenants |
| `SucursalAdminViewSet` | `api/views/admin.py` | Admin - Sucursales |
| `UsuarioAdminViewSet` | `api/views/admin.py` | Admin - Usuarios |
| `GrabadoAdminViewSet` | `api/views/admin.py` | Admin - Grabados |
| `LeyCasoAdminViewSet` | `api/views/admin.py` | Admin - Leyes |

---

## Validación en CI

Agregar al workflow de CI del backend:

```yaml
- name: Validate OpenAPI schema
  run: python manage.py spectacular --validate --fail-on-warn
  env:
    DJANGO_SETTINGS_MODULE: config.settings.test
    DJANGO_SECRET_KEY: ci-test-secret-key-not-real
```

Esto detecta problemas de introspección (campos sin tipo, serializers ambiguos) antes de merge.

---

## Convención para PRs

Cuando un PR toca endpoints de la API:

1. Si es un ViewSet con serializer → no se necesita acción extra (auto-documentado).
2. Si es una vista custom (APIView / @api_view) → el PR **debe** incluir el decorador `@extend_schema` correspondiente.
3. Si se agrega un grupo nuevo de endpoints → agregar el tag en `SPECTACULAR_SETTINGS['TAGS']`.
4. Verificar localmente que `python manage.py spectacular --validate` pasa sin warnings.
