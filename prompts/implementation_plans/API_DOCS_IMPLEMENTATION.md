# Implementation Plan: API Documentation with drf-spectacular

**Reference strategy:** `grabakar-docs/tecnico/API_DOCS_ESTRATEGIA.md`
**Scope:** `grabakar-backend/` only
**Estimated effort:** ~1 hour

---

## Pre-conditions

- All changes are in `grabakar-backend/`.
- Do NOT edit the strategy document (`grabakar-docs/tecnico/API_DOCS_ESTRATEGIA.md`).
- Do NOT modify any existing endpoint logic — this is purely additive.
- The reference contract document is `grabakar-docs/tecnico/API_CONTRACTS.md`. Use it to confirm response shapes when writing `@extend_schema` annotations.

---

## Task 1: Install drf-spectacular

**File:** `requirements.txt`

Add one line at the end:

```
drf-spectacular>=0.28.0
```

**Verification:** `pip install -r requirements.txt` completes without errors.

---

## Task 2: Configure settings

**File:** `config/settings/base.py`

### 2a. Add to INSTALLED_APPS

Add `'drf_spectacular'` to the `INSTALLED_APPS` list, after `'django_filters'`:

```python
INSTALLED_APPS = [
    ...
    'django_filters',
    'drf_spectacular',
    'django_celery_beat',
    ...
]
```

### 2b. Add DEFAULT_SCHEMA_CLASS to REST_FRAMEWORK

Add to the existing `REST_FRAMEWORK` dict:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ('rest_framework_simplejwt.authentication.JWTAuthentication',),
    'DEFAULT_PERMISSION_CLASSES': ('rest_framework.permissions.IsAuthenticated',),
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',),
    'EXCEPTION_HANDLER': 'api.utils.exceptions.custom_exception_handler',
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```

### 2c. Add SPECTACULAR_SETTINGS

Add after the `REST_FRAMEWORK` dict:

```python
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

---

## Task 3: Add URL routes for docs

**File:** `api/urls.py`

### 3a. Add imports

At the top of the file, add:

```python
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)
```

### 3b. Add URL patterns

Add three paths at the end of `urlpatterns`, before the router `include` lines:

```python
urlpatterns = [
    path('health/', health),
    path('auth/login/', LoginView.as_view()),
    path('auth/refresh/', RefreshView.as_view()),
    path('config/', tenant_config),
    path('sync/upload/', SyncUploadView.as_view()),
    path('sync/status/', SyncStatusView.as_view()),
    path('reportes/diario/', ReporteDiarioView.as_view()),
    path('reportes/mensual/', ReporteMensualView.as_view()),
    path('reportes/plataforma/', ReportePlataformaView.as_view()),
    path('admin/dashboard/', DashboardView.as_view()),
    path('admin/analytics/grabados/', AnalyticsGrabadosView.as_view()),
    path('admin/analytics/sync/', AnalyticsSyncView.as_view()),
    path('admin/', include(admin_router.urls)),

    # API Documentation
    path('schema/', SpectacularAPIView.as_view(), name='schema'),
    path('docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),

    path('', include(router.urls)),
]
```

**Verification:** Run `python manage.py runserver` and open `http://localhost:8000/api/v1/docs/` — Swagger UI should load with all ModelViewSet endpoints auto-documented.

---

## Task 4: Annotate api/views/health.py

**File:** `api/views/health.py`

The full file after changes:

```python
from django.utils.timezone import now
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from drf_spectacular.utils import extend_schema, inline_serializer
from rest_framework import serializers


@extend_schema(
    tags=['Health'],
    responses={200: inline_serializer('HealthResponse', fields={
        'status': serializers.CharField(),
        'timestamp': serializers.CharField(),
    })},
)
@api_view(['GET'])
@permission_classes([AllowAny])
def health(request):
    return Response({'status': 'ok', 'timestamp': now().isoformat()})
```

Note: `@extend_schema` must be placed BEFORE `@api_view` (outermost decorator).

---

## Task 5: Annotate api/views/auth.py

**File:** `api/views/auth.py`

### 5a. Add imports at top

```python
from drf_spectacular.utils import extend_schema, extend_schema_view, inline_serializer
from rest_framework import serializers as drf_serializers
```

### 5b. Annotate LoginView.post

Add `@extend_schema` decorator to the `post` method of `LoginView`:

```python
class LoginView(APIView):
    permission_classes = [AllowAny]

    @extend_schema(
        tags=['Auth'],
        request=inline_serializer('LoginRequest', fields={
            'username': drf_serializers.CharField(),
            'password': drf_serializers.CharField(),
        }),
        responses={
            200: inline_serializer('LoginResponse', fields={
                'access_token': drf_serializers.CharField(),
                'refresh_token': drf_serializers.CharField(),
                'offline_token': drf_serializers.CharField(),
                'token_expiry': drf_serializers.IntegerField(),
                'offline_token_expiry': drf_serializers.IntegerField(),
                'usuario': drf_serializers.DictField(),
                'tenant': drf_serializers.DictField(),
                'leyes_activas': drf_serializers.ListField(child=drf_serializers.DictField()),
                'vidrios_config': drf_serializers.DictField(),
            }),
            401: inline_serializer('LoginErrorResponse', fields={
                'error': drf_serializers.CharField(),
                'code': drf_serializers.CharField(),
            }),
        },
    )
    def post(self, request):
        ...  # existing code unchanged
```

### 5c. Annotate RefreshView

Add `@extend_schema_view` decorator to the `RefreshView` class:

```python
@extend_schema_view(
    post=extend_schema(
        tags=['Auth'],
        request=inline_serializer('RefreshRequest', fields={
            'refresh_token': drf_serializers.CharField(),
        }),
        responses={
            200: inline_serializer('RefreshResponse', fields={
                'access': drf_serializers.CharField(),
                'access_token': drf_serializers.CharField(),
            }),
            401: inline_serializer('RefreshErrorResponse', fields={
                'error': drf_serializers.CharField(),
                'code': drf_serializers.CharField(),
            }),
        },
    ),
)
class RefreshView(TokenRefreshView):
    serializer_class = CustomTokenRefreshSerializer
```

---

## Task 6: Annotate api/views/config.py

**File:** `api/views/config.py`

Add imports and decorator. The full file after changes:

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from drf_spectacular.utils import extend_schema, inline_serializer
from rest_framework import serializers

from api.models import LeyCaso, DEFAULT_VIDRIOS_CONFIG


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
def tenant_config(request):
    tenant = request.user.tenant
    leyes = list(LeyCaso.objects.filter(activa=True).values('id', 'nombre', 'descripcion'))
    vidrios_config = tenant.configuracion_json.get('vidrios_config', DEFAULT_VIDRIOS_CONFIG)
    return Response({
        'tenant': {'id': tenant.id, 'nombre': tenant.nombre, 'logo_url': tenant.logo_url or '', 'color_primario': tenant.color_primario, 'color_secundario': tenant.color_secundario},
        'leyes_activas': leyes,
        'vidrios_config': vidrios_config,
    })
```

---

## Task 7: Annotate api/views/sync.py

**File:** `api/views/sync.py`

### 7a. Add imports at top

```python
from drf_spectacular.utils import extend_schema, inline_serializer
from rest_framework import serializers as drf_serializers
```

### 7b. Annotate SyncUploadView.post

```python
class SyncUploadView(APIView):
    permission_classes = [IsAuthenticated]

    @extend_schema(
        tags=['Sync'],
        request=inline_serializer('SyncUploadRequest', fields={
            'device_id': drf_serializers.CharField(),
            'batch': drf_serializers.ListField(child=drf_serializers.DictField()),
        }),
        responses={200: inline_serializer('SyncUploadResponse', fields={
            'procesados': drf_serializers.IntegerField(),
            'exitosos': drf_serializers.IntegerField(),
            'fallidos': drf_serializers.IntegerField(),
            'errores': drf_serializers.ListField(child=drf_serializers.DictField()),
        })},
    )
    def post(self, request):
        ...  # existing code unchanged
```

### 7c. Annotate SyncStatusView.get

```python
class SyncStatusView(APIView):
    permission_classes = [IsAuthenticated]

    @extend_schema(
        tags=['Sync'],
        parameters=[
            {'name': 'device_id', 'in': 'query', 'required': True, 'schema': {'type': 'string'}},
        ],
        responses={200: inline_serializer('SyncStatusResponse', fields={
            'device_id': drf_serializers.CharField(),
            'ultima_sincronizacion': drf_serializers.DateTimeField(allow_null=True),
            'registros_sincronizados': drf_serializers.IntegerField(),
            'registros_pendientes_servidor': drf_serializers.IntegerField(),
        })},
    )
    def get(self, request):
        ...  # existing code unchanged
```

Note: For `parameters` in `SyncStatusView`, use `OpenApiParameter` instead of a raw dict:

```python
from drf_spectacular.utils import extend_schema, inline_serializer, OpenApiParameter
from drf_spectacular.types import OpenApiTypes

# Then in the decorator:
parameters=[
    OpenApiParameter('device_id', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=True),
],
```

---

## Task 8: Annotate api/views/admin.py

**File:** `api/views/admin.py`

### 8a. Add imports at top

```python
from drf_spectacular.utils import extend_schema, extend_schema_view, inline_serializer, OpenApiParameter
from drf_spectacular.types import OpenApiTypes
from rest_framework import serializers as drf_serializers
```

### 8b. Annotate DashboardView.get

```python
class DashboardView(APIView):
    permission_classes = [IsPlatformAdmin]

    @extend_schema(
        tags=['Admin - Dashboard'],
        parameters=[
            OpenApiParameter('fecha', type=OpenApiTypes.DATE, location=OpenApiParameter.QUERY, required=False),
        ],
        responses={200: inline_serializer('DashboardResponse', fields={
            'fecha': drf_serializers.CharField(),
            'grabados_hoy': drf_serializers.IntegerField(),
            'grabados_mes': drf_serializers.IntegerField(),
            'grabados_total': drf_serializers.IntegerField(),
            'tenants_activos': drf_serializers.IntegerField(),
            'usuarios_activos': drf_serializers.IntegerField(),
            'sync_pendientes': drf_serializers.IntegerField(),
            'sync_errores': drf_serializers.IntegerField(),
            'sync_tasa_exito': drf_serializers.FloatField(),
        })},
    )
    def get(self, request):
        ...  # existing code unchanged
```

### 8c. Annotate AnalyticsGrabadosView.get

```python
class AnalyticsGrabadosView(APIView):
    permission_classes = [IsPlatformAdmin]

    @extend_schema(
        tags=['Admin - Dashboard'],
        parameters=[
            OpenApiParameter('dias', type=OpenApiTypes.INT, location=OpenApiParameter.QUERY, required=False, description='Number of days (default 30)'),
            OpenApiParameter('tenant_id', type=OpenApiTypes.INT, location=OpenApiParameter.QUERY, required=False),
        ],
        responses={200: inline_serializer('AnalyticsGrabadosResponse', fields={
            'periodo': drf_serializers.DictField(),
            'serie': drf_serializers.ListField(child=drf_serializers.DictField()),
            'por_tenant': drf_serializers.ListField(child=drf_serializers.DictField()),
            'por_tipo_movimiento': drf_serializers.ListField(child=drf_serializers.DictField()),
        })},
    )
    def get(self, request):
        ...  # existing code unchanged
```

### 8d. Annotate AnalyticsSyncView.get

```python
class AnalyticsSyncView(APIView):
    permission_classes = [IsPlatformAdmin]

    @extend_schema(
        tags=['Admin - Dashboard'],
        parameters=[
            OpenApiParameter('dias', type=OpenApiTypes.INT, location=OpenApiParameter.QUERY, required=False, description='Number of days (default 30)'),
            OpenApiParameter('tenant_id', type=OpenApiTypes.INT, location=OpenApiParameter.QUERY, required=False),
        ],
        responses={200: inline_serializer('AnalyticsSyncResponse', fields={
            'resumen': drf_serializers.DictField(),
            'serie': drf_serializers.ListField(child=drf_serializers.DictField()),
        })},
    )
    def get(self, request):
        ...  # existing code unchanged
```

### 8e. Tag the ModelViewSets

Use `@extend_schema_view` on each admin ViewSet to assign tags:

```python
@extend_schema_view(
    list=extend_schema(tags=['Admin - Tenants']),
    retrieve=extend_schema(tags=['Admin - Tenants']),
    create=extend_schema(tags=['Admin - Tenants']),
    update=extend_schema(tags=['Admin - Tenants']),
    partial_update=extend_schema(tags=['Admin - Tenants']),
    destroy=extend_schema(tags=['Admin - Tenants']),
)
class TenantAdminViewSet(viewsets.ModelViewSet):
    ...
```

Repeat for each admin ViewSet with the corresponding tag:
- `SucursalAdminViewSet` → `'Admin - Sucursales'`
- `UsuarioAdminViewSet` → `'Admin - Usuarios'` (also annotate `reset_password` action)
- `GrabadoAdminViewSet` → `'Admin - Grabados'` (ReadOnlyModelViewSet, so only `list` and `retrieve`)
- `LeyCasoAdminViewSet` → `'Admin - Leyes'`

For `UsuarioAdminViewSet.reset_password` custom action:

```python
@extend_schema(
    tags=['Admin - Usuarios'],
    request=inline_serializer('ResetPasswordRequest', fields={
        'new_password': drf_serializers.CharField(min_length=8),
    }),
    responses={200: inline_serializer('ResetPasswordResponse', fields={
        'message': drf_serializers.CharField(),
    })},
)
@action(detail=True, methods=['post'])
def reset_password(self, request, pk=None):
    ...
```

---

## Task 9: Annotate api/views/reportes.py

**File:** `api/views/reportes.py`

### 9a. Add imports at top

```python
from drf_spectacular.utils import extend_schema, inline_serializer, OpenApiParameter
from drf_spectacular.types import OpenApiTypes
from rest_framework import serializers as drf_serializers
```

### 9b. Annotate ReporteDiarioView.get

```python
class ReporteDiarioView(APIView):
    permission_classes = [IsAuthenticated]

    @extend_schema(
        tags=['Reportes'],
        parameters=[
            OpenApiParameter('fecha', type=OpenApiTypes.DATE, location=OpenApiParameter.QUERY, required=False, description='Fecha (YYYY-MM-DD, default hoy)'),
            OpenApiParameter('operador_id', type=OpenApiTypes.INT, location=OpenApiParameter.QUERY, required=False),
            OpenApiParameter('formato', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=False, description='json | csv | xlsx (default json)'),
            OpenApiParameter('filtro_fecha', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=False, description='sync | creacion (default sync)'),
        ],
        responses={
            200: inline_serializer('ReporteDiarioResponse', fields={
                'fecha': drf_serializers.CharField(),
                'total_grabados': drf_serializers.IntegerField(),
                'por_operador': drf_serializers.ListField(child=drf_serializers.DictField()),
                'por_tipo': drf_serializers.ListField(child=drf_serializers.DictField()),
                'registros': drf_serializers.ListField(child=drf_serializers.DictField()),
            }),
            403: inline_serializer('PermissionDeniedResponse', fields={
                'error': drf_serializers.CharField(),
                'code': drf_serializers.CharField(),
            }),
        },
    )
    def get(self, request):
        ...  # existing code unchanged
```

### 9c. Annotate ReporteMensualView.get

```python
class ReporteMensualView(APIView):
    permission_classes = [IsAuthenticated]

    @extend_schema(
        tags=['Reportes'],
        parameters=[
            OpenApiParameter('mes', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=False, description='Mes (YYYY-MM, default mes actual)'),
            OpenApiParameter('operador_id', type=OpenApiTypes.INT, location=OpenApiParameter.QUERY, required=False),
            OpenApiParameter('formato', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=False, description='json | csv | xlsx (default json)'),
            OpenApiParameter('filtro_fecha', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=False, description='sync | creacion (default sync)'),
        ],
        responses={200: inline_serializer('ReporteMensualResponse', fields={
            'mes': drf_serializers.CharField(),
            'total_grabados': drf_serializers.IntegerField(),
            'por_operador': drf_serializers.ListField(child=drf_serializers.DictField()),
            'por_tipo': drf_serializers.ListField(child=drf_serializers.DictField()),
            'registros': drf_serializers.ListField(child=drf_serializers.DictField()),
        })},
    )
    def get(self, request):
        ...  # existing code unchanged
```

### 9d. Annotate ReportePlataformaView.get

```python
class ReportePlataformaView(APIView):
    permission_classes = [IsAuthenticated]

    @extend_schema(
        tags=['Reportes'],
        parameters=[
            OpenApiParameter('fecha_inicio', type=OpenApiTypes.DATE, location=OpenApiParameter.QUERY, required=True),
            OpenApiParameter('fecha_fin', type=OpenApiTypes.DATE, location=OpenApiParameter.QUERY, required=True),
            OpenApiParameter('filtro_fecha', type=OpenApiTypes.STR, location=OpenApiParameter.QUERY, required=False, description='sync | creacion (default sync)'),
        ],
        responses={
            (200, 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'): bytes,
            403: inline_serializer('PermissionDeniedReportePlataforma', fields={
                'error': drf_serializers.CharField(),
                'code': drf_serializers.CharField(),
            }),
        },
    )
    def get(self, request):
        ...  # existing code unchanged
```

---

## Task 10: Tag the operator GrabadoViewSet

**File:** `api/views/grabados.py`

### 10a. Add imports

```python
from drf_spectacular.utils import extend_schema, extend_schema_view
```

### 10b. Tag the ViewSet

```python
@extend_schema_view(
    list=extend_schema(tags=['Grabados']),
    retrieve=extend_schema(tags=['Grabados']),
    create=extend_schema(tags=['Grabados']),
    update=extend_schema(tags=['Grabados']),
    partial_update=extend_schema(tags=['Grabados']),
    destroy=extend_schema(tags=['Grabados']),
)
class GrabadoViewSet(TenantQuerySetMixin, viewsets.ModelViewSet):
    ...
```

---

## Task 11: Add schema validation to CI

**File:** `grabakar-backend/.github/workflows/ci.yml`

Add a new job `schema` after the `test` job:

```yaml
  schema:
    runs-on: ubuntu-latest
    env:
      DJANGO_SETTINGS_MODULE: config.settings.test
      DJANGO_SECRET_KEY: ci-test-secret-key-not-real
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('requirements.txt') }}

      - name: Install deps
        run: pip install -r requirements.txt

      - name: Validate OpenAPI schema
        run: python manage.py spectacular --validate --fail-on-warn
```

---

## Task 12: Verify locally

Run the following checks in order:

```bash
cd grabakar-backend

# 1. Install dependencies
pip install -r requirements.txt

# 2. Validate schema generation (no warnings)
python manage.py spectacular --validate --fail-on-warn

# 3. Start server and verify Swagger UI loads
python manage.py runserver
# Open http://localhost:8000/api/v1/docs/ in browser
# Open http://localhost:8000/api/v1/redoc/ in browser

# 4. Lint check (no ruff errors introduced)
ruff check .
```

---

## Files Modified (summary)

| File | Change |
|------|--------|
| `requirements.txt` | Add `drf-spectacular>=0.28.0` |
| `config/settings/base.py` | Add to `INSTALLED_APPS`, `REST_FRAMEWORK`, add `SPECTACULAR_SETTINGS` |
| `api/urls.py` | Add 3 URL patterns + imports |
| `api/views/health.py` | Add `@extend_schema` to `health` |
| `api/views/auth.py` | Add `@extend_schema` to `LoginView.post`, `@extend_schema_view` to `RefreshView` |
| `api/views/config.py` | Add `@extend_schema` to `tenant_config` |
| `api/views/sync.py` | Add `@extend_schema` to `SyncUploadView.post` and `SyncStatusView.get` |
| `api/views/admin.py` | Add `@extend_schema` to 3 custom views + `@extend_schema_view` to 5 ViewSets + `reset_password` action |
| `api/views/reportes.py` | Add `@extend_schema` to 3 report views |
| `api/views/grabados.py` | Add `@extend_schema_view` to `GrabadoViewSet` |
| `.github/workflows/ci.yml` | Add `schema` validation job |

## Files NOT modified

- No serializer files changed.
- No model files changed.
- No existing view logic changed.
- No URL routing logic changed (only additive paths).

---

## Important Notes for the Agent

1. **Do not rewrite view method bodies.** Only add decorators above method definitions. The `...` in code samples above means "keep existing code exactly as-is".
2. **Inline serializer names must be globally unique.** Each `inline_serializer('Name', ...)` name across the entire project must be different. The names in this plan are already unique.
3. **Decorator order matters for functional views.** For `@api_view` decorated functions, `@extend_schema` must be the outermost (topmost) decorator.
4. **Import aliasing.** In files that already `from rest_framework import serializers`, import `drf_spectacular` serializers as `drf_serializers` to avoid name collision. In files that don't import `serializers` from DRF, you can import it directly as `serializers`.
5. **Run `ruff check .` after all changes** to ensure no linting violations. Fix any that appear.
6. **The `config/settings/test.py` inherits from `local.py` which inherits from `base.py`**, so the schema validation CI step will pick up all the new settings automatically.
