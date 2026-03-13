# Agente Backend — grabakar-backend

Lee `AGENT_RULES.md` primero. Estas son reglas adicionales específicas para el backend.

> **Implementation Plan:** Read [BACKEND_IMPLEMENTATION.md](implementation_plans/BACKEND_IMPLEMENTATION.md) for exact step-by-step implementation instructions, code templates, file structures, and testing checklists for every backend task.

## Stack

- Django 5+ con Django REST Framework
- PostgreSQL 16+
- Celery + Redis para tareas asíncronas
- Docker Compose para desarrollo local
- Auth: djangorestframework-simplejwt

## Estructura del Proyecto

```
grabakar-backend/
  config/
    settings/
      base.py           # Settings compartidos
      local.py          # Desarrollo local (DEBUG=True, DB local)
      production.py     # Producción (variables de entorno, seguridad)
  api/
    models.py           # Todos los modelos en un archivo
    views/              # Viewsets y vistas, agrupados por dominio
    serializers/        # Serializers DRF
    services/           # Lógica de negocio (NO en views ni serializers)
    utils/              # Funciones auxiliares puras
    tests/              # Tests organizados por dominio
    urls.py
    admin.py
```

Una sola app Django: `api`. No fragmentar en múltiples apps salvo justificación fuerte.

## API

- Versionado por URL: `/api/v1/`
- Serializers DRF con nested writes para relaciones como Grabado → Vidrios.
- ViewSets para CRUD estándar. APIViews para endpoints custom.
- Todos los mensajes de error en español.

## Multi-tenant

- **TODAS las querysets deben filtrar por `request.user.tenant_id`.**
- Nunca exponer datos de un tenant a otro. Esto es requisito de seguridad crítico.
- Usar un mixin o método base en los viewsets para aplicar el filtro automáticamente.
- Validar tenant_id en creación y actualización también.

## Auth

- JWT via simplejwt. Tokens: access (corta vida) + refresh (larga vida).
- El payload del token debe incluir `user_id` y `tenant_id`.
- Endpoints: `/api/v1/auth/login/`, `/api/v1/auth/refresh/`, `/api/v1/auth/logout/`.

## Celery

- Usar para: generación de reportes, procesamiento de sync desde frontend, tareas pesadas.
- Redis como broker y result backend.
- Las tasks van en `api/tasks.py`.

## Docker Compose

Servicios:
- `django` — app principal
- `postgres` — PostgreSQL 16+
- `redis` — broker para Celery

## Testing

- pytest + factory_boy para fixtures.
- Mínimo 80% de cobertura en lógica de negocio (`services/`, `serializers/` con validaciones custom).
- Testear siempre: filtrado multi-tenant, validaciones, permisos, edge cases de negocio.
- Usar `APIClient` de DRF para tests de integración de endpoints.

## Reglas Específicas

- Lógica de negocio va en `services/`, NO en views ni serializers. Views orquestan, services ejecutan.
- No usar `raw()` ni SQL directo. Si es inevitable, parametrizar siempre.
- No hardcodear nombres de marca. Usar modelo Tenant.
- Migrations: una por feature. No generar migrations vacías ni innecesarias.

## Documentación de Referencia

- Contratos API: `grabakar-docs/tecnico/API_CONTRACTS.md`
- Modelo de datos: `grabakar-docs/tecnico/MODELO_DATOS.md`
- Specs de features: `grabakar-docs/fases/`, `grabakar-docs/backlog/`
