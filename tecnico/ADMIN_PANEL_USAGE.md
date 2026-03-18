# Admin Panel — Guía de Uso (Runbook)

Esta guía describe **cómo operar** el panel web de administración de GrabaKar: acceso, flujos principales, y troubleshooting.

- **UI**: SPA React en `grabakar-admin/`
- **API**: Endpoints admin en `/api/v1/admin/*`
- **Permiso**: requiere `is_staff=True` (Django `AbstractUser`)

Docs relacionadas:

- Feature spec: `grabakar-docs/features/ADMIN_PANEL.md`
- Contratos API: `grabakar-docs/tecnico/ADMIN_PANEL_API.md`

---

## 1. Requisitos de Acceso

### 1.1 ¿Quién puede entrar?

Solo usuarios con **`is_staff=True`**.

- Los roles `operador/supervisor/admin` siguen siendo tenant-scoped.
- `is_staff` es **plataforma-scoped** (permite cross-tenant).

### 1.2 Cómo habilitar acceso (local/ops)

La forma recomendada es marcar un usuario como staff desde Django Admin (`/admin/`) o desde shell:

```bash
cd grabakar-backend
.venv/bin/python manage.py shell
```

```python
from api.models import Usuario
u = Usuario.objects.get(username="plataforma_admin")
u.is_staff = True
u.save()
```

---

## 2. Cómo levantar el panel (desarrollo local)

### 2.1 Backend

Debe estar corriendo el backend en `http://localhost:8000` y permitir CORS desde el admin.

En settings, el repo ya contempla `http://localhost:5174` como origin por defecto.

### 2.2 Frontend admin

```bash
cd grabakar-admin
npm install
npm run dev
```

- URL: `http://localhost:5174`
- Config: `.env.local` contiene `VITE_API_URL=http://localhost:8000`

---

## 3. Login / Sesión

### 3.1 Login

- Ir a `/login`
- Ingresar `username` y `password`
- El panel llama a `POST /api/v1/auth/login/`
- Si el usuario **no** tiene `is_staff`, el panel muestra:
  - _“Tu cuenta no tiene acceso al panel de administración.”_

### 3.2 Tokens

El admin panel guarda en `localStorage`:

- `admin_access_token`
- `admin_refresh_token`
- `admin_user`

Nota: el panel es **online-only** (no usa `offline_token`).

### 3.3 Logout

El botón “Cerrar sesión” elimina tokens locales y redirige a `/login`.

---

## 4. Navegación (secciones del sidebar)

El panel tiene estas secciones:

- **Dashboard**
- **Tenants**
- **Sucursales**
- **Usuarios**
- **Grabados**
- **Reportes**
- **Configuración**

---

## 5. Dashboard (operación diaria)

Objetivo: **ver salud de plataforma** en 30–60 segundos.

### 5.1 KPIs

Muestra, entre otros:

- Grabados hoy / mes / total
- Tenants activos
- Usuarios activos
- Sync pendientes / errores
- Tasa de éxito de sync

### 5.2 Gráficos

- Serie temporal “grabados últimos 30 días”
- Distribución por tipo de movimiento
- Ranking “grabados por tenant”

---

## 6. Tenants (concesionarias)

Objetivo: crear y mantener “clientes” (tenants) y su configuración de branding.

### 6.1 Crear tenant

En “Tenants” → “Nuevo tenant”:

- `nombre` (requerido)
- `tipo_cliente` (opcional)
- `color_primario` / `color_secundario` (hex)

El panel llama a:

- `POST /api/v1/admin/tenants/`

### 6.2 Ver tenant (detalle)

Desde la lista → “Ver”:

- Resumen de conteos
- Lista de sucursales del tenant
- Usuarios recientes

El panel llama a:

- `GET /api/v1/admin/tenants/{id}/`

---

## 7. Sucursales

Objetivo: administrar las sedes por tenant.

### 7.1 Filtrar por tenant

En “Sucursales” seleccionar “Todos los tenants” o uno específico.

El panel llama a:

- `GET /api/v1/admin/sucursales/?tenant_id=N`

### 7.2 Crear sucursal

“Nueva sucursal”:

- seleccionar tenant
- `nombre` (requerido)
- `activa` (checkbox)

El panel llama a:

- `POST /api/v1/admin/sucursales/`

---

## 8. Usuarios

Objetivo: gestión cross-tenant de usuarios y acceso staff.

### 8.1 Crear usuario

“Nuevo usuario”:

- `username`, `password` (mínimo 8)
- `nombre_completo`, `email` (opcional)
- `tenant`, `sucursal` (opcional)
- `rol` (operador/supervisor/admin)
- `activo`
- **Acceso admin panel** → `is_staff`

El panel llama a:

- `POST /api/v1/admin/usuarios/`

### 8.2 Filtros

- tenant
- rol

El panel llama a:

- `GET /api/v1/admin/usuarios/?tenant_id=N&rol=operador`

### 8.3 Reset de contraseña

Desde API (si se expone en UI en una iteración siguiente):

- `POST /api/v1/admin/usuarios/{id}/reset_password/`

---

## 9. Grabados (búsqueda y auditoría)

Objetivo: encontrar registros y auditar inconsistencias.

### 9.1 Búsqueda

Filtros en UI:

- tenant
- estado sync (pendiente/sincronizado/error)
- patente (partial)
- rango fechas (desde/hasta)

El panel llama a:

- `GET /api/v1/admin/grabados/` con query params

### 9.2 Detalle de grabado

Incluye:

- Campos principales
- Lista de vidrios (`ImpresionVidrio`)

El panel llama a:

- `GET /api/v1/admin/grabados/{uuid}/`

---

## 10. Reportes (XLSX plataforma)

Objetivo: descargar el reporte global XLSX para operación.

### 10.1 Descargar XLSX plataforma

En “Reportes”:

- Definir `fecha_inicio` y `fecha_fin`
- Click “Descargar XLSX”

El panel llama a:

- `GET /api/v1/reportes/plataforma/?fecha_inicio=YYYY-MM-DD&fecha_fin=YYYY-MM-DD`

Permiso:

- `rol=admin` **o** `is_staff=True` (plataforma)

---

## 11. Configuración (Leyes/Casos)

Objetivo: administrar `LeyCaso` (nombre, descripción, activa).

Acciones:

- Crear ley/caso
- Editar

El panel llama a:

- `GET /api/v1/admin/leyes/`
- `POST /api/v1/admin/leyes/`
- `PUT /api/v1/admin/leyes/{id}/`

---

## 12. Troubleshooting

### 12.1 “No tienes acceso al panel…”

Causa:

- usuario autenticado no tiene `is_staff=True`

Solución:

- activar `is_staff` en el usuario (ver §1.2)

### 12.2 403 en endpoints `/api/v1/admin/*`

Causa:

- falta `Authorization: Bearer …` o no es staff

Solución:

- re-login en el panel
- verificar `is_staff=True`

### 12.3 CORS error en navegador

Causa:

- origin del admin panel no está en `CORS_ALLOWED_ORIGINS`

Solución:

- agregar `http://localhost:5174` (dev) y/o `https://admin.<dominio>` (prod) en `CORS_ALLOWED_ORIGINS`

### 12.4 El panel redirige a `/login` inesperadamente

Causa:

- el backend devolvió 401 (token inválido/expirado)
- el panel limpia tokens y redirige

Solución:

- volver a autenticar
- revisar expiración del JWT y reloj del sistema

---

## 13. Notas de Seguridad Operacional

- **No** otorgar `is_staff` a usuarios finales de concesionarias.
- Usar contraseñas fuertes y rotación para cuentas de plataforma.
- Auditar accesos (Django `last_login`) y limitar exposición pública del admin panel.

