# RBAC — Panel de administración

**Fuente de negocio:** [producto/PANEL_ADMIN_PERSONAS_Y_PERMISOS.md](../producto/PANEL_ADMIN_PERSONAS_Y_PERMISOS.md)

**Modelo en código (`Usuario.panel_persona`):**

| Valor | Persona |
|-------|---------|
| `none` | Sin acceso al panel web |
| `platform_admin` | Administrador de plataforma GrabaKar (`is_staff=True` obligatorio) |
| `tenant_admin` | Administrador de cliente (un tenant) |
| `panel_operator_readonly` | Operador con panel solo lectura (`rol=operador`, `sucursal` obligatoria) |

El login (`POST /api/v1/auth/login/`) incluye `usuario.panel_persona` y `usuario.sucursal` (si aplica). El panel web (`grabakar-admin`) solo permite sesión si `panel_persona` ≠ `none`.

---

## Matriz endpoint × persona

Leyenda: **✓** permitido · **R** solo lectura (GET/list/retrieve) · **—** 403

| Método | Ruta | `platform_admin` | `tenant_admin` | `panel_operator_readonly` |
|--------|------|------------------|------------------|----------------------------|
| GET | `/api/v1/admin/dashboard/` | ✓ (global) | ✓ (tenant) | ✓ (sucursal) |
| GET | `/api/v1/admin/analytics/grabados/` | ✓ | ✓ | ✓ |
| GET | `/api/v1/admin/analytics/sync/` | ✓ | ✓ | ✓ |
| GET,POST,PATCH,DELETE | `/api/v1/admin/tenants/` | ✓ | R (solo su tenant; POST/DELETE plataforma) | — |
| GET,POST,PATCH,DELETE | `/api/v1/admin/sucursales/` | ✓ | ✓ (tenant) | R |
| GET,POST,PATCH,DELETE | `/api/v1/admin/usuarios/` | ✓ | ✓ (tenant) | — |
| GET | `/api/v1/admin/grabados/` | ✓ | ✓ (tenant) | ✓ (sucursal) |
| GET,POST,PATCH,DELETE | `/api/v1/admin/leyes/` | ✓ | R | R |

**Notas:**

- `tenant_id` en analytics solo aplica a `platform_admin`; los demás perfiles quedan forzados al alcance de su usuario.
- Escritura en leyes (`POST`/`PATCH`/`DELETE` `/admin/leyes/`) solo **platform_admin**.
- Los mensajes de permiso usan el texto de `PanelAdminPermission` / `IsPlatformPanelAdmin` / `DenyPanelOperatorWritePermission` (403).

---

## Filtrado automático (queryset)

Implementación: `api/panel_scope.py` + `get_queryset` en `api/views/admin.py`.

| Recurso | `platform_admin` | `tenant_admin` | `panel_operator_readonly` |
|---------|-------------------|----------------|----------------------------|
| Grabados | todos | `tenant_id` | `tenant_id` + `sucursal_id` |
| Tenants | todos | solo `pk = user.tenant_id` | — (sin acceso) |
| Sucursales | todos | `tenant_id` | solo `pk = user.sucursal_id` |
| Usuarios | todos | `tenant_id` | — (sin acceso) |

---

## UI (`grabakar-admin`)

- **Navegación:** `Sidebar` oculta Tenants/Usuarios a operadores de panel; Reportes y Configuración solo plataforma.
- **Login:** `AuthContext` exige `panel_persona` en `{ platform_admin, tenant_admin, panel_operator_readonly }`.

---

## Pruebas

Regresión en `grabakar-backend/api/tests/test_admin_rbac.py`.

---

## Referencias

- [RBAC_PANEL_ADMIN_DISEÑO.md](RBAC_PANEL_ADMIN_DISEÑO.md)
- [BACKLOG.md](../planificacion/BACKLOG.md) — P3-01, P3-02, P3-08
