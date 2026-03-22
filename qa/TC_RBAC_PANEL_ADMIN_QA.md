# TC — RBAC panel (QA checklist — post‑implementación)

**Propósito:** Lista de verificación para el agente/humano de QA alineada a `tecnico/RBAC_PANEL_ADMIN.md` y al código en `grabakar-backend` / `grabakar-admin`.
**Registro de ejecución:** completar un archivo `REGISTRO_QA_<YYYY>_RBAC_OPENAPI.md` usando [REGISTRO_QA_PLANTILLA.md](REGISTRO_QA_PLANTILLA.md).

**Personas (`Usuario.panel_persona`):** `none` · `platform_admin` · `tenant_admin` · `panel_operator_readonly`

---

## Auth y sesión panel

| ID | Caso | Esperado |
|----|------|----------|
| RBAC-QA-01 | Login panel con usuario `panel_persona=none` | Acceso denegado en UI (mensaje claro) |
| RBAC-QA-02 | Login con `platform_admin` | Entrada al panel; menú completo según Sidebar |
| RBAC-QA-03 | Login con `tenant_admin` | Entrada; sin Reportes/Config si solo plataforma |
| RBAC-QA-04 | Login con `panel_operator_readonly` | Entrada; sin Tenants ni Usuarios en nav |
| RBAC-QA-05 | Login APK (operador `none`) sigue devolviendo tokens | 200; `usuario.panel_persona` presente |

---

## API admin (por persona)

| ID | Caso | Esperado |
|----|------|----------|
| RBAC-QA-10 | `GET /admin/dashboard/` cada persona | KPIs acotados según matriz RBAC |
| RBAC-QA-11 | `tenant_admin` lista `/admin/tenants/` | Solo su tenant |
| RBAC-QA-12 | `panel_operator_readonly` lista `/admin/tenants/` | 403 |
| RBAC-QA-13 | `panel_operator_readonly` `POST /admin/sucursales/` | 403 |
| RBAC-QA-14 | `tenant_admin` `POST /admin/tenants/` | 403 |
| RBAC-QA-15 | `tenant_admin` `POST /admin/leyes/` | 403 |
| RBAC-QA-16 | `tenant_admin` `GET /admin/leyes/` | 200 listado |
| RBAC-QA-17 | Operador APK (`none`) `GET /admin/dashboard/` | 403 |

---

## UI (`grabakar-admin`)

| ID | Caso | Esperado |
|----|------|----------|
| RBAC-QA-20 | Sidebar según persona | Coherente con prompt de QA |
| RBAC-QA-21 | Crear usuario como plataforma | Checkbox staff / persona coherente |
| RBAC-QA-22 | Crear usuario como tenant admin | Selector de persona panel |

---

## OpenAPI

| ID | Caso | Esperado |
|----|------|----------|
| RBAC-QA-30 | `spectacular --validate --fail-on-warn` en CI/local | Exit 0 |
| RBAC-QA-31 | Swagger carga y lista rutas `/api/v1/*` | Sin error de carga |

---

*Los IDs son orientativos; el registro final vive en `REGISTRO_QA_*.md`.*
