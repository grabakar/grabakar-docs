# ADMIN_PANEL — Panel de Administración Web

**Fase**: Nueva (post Fase 2)
**Estado**: Implementado (RBAC por `Usuario.panel_persona`)

## Contexto

El sistema opera con Django Admin básico para gestión de datos. A medida que crece la cantidad de tenants (concesionarias), se necesita un panel de administración dedicado con visibilidad cross-tenant, CRUD completo, y analíticas operativas. Este panel es para el equipo GrabaKar (plataforma), no para los clientes individuales.

## Objetivo

Panel web desktop-first donde el equipo de plataforma puede:

- Ver y gestionar todos los tenants (concesionarias), sucursales, usuarios y grabados
- Monitorear salud de sincronización y actividad operativa
- Consultar analíticas simples (volumen de grabados, distribución por tenant, tendencias)
- Crear y configurar nuevos tenants y sucursales sin tocar Django Admin
- Descargar reportes existentes (XLSX plataforma)

## Usuarios Objetivo

| Quién | Acceso | Necesidad |
|-------|--------|-----------|
| Equipo GrabaKar (operaciones) | `panel_persona=platform_admin` (y `is_staff=True`) | Visibilidad completa cross-tenant, gestión de datos |
| Soporte técnico | `panel_persona=platform_admin` (y `is_staff=True`) | Debugging de sync, búsqueda de grabados, estado de dispositivos |

No es para operadores, supervisores ni admins de tenant. Esos roles siguen usando la app móvil/PWA.

## Decisión Arquitectónica

### SPA Separada (`grabakar-admin/`)

El panel vive como una aplicación React independiente, separada de `grabakar-frontend/`.

**Razones:**

1. `grabakar-frontend` es offline-first, mobile-first, empaquetada con Capacitor para Android. Mezclar un dashboard desktop en esa app contamina su arquitectura.
2. El admin panel es online-only y desktop-first — requisitos opuestos al frontend móvil.
3. Separación permite despliegue independiente y desarrollo paralelo.
4. Comparte el mismo backend Django — solo se agregan endpoints bajo `/api/v1/admin/`.

```
grabado-patente-app/
├── grabakar-frontend/    # App móvil PWA (existente)
├── grabakar-backend/     # Django API (existente, se amplía)
├── grabakar-admin/       # Panel admin web (NUEVO)
├── grabakar-docs/        # Documentación (existente)
└── grabakar-infra/       # Infraestructura (existente)
```

### Alternativas descartadas

| Opción | Descartada por |
|--------|---------------|
| Rutas dentro de `grabakar-frontend` | Contamina la app offline-first con lógica online-only desktop |
| Django Admin mejorado (django-unfold) | Limitado en UX custom, gráficos, y flujos de datos complejos |
| Panel aparte con backend propio | Duplica lógica, modelos, y configuración innecesariamente |

## Autenticación y Permisos

### Estrategia: `panel_persona` (RBAC del panel)

El modelo `Usuario` incorpora `panel_persona` (RBAC) para controlar acceso y alcance en `grabakar-admin`.

- `panel_persona in {platform_admin, tenant_admin, panel_operator_readonly}` → acceso al panel (con alcance por tenant/sucursal).
- El login usa el mismo endpoint existente `POST /api/v1/auth/login/`.
- El frontend admin verifica `usuario.panel_persona` (no solo `is_staff`); `is_staff` se usa como compatibilidad/condición para `platform_admin`.
- Los endpoints admin aplican scoping y restricciones según la matriz en `tecnico/RBAC_PANEL_ADMIN.md`.

**No se crea un rol nuevo en `Usuario.Rol`.** Los roles (`operador`, `supervisor`, `admin`) son para lógica de tenant. `panel_persona` es ortogonal — un usuario puede ser `admin` de su tenant y tener `panel_persona=tenant_admin` (o `platform_admin` con `is_staff=True`).

### Respuesta de login ampliada

El endpoint `POST /api/v1/auth/login/` ya retorna el objeto `usuario`. Se agrega `panel_persona` (y `sucursal` cuando aplique) para que el frontend admin aplique RBAC:

```json
{
  "usuario": {
    "id": 1,
    "username": "plataforma_admin",
    "nombre_completo": "Admin GrabaKar",
    "rol": "admin",
    "panel_persona": "platform_admin",
    "is_staff": true
  }
}
```

## Stack Técnico

| Capa | Tecnología | Justificación |
|------|-----------|---------------|
| Framework | React 19 + TypeScript | Mismo stack que frontend, sin curva de aprendizaje nueva |
| Build | Vite 7 | Mismo que frontend |
| UI Components | shadcn/ui + Tailwind CSS v4 | Componentes composables, data tables robustos, dark mode nativo |
| Data Tables | TanStack Table (via shadcn/ui) | Sorting, filtros, paginación, visibilidad de columnas |
| Gráficos | Recharts | React-native, cubre bar/line/pie para las analíticas necesarias |
| Data Fetching | TanStack Query (React Query) | Cache, refetch automático, estados de loading/error |
| HTTP | Axios | Interceptor para JWT, retry, base URL configurable |
| Routing | React Router v7 | Mismo que frontend |
| Formularios | react-hook-form + zod | Mismo patrón que frontend |

## Estructura de Carpetas

```
grabakar-admin/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── api/                    # Cliente HTTP y hooks de TanStack Query
│   │   ├── client.ts           # Instancia Axios con interceptor JWT
│   │   ├── tenants.ts          # useTenantsQuery, useCreateTenant, etc.
│   │   ├── sucursales.ts
│   │   ├── usuarios.ts
│   │   ├── grabados.ts
│   │   └── analytics.ts
│   ├── components/
│   │   ├── layout/             # Sidebar, TopBar, PageContainer
│   │   ├── ui/                 # Componentes shadcn/ui
│   │   ├── tables/             # Configuraciones de columnas por entidad
│   │   └── charts/             # Wrappers de Recharts
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── DashboardPage.tsx
│   │   ├── TenantsPage.tsx
│   │   ├── TenantDetailPage.tsx
│   │   ├── SucursalesPage.tsx
│   │   ├── UsuariosPage.tsx
│   │   ├── GrabadosPage.tsx
│   │   ├── GrabadoDetailPage.tsx
│   │   ├── ReportesPage.tsx
│   │   └── ConfigPage.tsx
│   ├── contexts/
│   │   └── AuthContext.tsx      # JWT auth, panel_persona gate
│   ├── hooks/
│   │   └── useAuth.ts
│   └── types/
│       └── models.ts           # Tipos TS alineados con modelos Django
├── public/
├── index.html
├── package.json
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── .env.local                  # VITE_API_URL=http://localhost:8000
└── .env.production             # VITE_API_URL=https://api.grabakar.cl
```

## Páginas y Funcionalidad

### 1. Login

Pantalla simple: username + password. Al autenticarse, verifica que `usuario.panel_persona` esté habilitado para el panel (y que aplique RBAC). Si no, muestra error: _"Tu cuenta no tiene acceso al panel de administración."_

No hay tokens offline ni lógica de desconexión — el panel es 100% online.

### 2. Dashboard (Home)

**KPI Cards** (fila superior):

| KPI | Fuente | Descripción |
|-----|--------|-------------|
| Total Grabados Hoy | `Grabado.filter(fecha_creacion_local__date=today).count()` | Actividad del día |
| Total Grabados Mes | Idem, filtrado por mes actual | Actividad mensual |
| Tenants Activos | `Tenant.objects.count()` | Total de concesionarias |
| Usuarios Activos | `Usuario.filter(activo=True).count()` | Operadores y supervisores activos |
| Sync Pendientes | `Grabado.filter(estado_sync='pendiente').count()` | Registros sin sincronizar |
| Sync Errores | `Grabado.filter(estado_sync='error').count()` | Registros con error de sync |

**Gráficos:**

- **Grabados últimos 30 días** — Line chart, agrupable por tenant (selector dropdown)
- **Grabados por tenant** — Horizontal bar chart, top 10 tenants por volumen
- **Distribución por tipo_movimiento** — Donut chart (venta / demo / capacitación)
- **Salud de sync** — Stacked bar: sincronizado vs pendiente vs error, por día

### 3. Tenants (Concesionarias)

**Tabla principal:**

| Columna | Fuente |
|---------|--------|
| Nombre | `tenant.nombre` |
| Tipo Cliente | `tenant.tipo_cliente` |
| Sucursales | `COUNT(sucursales)` |
| Usuarios | `COUNT(usuarios)` |
| Grabados (mes) | `COUNT(grabados)` filtrado por mes |
| Colores | Swatch visual de `color_primario` + `color_secundario` |

Acciones: Crear nuevo, Editar, Ver detalle.

**Detalle de Tenant:**

- Formulario editable: nombre, tipo_cliente, logo_url, colores, configuracion_json
- Preview de branding (simulación de cómo se ve la app con esos colores)
- Tabla de sucursales del tenant (CRUD inline)
- Tabla de usuarios del tenant
- Mini-gráfico: grabados del último mes para ese tenant

### 4. Sucursales

**Tabla principal:**

| Columna | Fuente |
|---------|--------|
| Nombre | `sucursal.nombre` |
| Tenant | `sucursal.tenant.nombre` |
| Activa | `sucursal.activa` (badge) |
| Grabados (mes) | `COUNT(grabados)` filtrado |

Filtros: por tenant (dropdown), por estado activa/inactiva.

Acciones: Crear (con selector de tenant), Editar, Toggle activa.

### 5. Usuarios

**Tabla principal:**

| Columna | Fuente |
|---------|--------|
| Nombre | `usuario.nombre_completo` |
| Username | `usuario.username` |
| Rol | `usuario.rol` (badge con color) |
| Tenant | `usuario.tenant.nombre` |
| Sucursal | `usuario.sucursal.nombre` |
| Activo | `usuario.activo` (badge) |
| Último login | `usuario.last_login` |

Filtros: por tenant, por rol, por activo/inactivo.

Acciones: Crear usuario (asignar a tenant y sucursal), Editar, Desactivar.

### 6. Grabados

**Tabla principal:**

| Columna | Fuente |
|---------|--------|
| Patente | `grabado.patente` |
| VIN | `grabado.vin_chasis` |
| Tipo Mov. | `grabado.tipo_movimiento` |
| Tipo Veh. | `grabado.tipo_vehiculo` |
| Operador | `grabado.usuario_responsable.nombre_completo` |
| Tenant | `grabado.tenant.nombre` |
| Sucursal | `grabado.sucursal.nombre` |
| Fecha | `grabado.fecha_creacion_local` |
| Sync | `grabado.estado_sync` (badge con color) |
| Duplicado | `grabado.es_duplicado` (icono) |

Filtros: tenant, sucursal, rango de fechas, estado_sync, tipo_movimiento, búsqueda por patente.

**Detalle de Grabado:** Al hacer click en una fila se abre la vista de detalle con todos los campos + tabla de `ImpresionVidrio` asociadas (nombre_vidrio, impreso, cantidad_impresiones, fecha_impresion).

### 7. Reportes

Interfaz para descargar los reportes existentes:

- Botón para descargar reporte XLSX plataforma (llama a `GET /api/v1/reportes/plataforma/`)
- Selector de fecha y tenant opcional
- Historial de descargas recientes (si se implementa)

### 8. Configuración

- **LeyCaso**: Tabla CRUD de leyes/normativas (nombre, descripción, activa toggle)
- **Configuración global**: Vidrios default por tipo de vehículo, expiración de tokens

## Diseño de UI

### Layout

```
┌──────────────────────────────────────────────────┐
│  TopBar: Logo GrabaKar Admin │ Search │ User ▾   │
├──────────┬───────────────────────────────────────┤
│          │                                       │
│ Sidebar  │  Page Content                         │
│          │                                       │
│ Dashboard│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │
│ Tenants  │  │ KPI │ │ KPI │ │ KPI │ │ KPI │    │
│ Sucursal │  └─────┘ └─────┘ └─────┘ └─────┘    │
│ Usuarios │                                       │
│ Grabados │  ┌───────────────────────────────┐    │
│ Reportes │  │                               │    │
│ Config   │  │        Chart Area              │    │
│          │  │                               │    │
│          │  └───────────────────────────────┘    │
│          │                                       │
│          │  ┌───────────────────────────────┐    │
│          │  │        Data Table              │    │
│          │  │                               │    │
│          │  └───────────────────────────────┘    │
└──────────┴───────────────────────────────────────┘
```

### Principios de UX

- Sidebar colapsable con iconos + labels
- Dark mode toggle (shadcn/ui lo soporta nativamente)
- Tablas con paginación server-side, no carga todo en memoria
- Feedback visual: toasts para acciones exitosas/fallidas
- Formularios con validación inline (react-hook-form + zod)
- Breadcrumbs para navegación jerárquica (Tenants → Detalle → Sucursales)

## Cambios en Backend

### Nuevos archivos

| Archivo | Propósito |
|---------|-----------|
| `api/views/admin.py` | ViewSets para todos los endpoints admin |
| `api/serializers/admin.py` | Serializers con campos cross-tenant y annotaciones de conteo |
| `api/permissions.py` | Permissions RBAC del panel (por `panel_persona`) |
| `api/urls.py` | Registrar rutas `/admin/*` (actualizar existente) |

### Con cambios de modelo

Se agregó `Usuario.panel_persona` (RBAC del panel). `is_staff` se mantiene como condición/compatibilidad para `platform_admin`.

### CORS

Agregar el origen del admin panel a `CORS_ALLOWED_ORIGINS` en settings:

```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:5174",      # grabakar-admin dev (Vite default +1)
    "https://admin.grabakar.cl",  # producción
    # ... orígenes existentes del frontend móvil
]
```

### Serializer de login

Agregar `panel_persona` (y `sucursal` cuando aplique) al objeto `usuario` del login response para que el frontend admin aplique RBAC:

```python
class UsuarioLoginSerializer(serializers.ModelSerializer):
    class Meta:
        model = Usuario
        fields = ['id', 'username', 'nombre_completo', 'rol', 'panel_persona', 'sucursal']
```

## Contratos de API

Ver documento técnico: [ADMIN_PANEL_API.md](../tecnico/ADMIN_PANEL_API.md)

## Plan de Implementación

| Paso | Alcance | Dependencias | Estimación |
|------|---------|-------------|------------|
| 1 | Backend: endpoints admin (views, serializers, permissions, urls) | Ninguna | ~1 día |
| 2 | Scaffold `grabakar-admin/` (Vite + React + Tailwind + shadcn/ui) | Ninguna | ~2 horas |
| 3 | Auth flow: LoginPage, AuthContext, route guards, interceptor JWT | Paso 1, 2 | ~3 horas |
| 4 | Layout: Sidebar, TopBar, PageContainer con React Router | Paso 3 | ~2 horas |
| 5 | DashboardPage: KPIs + gráficos Recharts | Paso 1, 4 | ~4 horas |
| 6 | TenantsPage + TenantDetailPage: tabla, CRUD, detalle | Paso 1, 4 | ~4 horas |
| 7 | SucursalesPage + UsuariosPage: tablas con filtros | Paso 1, 4 | ~4 horas |
| 8 | GrabadosPage + GrabadoDetailPage: tabla cross-tenant, detalle con vidrios | Paso 1, 4 | ~4 horas |
| 9 | ReportesPage: descarga XLSX, selector de fecha/tenant | Paso 4 | ~2 horas |
| 10 | ConfigPage: CRUD de LeyCaso | Paso 1, 4 | ~2 horas |
| 11 | Docker Compose: servir admin panel, actualizar CORS | Paso 2 | ~1 hora |

**Estimación total: ~3-4 días de trabajo enfocado.**

Pasos 1 y 2 son paralelizables. Pasos 5-10 son paralelizables entre sí una vez completados los pasos 1-4.

## Criterio de Aceptación

1. Un usuario con `panel_persona` habilitado puede autenticarse en el admin panel y ver el dashboard (según alcance).
2. Un usuario con `panel_persona=none` recibe error al intentar acceder.
3. El dashboard muestra KPIs actualizados y gráficos de los últimos 30 días.
4. Se pueden crear, editar y ver tenants con todas sus sucursales y usuarios asociados.
5. La tabla de grabados soporta filtros por tenant, fecha, estado de sync y búsqueda por patente.
6. El detalle de un grabado muestra las impresiones de vidrio asociadas.
7. Los reportes XLSX se descargan correctamente desde el panel.
8. Todas las tablas usan paginación server-side.
9. Los endpoints admin retornan 403 para usuarios sin permiso (según `panel_persona` y alcance).
