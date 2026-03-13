# Backend Agent — Implementation Plan

Technical implementation guide for all backend tasks in `grabakar-backend`. This document eliminates ambiguity: follow it step-by-step.

---

## Repository Setup (P0-01)

### Step 1: Initialize Django Project

```bash
mkdir grabakar-backend && cd grabakar-backend
python -m venv .venv && source .venv/bin/activate
pip install django==5.1 djangorestframework djangorestframework-simplejwt \
  django-cors-headers django-filter celery redis psycopg2-binary \
  factory-boy pytest pytest-django pytest-cov ruff gunicorn dj-database-url \
  django-ratelimit django-celery-beat
pip freeze > requirements.txt
django-admin startproject config .
python manage.py startapp api
```

### Step 2: Settings Split

Create `config/settings/` directory with three files:

**`config/settings/base.py`** — Shared settings:
```python
import os
from pathlib import Path
from datetime import timedelta

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = os.getenv('DJANGO_SECRET_KEY', 'change-me')
DEBUG = os.getenv('DJANGO_DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.getenv('DJANGO_ALLOWED_HOSTS', 'localhost,127.0.0.1').split(',')

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework_simplejwt',
    'rest_framework_simplejwt.token_blacklist',
    'corsheaders',
    'django_filters',
    'django_celery_beat',
    'api',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'config.urls'
AUTH_USER_MODEL = 'api.Usuario'
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# DRF
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ('rest_framework_simplejwt.authentication.JWTAuthentication',),
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',),
    'EXCEPTION_HANDLER': 'api.utils.custom_exception_handler',
}

# JWT
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=int(os.getenv('JWT_ACCESS_TOKEN_LIFETIME_HOURS', 3))),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=int(os.getenv('JWT_REFRESH_TOKEN_LIFETIME_DAYS', 7))),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
}

# Celery
CELERY_BROKER_URL = os.getenv('CELERY_BROKER_URL', 'redis://redis:6379/0')
CELERY_RESULT_BACKEND = os.getenv('CELERY_RESULT_BACKEND', 'redis://redis:6379/1')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'

# CORS
CORS_ALLOWED_ORIGINS = os.getenv('CORS_ALLOWED_ORIGINS', 'http://localhost:5173').split(',')
CORS_ALLOW_CREDENTIALS = True

# Database — overridden per environment
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'grabakar'),
        'USER': os.getenv('DB_USER', 'grabakar'),
        'PASSWORD': os.getenv('DB_PASSWORD', 'grabakar_dev'),
        'HOST': os.getenv('DB_HOST', 'postgres'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
LANGUAGE_CODE = 'es-cl'
TIME_ZONE = 'America/Santiago'
USE_TZ = True
```

**`config/settings/local.py`**:
```python
from .base import *
DEBUG = True
```

**`config/settings/production.py`**:
```python
from .base import *
DEBUG = False
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

Delete the original `config/settings.py` file.

### Step 3: Docker Files

**`Dockerfile`** — Exact content from `tecnico/DEPLOYMENT.md` (multi-stage: builder + runtime).

**`docker-compose.yml`** — Exact content from `tecnico/DEPLOYMENT.md` (5 services: django, celery, celery-beat, postgres, redis).

**`.env.example`** — Exact content from `tecnico/DEPLOYMENT.md` environment variables table. Copy to `.env` for local dev.

### Step 4: Project Config Files

**`config/urls.py`**:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('api.urls')),
]
```

**`config/celery.py`**:
```python
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.local')
app = Celery('config')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

**`config/__init__.py`**: Add `from .celery import app as celery_app; __all__ = ('celery_app',)`

**`pytest.ini`**:
```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.local
python_files = tests.py test_*.py *_tests.py
```

### Step 5: Folder Structure

```
api/
├── __init__.py
├── models.py           # All models in one file
├── admin.py
├── urls.py
├── permissions.py
├── views/
│   ├── __init__.py
│   ├── auth.py
│   ├── grabados.py
│   ├── sync.py
│   ├── config.py
│   └── reportes.py
├── serializers/
│   ├── __init__.py
│   ├── auth.py
│   ├── grabados.py
│   └── sync.py
├── services/
│   ├── __init__.py
│   ├── sync_service.py
│   └── reporte_service.py
├── tasks/
│   ├── __init__.py
│   └── sync_tasks.py
├── utils/
│   ├── __init__.py
│   └── exceptions.py
└── tests/
    ├── __init__.py
    ├── conftest.py      # Shared fixtures (factories, api_client)
    ├── test_auth.py
    ├── test_grabados.py
    ├── test_sync.py
    └── test_config.py
```

### Step 6: `.cursorrules`

Create at repo root. Content:

```
# grabakar-backend — Agent Context

Read `AGENT_RULES.md` and `BACKEND_AGENT.md` from grabakar-docs/prompts/ before any work.
Read the relevant feature spec from grabakar-docs/features/ for your task.
Read `BACKEND_IMPLEMENTATION.md` from grabakar-docs/prompts/implementation_plans/ for exact steps.

## Stack
Django 5+ | DRF | PostgreSQL 16+ | Celery + Redis | Docker Compose | pytest + factory_boy

## Key Rules
- Single app: `api/`. No multi-app split.
- Business logic in `api/services/`, NEVER in views or serializers.
- ALL querysets filter by `request.user.tenant`. Use TenantQuerySetMixin.
- ALL user-facing text in Spanish (Chile).
- NEVER hardcode "GrabaKar". Use Tenant model.
- Tests for all business logic. Min 80% coverage on services/ and serializers/.
- Conventional commits in Spanish: `feat(auth): implementar login JWT`
- Branch naming: `feature/<fase>-<nombre>` (e.g. `feature/f1-login-auth`)

## PDCA
CHECK (read state) > ACT (diagnose) > PLAN (define steps) > DO (implement)

## Docs Location
- API contracts: grabakar-docs/tecnico/API_CONTRACTS.md
- Data model: grabakar-docs/tecnico/MODELO_DATOS.md
- Feature specs: grabakar-docs/features/
- Implementation plans: grabakar-docs/prompts/implementation_plans/
```

### Step 7: Health Endpoint

In `api/views/health.py`:
```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from django.utils.timezone import now

@api_view(['GET'])
@permission_classes([AllowAny])
def health(request):
    return Response({'status': 'ok', 'timestamp': now().isoformat()})
```

Add to `api/urls.py`: `path('health/', health)`

### Verification

```bash
docker compose up -d
# Wait for services, then:
curl http://localhost:8000/api/v1/health/  # → {"status": "ok", ...}
docker compose exec django python manage.py test  # 0 tests, 0 errors
docker compose exec django ruff check .  # No lint errors
```

---

## Models (P1-01)

### Exact Models

Copy the exact model code from `tecnico/MODELO_DATOS.md`. The five models are:

1. **`Tenant`** — `nombre`, `logo_url`, `color_primario`, `color_secundario`, `configuracion_json` (JSONField)
2. **`Usuario`** — extends `AbstractUser`, adds `tenant` (FK), `nombre_completo`, `rol` (TextChoices: operador/supervisor/admin), `activo`
3. **`LeyCaso`** — `nombre`, `descripcion`, `activa`
4. **`Grabado`** — UUID PK, `patente`, `vin_chasis`, `orden_trabajo`, `usuario_responsable` (FK Usuario, PROTECT), `ley_caso` (FK, PROTECT), `tipo_movimiento`, `tipo_vehiculo`, `formato_impresion`, `fecha_creacion_local`, `fecha_sincronizacion` (nullable), `device_id`, `estado_sync`, `es_duplicado`, `tenant` (FK, CASCADE). Indexes: `idx_duplicado`, `idx_sync_queue`, `idx_purge`, `idx_tenant`.
5. **`ImpresionVidrio`** — UUID PK, `grabado` (FK Grabado, CASCADE), `numero_vidrio`, `nombre_vidrio`, `fecha_impresion` (nullable), `impreso`, `cantidad_impresiones`. Unique together: `(grabado, numero_vidrio)`.

### Admin Registration

```python
# api/admin.py
from django.contrib import admin
from .models import Tenant, Usuario, LeyCaso, Grabado, ImpresionVidrio

@admin.register(Tenant)
class TenantAdmin(admin.ModelAdmin):
    list_display = ('nombre', 'color_primario')

@admin.register(Usuario)
class UsuarioAdmin(admin.ModelAdmin):
    list_display = ('username', 'nombre_completo', 'rol', 'tenant', 'activo')
    list_filter = ('rol', 'tenant', 'activo')

@admin.register(LeyCaso)
class LeyCasoAdmin(admin.ModelAdmin):
    list_display = ('nombre', 'activa')

@admin.register(Grabado)
class GrabadoAdmin(admin.ModelAdmin):
    list_display = ('uuid', 'patente', 'tipo_movimiento', 'estado_sync', 'tenant')
    list_filter = ('estado_sync', 'tipo_movimiento', 'tenant')

@admin.register(ImpresionVidrio)
class ImpresionVidrioAdmin(admin.ModelAdmin):
    list_display = ('uuid', 'grabado', 'nombre_vidrio', 'cantidad_impresiones')
```

### Initial Fixtures

Create `api/fixtures/initial.json`:
```json
[
  {"model": "api.tenant", "pk": 1, "fields": {"nombre": "GrabaKar Demo", "color_primario": "#1976D2", "color_secundario": "#424242", "configuracion_json": {}}},
  {"model": "api.leycaso", "pk": 1, "fields": {"nombre": "Ley 20.580", "descripcion": "Grabado obligatorio de patente en vidrios", "activa": true}},
  {"model": "api.usuario", "pk": 1, "fields": {"username": "admin", "password": "pbkdf2_sha256$...", "tenant": 1, "nombre_completo": "Admin Demo", "rol": "admin", "activo": true, "is_staff": true, "is_superuser": true}}
]
```

Use `python manage.py createsuperuser` for the admin, then `python manage.py dumpdata api.usuario --pk 1 > tmp.json` to get the hashed password. Replace in fixture.

### Verification

```bash
python manage.py makemigrations api
python manage.py migrate
python manage.py loaddata initial
python manage.py runserver  # Check admin at /admin/
```

### Tests: `api/tests/conftest.py`

```python
import pytest
from rest_framework.test import APIClient
from api.models import Tenant, Usuario, LeyCaso

@pytest.fixture
def tenant(db):
    return Tenant.objects.create(nombre='Test Tenant', color_primario='#1976D2', color_secundario='#424242')

@pytest.fixture
def otro_tenant(db):
    return Tenant.objects.create(nombre='Otro Tenant')

@pytest.fixture
def ley(db):
    return LeyCaso.objects.create(nombre='Ley 20.580', activa=True)

@pytest.fixture
def usuario_operador(db, tenant):
    return Usuario.objects.create_user(username='operador1', password='test1234', tenant=tenant, nombre_completo='Operador Uno', rol='operador')

@pytest.fixture
def usuario_supervisor(db, tenant):
    return Usuario.objects.create_user(username='supervisor1', password='test1234', tenant=tenant, nombre_completo='Super Uno', rol='supervisor')

@pytest.fixture
def usuario_admin(db, tenant):
    return Usuario.objects.create_user(username='admin1', password='test1234', tenant=tenant, nombre_completo='Admin Uno', rol='admin')

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def auth_client(api_client, usuario_operador):
    api_client.force_authenticate(user=usuario_operador)
    return api_client
```

---

## Authentication (P1-02)

### Custom Token Claims

```python
# api/serializers/auth.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework import serializers

class CustomTokenObtainSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token['tenant_id'] = user.tenant_id
        token['rol'] = user.rol
        token['nombre_completo'] = user.nombre_completo
        return token

class LoginResponseSerializer(serializers.Serializer):
    """Serializes the full login response including tokens + config."""
    access_token = serializers.CharField()
    refresh_token = serializers.CharField()
    offline_token = serializers.CharField()
    token_expiry = serializers.IntegerField()
    offline_token_expiry = serializers.IntegerField()
    usuario = serializers.DictField()
    tenant = serializers.DictField()
    leyes_activas = serializers.ListField()
    vidrios_config = serializers.DictField()
```

### Login View

```python
# api/views/auth.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import AllowAny
from rest_framework_simplejwt.tokens import RefreshToken
from django.contrib.auth import authenticate
from django.conf import settings
from datetime import timedelta
import jwt, time

class LoginView(APIView):
    permission_classes = [AllowAny]

    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        if not username or not password:
            return Response({'error': 'El nombre de usuario y la contraseña son obligatorios.', 'code': 'VALIDATION_ERROR'}, status=400)
        user = authenticate(username=username, password=password)
        if not user or not user.activo:
            return Response({'error': 'Credenciales inválidas', 'code': 'INVALID_CREDENTIALS'}, status=401)

        refresh = RefreshToken.for_user(user)
        refresh['tenant_id'] = user.tenant_id
        refresh['rol'] = user.rol

        # Offline token: 72h, scope=offline
        offline_hours = int(settings.SIMPLE_JWT.get('OFFLINE_TOKEN_LIFETIME_HOURS', 72))
        offline_payload = {
            'user_id': user.id, 'tenant_id': user.tenant_id, 'rol': user.rol,
            'scope': 'offline', 'exp': int(time.time()) + (offline_hours * 3600),
        }
        offline_token = jwt.encode(offline_payload, settings.SECRET_KEY, algorithm='HS256')

        tenant = user.tenant
        leyes = list(LeyCaso.objects.filter(activa=True).values('id', 'nombre', 'descripcion'))
        vidrios_config = tenant.configuracion_json.get('vidrios_config', {
            'auto': [
                {'numero': 1, 'nombre': 'Parabrisas'}, {'numero': 2, 'nombre': 'Luneta trasera'},
                {'numero': 3, 'nombre': 'Lateral delantero izq.'}, {'numero': 4, 'nombre': 'Lateral delantero der.'},
                {'numero': 5, 'nombre': 'Lateral trasero izq.'}, {'numero': 6, 'nombre': 'Lateral trasero der.'},
            ],
            'moto': [{'numero': 1, 'nombre': 'Parabrisas'}],
        })

        return Response({
            'access_token': str(refresh.access_token),
            'refresh_token': str(refresh),
            'offline_token': offline_token,
            'token_expiry': int(settings.SIMPLE_JWT['ACCESS_TOKEN_LIFETIME'].total_seconds()),
            'offline_token_expiry': offline_hours * 3600,
            'usuario': {'id': user.id, 'username': user.username, 'nombre_completo': user.nombre_completo, 'rol': user.rol},
            'tenant': {'id': tenant.id, 'nombre': tenant.nombre, 'logo_url': tenant.logo_url or '', 'color_primario': tenant.color_primario, 'color_secundario': tenant.color_secundario},
            'leyes_activas': leyes,
            'vidrios_config': vidrios_config,
        })
```

### URLs

```python
# api/urls.py
from django.urls import path
from rest_framework_simplejwt.views import TokenRefreshView
from .views.auth import LoginView
from .views.health import health

urlpatterns = [
    path('health/', health),
    path('auth/login/', LoginView.as_view()),
    path('auth/refresh/', TokenRefreshView.as_view()),
]
```

### Tests: `api/tests/test_auth.py`

Test cases (implement each):
1. `test_login_exitoso` — valid credentials → 200, response has all fields
2. `test_login_credenciales_invalidas` — wrong password → 401 with Spanish error
3. `test_login_usuario_inactivo` — `activo=False` → 401
4. `test_login_campos_vacios` — empty body → 400
5. `test_refresh_token` — valid refresh → 200 with new access
6. `test_login_retorna_config_tenant` — response contains tenant branding
7. `test_login_retorna_leyes_activas` — response contains active leyes
8. `test_login_retorna_vidrios_config` — response contains glass config

---

## Tenant Isolation Mixin (Critical)

```python
# api/permissions.py
from rest_framework.permissions import IsAuthenticated

class TenantQuerySetMixin:
    def get_queryset(self):
        qs = super().get_queryset()
        if hasattr(self.request, 'user') and self.request.user.is_authenticated:
            return qs.filter(tenant=self.request.user.tenant)
        return qs.none()
```

Apply to **every** ViewSet and class-based view that accesses tenant data. No exceptions.

Test: `test_usuario_no_ve_datos_de_otro_tenant` — create data in `otro_tenant`, query with authenticated user from first tenant → 0 results.

---

## Grabados CRUD (P1-03)

### Serializer

```python
# api/serializers/grabados.py
from rest_framework import serializers
from api.models import Grabado, ImpresionVidrio
import re

class ImpresionVidrioSerializer(serializers.ModelSerializer):
    class Meta:
        model = ImpresionVidrio
        fields = ['uuid', 'numero_vidrio', 'nombre_vidrio', 'fecha_impresion', 'cantidad_impresiones']

class GrabadoSerializer(serializers.ModelSerializer):
    vidrios = ImpresionVidrioSerializer(source='impresiones', many=True, read_only=True)

    class Meta:
        model = Grabado
        fields = ['uuid', 'patente', 'vin_chasis', 'orden_trabajo', 'usuario_responsable',
                  'ley_caso', 'tipo_movimiento', 'tipo_vehiculo', 'formato_impresion',
                  'fecha_creacion_local', 'fecha_sincronizacion', 'device_id', 'estado_sync',
                  'es_duplicado', 'vidrios']
        read_only_fields = ['fecha_sincronizacion', 'estado_sync']

    def validate_patente(self, value):
        normalized = re.sub(r'[\s\-]', '', value).upper()
        if not re.match(r'^[A-Z0-9]{6,10}$', normalized):
            raise serializers.ValidationError('La patente debe tener entre 6 y 10 caracteres alfanuméricos.')
        return normalized

    def validate_vin_chasis(self, value):
        if value and len(value) != 17:
            raise serializers.ValidationError('El VIN/Chasis debe tener 17 caracteres.')
        return value

class GrabadoCreateSerializer(GrabadoSerializer):
    vidrios = ImpresionVidrioSerializer(many=True, required=False)

    class Meta(GrabadoSerializer.Meta):
        fields = GrabadoSerializer.Meta.fields

    def create(self, validated_data):
        vidrios_data = validated_data.pop('vidrios', [])
        validated_data['tenant'] = self.context['request'].user.tenant
        grabado = Grabado.objects.create(**validated_data)
        for vidrio in vidrios_data:
            ImpresionVidrio.objects.create(grabado=grabado, **vidrio)
        return grabado
```

### ViewSet

```python
# api/views/grabados.py
from rest_framework import viewsets
from api.models import Grabado
from api.serializers.grabados import GrabadoSerializer, GrabadoCreateSerializer
from api.permissions import TenantQuerySetMixin

class GrabadoViewSet(TenantQuerySetMixin, viewsets.ModelViewSet):
    queryset = Grabado.objects.all()
    lookup_field = 'uuid'
    filterset_fields = ['tipo_movimiento', 'estado_sync']

    def get_serializer_class(self):
        if self.action == 'create':
            return GrabadoCreateSerializer
        return GrabadoSerializer

    def get_queryset(self):
        qs = super().get_queryset()
        if self.request.user.rol == 'operador':
            qs = qs.filter(usuario_responsable=self.request.user)
        return qs.order_by('-fecha_creacion_local')
```

### Tests: `api/tests/test_grabados.py`

Test cases:
1. `test_crear_grabado` — POST with valid data → 201
2. `test_crear_grabado_con_vidrios` — nested creation works
3. `test_listar_grabados_operador_solo_propios` — operador sees only their records
4. `test_listar_grabados_supervisor_ve_todos_del_tenant` — supervisor sees all
5. `test_listar_grabados_no_ve_otro_tenant` — tenant isolation
6. `test_validacion_patente_corta` — < 6 chars → 400
7. `test_validacion_vin_longitud` — != 17 chars → 400
8. `test_detalle_grabado` — GET by UUID → 200 with vidrios
9. `test_paginacion` — > 20 results paginated

---

## Sync Upload (P1-04)

### Service

```python
# api/services/sync_service.py
from django.utils import timezone
from api.models import Grabado, ImpresionVidrio

class SyncService:
    @staticmethod
    def procesar_batch(device_id, batch, usuario):
        resultados = []
        for registro in batch:
            try:
                existente = Grabado.objects.filter(uuid=registro['uuid']).first()
                if existente:
                    if registro['fecha_creacion_local'] > str(existente.fecha_creacion_local):
                        SyncService._actualizar_grabado(existente, registro, usuario)
                        resultados.append({'uuid': str(registro['uuid']), 'status': 'ok'})
                    else:
                        resultados.append({'uuid': str(registro['uuid']), 'status': 'ignorado',
                                          'mensaje': 'Registro más reciente ya existe en servidor'})
                else:
                    SyncService._crear_grabado(registro, usuario)
                    resultados.append({'uuid': str(registro['uuid']), 'status': 'ok'})
            except Exception as e:
                resultados.append({'uuid': str(registro.get('uuid', '')), 'status': 'error', 'mensaje': str(e)})
        return {
            'procesados': len(batch),
            'exitosos': sum(1 for r in resultados if r['status'] == 'ok'),
            'fallidos': sum(1 for r in resultados if r['status'] == 'error'),
            'resultados': resultados,
        }

    @staticmethod
    def _crear_grabado(data, usuario):
        impresiones = data.pop('impresiones', [])
        data.pop('tenant_id', None)
        grabado = Grabado.objects.create(
            tenant=usuario.tenant,
            fecha_sincronizacion=timezone.now(),
            estado_sync='sincronizado',
            **{k: v for k, v in data.items() if k != 'impresiones'}
        )
        for imp in impresiones:
            ImpresionVidrio.objects.create(grabado=grabado, **imp)
        return grabado

    @staticmethod
    def _actualizar_grabado(existente, data, usuario):
        update_fields = ['vin_chasis', 'orden_trabajo', 'tipo_movimiento', 'formato_impresion', 'es_duplicado']
        for field in update_fields:
            if field in data:
                setattr(existente, field, data[field])
        existente.fecha_sincronizacion = timezone.now()
        existente.estado_sync = 'sincronizado'
        existente.save()
        # Update impresiones
        if 'impresiones' in data:
            existente.impresiones.all().delete()
            for imp in data['impresiones']:
                ImpresionVidrio.objects.create(grabado=existente, **imp)
```

### View

```python
# api/views/sync.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from api.services.sync_service import SyncService

class SyncUploadView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        batch = request.data.get('batch', request.data.get('registros', []))
        device_id = request.data.get('device_id', '')
        if not batch:
            return Response({'error': 'El batch no puede estar vacío.', 'code': 'VALIDATION_ERROR'}, status=400)
        if len(batch) > 100:
            return Response({'error': 'El batch no puede exceder 100 registros.', 'code': 'VALIDATION_ERROR'}, status=400)
        resultado = SyncService.procesar_batch(device_id, batch, request.user)
        return Response(resultado)
```

### Tests: `api/tests/test_sync.py`

Test cases:
1. `test_batch_valido_crea_grabados` — 3 records → 3 created
2. `test_batch_con_uuid_duplicado_actualiza` — existing UUID with newer date → updated
3. `test_batch_con_uuid_duplicado_ignora` — existing UUID with older date → ignored
4. `test_batch_con_errores_parciales` — 3 records, 1 invalid → 2 ok, 1 error
5. `test_batch_vacio` — → 400
6. `test_batch_excede_limite` — > 100 → 400
7. `test_sin_autenticacion` — → 401
8. `test_tenant_aislamiento_en_sync` — data validated against user's tenant

---

## Config API (P1-05)

```python
# api/views/config.py
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from api.models import LeyCaso

@api_view(['GET'])
@permission_classes([IsAuthenticated])
def tenant_config(request):
    tenant = request.user.tenant
    leyes = list(LeyCaso.objects.filter(activa=True).values('id', 'nombre', 'descripcion'))
    vidrios_config = tenant.configuracion_json.get('vidrios_config', {
        'auto': [
            {'numero': 1, 'nombre': 'Parabrisas'}, {'numero': 2, 'nombre': 'Luneta trasera'},
            {'numero': 3, 'nombre': 'Lateral delantero izq.'}, {'numero': 4, 'nombre': 'Lateral delantero der.'},
            {'numero': 5, 'nombre': 'Lateral trasero izq.'}, {'numero': 6, 'nombre': 'Lateral trasero der.'},
        ],
        'moto': [{'numero': 1, 'nombre': 'Parabrisas'}],
    })
    return Response({
        'tenant': {'id': tenant.id, 'nombre': tenant.nombre, 'logo_url': tenant.logo_url or '',
                   'color_primario': tenant.color_primario, 'color_secundario': tenant.color_secundario},
        'leyes_activas': leyes,
        'vidrios_config': vidrios_config,
    })
```

---

## CI/CD (P0-04)

Create `.github/workflows/ci.yml` — exact content from `tecnico/TESTING.md` CI section (backend.yml).

---

## Custom Exception Handler

```python
# api/utils/exceptions.py
from rest_framework.views import exception_handler

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    if response is not None:
        if response.status_code == 401:
            response.data = {'error': 'No autorizado', 'code': 'UNAUTHORIZED'}
        elif response.status_code == 403:
            response.data = {'error': 'No tienes permiso para esta operación', 'code': 'PERMISSION_DENIED'}
        elif response.status_code == 404:
            response.data = {'error': 'Recurso no encontrado', 'code': 'NOT_FOUND'}
    return response
```

---

## Testing Checklist

Before any PR, verify:

| Check | Command |
|-------|---------|
| Lint | `ruff check .` |
| Tests | `pytest --cov=api --cov-fail-under=80` |
| Migrations | `python manage.py makemigrations --check --dry-run` |
| No raw SQL | `grep -rn "\.raw(" api/ && grep -rn "\.extra(" api/` (should return nothing) |
| No hardcoded brand | `grep -rn '"GrabaKar"' api/` (should return nothing except fixtures) |
| Tenant isolation | Every ViewSet inherits `TenantQuerySetMixin` |

---

## Celery Tasks (P2 — Sync + Reports)

```python
# api/tasks/sync_tasks.py
from celery import shared_task
from django.utils import timezone
from datetime import timedelta

@shared_task
def purgar_registros_antiguos():
    from api.models import Grabado
    limite = timezone.now() - timedelta(days=30)
    deleted, _ = Grabado.objects.filter(
        fecha_creacion_local__lt=limite,
        estado_sync='sincronizado'
    ).delete()
    return f'{deleted} registros purgados'

@shared_task
def generar_reporte_diario(tenant_id, fecha=None):
    from api.services.reporte_service import ReporteService
    return ReporteService.generar_diario(tenant_id, fecha)
```

---

## Parallel Work Rules

- Backend tasks P1-01 through P1-05 have a dependency chain. But P1-03, P1-04, P1-05 can run in parallel after P1-02 is done.
- Backend agent MUST NOT touch frontend files.
- Use API contracts (`tecnico/API_CONTRACTS.md`) as the interface contract with frontend.
- When done, commit with `feat(<scope>): <description in Spanish>` and push to `feature/<fase>-<nombre>`.
