---
## Metadatos
| Campo | Valor |
|-------|--------|
| **Tema / alcance** | Verificación post-implementación: OpenAPI (drf-spectacular) + RBAC panel (`panel_persona`, alcance tenant/sucursal) |
| **Ambiente** | local (Docker Compose) |
| **Commit / tag / build** | N/D (ejecución sobre entorno local actual) |
| **Fecha** | 2026-03-20 |
| **Ejecutor** | QA Agent (backend OpenAPI + API RBAC) |
| **Duración estimada** | ~5-10 min |

---
## 1. Alcance y exclusiones

- **Incluido:**
  - Validación OpenAPI con `python manage.py spectacular --validate --fail-on-warn`.
  - Verificación de carga de endpoints de documentación:
    - `/api/v1/docs/`
    - `/api/v1/redoc/`
    - `/api/v1/schema/`
  - Verificación RBAC del panel vía pruebas automatizadas del backend:
    - `pytest api/tests/test_admin_rbac.py`
- **Excluido (fuera de esta ronda):**
  - Verificación UI del panel web (`grabakar-admin`) vía navegación real (requiere automatización de navegador o pruebas manuales con login/visuals).
  - Ejecución de flujo APK/APK offline por falta de ADB operativo en esta sesión (in-device ya no forma parte de esta verificación RBAC/OpenAPI).

---
## 2. Datos y usuarios de prueba

Los datos de prueba se generan dentro de `pytest api/tests/test_admin_rbac.py` (fixtures que crean usuarios/tenants/sucursales con `panel_persona`).

| Usuario | `panel_persona` | `rol` | `is_staff` | Tenant | Sucursal | Uso previsto |
|---------|-----------------|-------|------------|--------|----------|--------------|
| `platform_user` | `platform_admin` | `ADMIN` | True | Tenant A (fixture) | N/D | Plataforma: lista tenants, KPIs y lectura restringida según matriz |
| `tenant_admin_user` | `tenant_admin` | `ADMIN` | False | Tenant A (fixture) | N/D | Tenant admin: solo ve su tenant; escritura denegada donde corresponda |
| `panel_operator` | `panel_operator_readonly` | `OPERADOR` | False | Tenant A (fixture) | `sucursal_a` | Operador panel readonly: no puede listar/crear donde corresponda |

---
## 3. Plan de pruebas

### 3.1 OpenAPI / Spectacular

| ID | Caso | Prioridad | Pasos resumidos | Resultado | Notas |
|----|------|-----------|-----------------|-----------|-------|
| O-01 | Validación Spectacular | P0 | `docker compose exec django python manage.py spectacular --validate --fail-on-warn` | OK (exit code 0) | Evidencia: [rbac_openapi_spectacular_validate_v2.txt](../../qa_reports/2026-03-16T12-37-33Z/rbac_openapi_spectacular_validate_v2.txt) |
| O-02 | Carga docs/redoc/schema | P0 | `curl /api/v1/docs/`, `/api/v1/redoc/`, `/api/v1/schema/` | OK (HTTP 200 en los 3) | Evidencia: [rbac_openapi_endpoints_v2.txt](../../qa_reports/2026-03-16T12-37-33Z/rbac_openapi_endpoints_v2.txt) |

### 3.2 API admin — RBAC

| ID | Caso | Persona | Endpoint / acción | Esperado HTTP | Resultado | Notas |
|----|------|---------|-------------------|---------------|-----------|-------|
| A-01..A-09 | Conjunto RBAC en `test_admin_rbac.py` | platform_admin/tenant_admin/panel_operator_readonly | Validaciones REST sobre `/api/v1/admin/*` | 200 o 403 según matriz | OK (pytest) | Evidencia: [rbac_openapi_pytest_admin_rbac_v2.txt](../../qa_reports/2026-03-16T12-37-33Z/rbac_openapi_pytest_admin_rbac_v2.txt) |

**Detalle de resumen:** `9 passed, 1 warning in 1.71s`

### 3.3 Panel web (`grabakar-admin`)

| ID | Caso | Persona | Resultado | Notas |
|----|------|---------|-----------|-------|
| U-01..U-04 | Login rechaza `none`, Sidebar, `UsuariosPage` coherente con API | none/platform_admin/tenant_admin/panel_operator_readonly | Bloqueado (sin UI automation en esta ronda) | Ver `TC_RBAC_PANEL_ADMIN_QA.md` para checklist y ejecutar manualmente si aplica |

### 3.4 Regresión (APK / auth común)

| ID | Caso | Resultado | Notas |
|----|------|-----------|-------|
| R-01 | Login APK y flujo básico (operador sin panel) | Bloqueado | Requiere in-device ADB/emulador operativo |

---
## 4. Casos límite y negativos

| ID | Descripción | Entrada / truco | Esperado | Resultado |
|----|-------------|-----------------|----------|-----------|
| B-01 | Operador panel readonly no debe listar tenants | `panel_operator_readonly` en `GET /api/v1/admin/tenants/` | 403 | OK (pytest) |
| B-02 | Operador panel readonly no debe listar usuarios | `panel_operator_readonly` en `GET /api/v1/admin/usuarios/` | 403 | OK (pytest) |
| B-03 | Operador panel readonly no debe hacer POST sucursal | `panel_operator_readonly` en `POST /api/v1/admin/sucursales/` | 403 | OK (pytest) |
| B-04 | Tenant admin no debe crear tenants | `tenant_admin` en `POST /api/v1/admin/tenants/` | 403 | OK (pytest) |
| B-05 | Escritura de leyes limitada (tenant admin no debe crear) | `tenant_admin` en `POST /api/v1/admin/leyes/` | 403 | OK (pytest) |

---
## 5. Ejecución automatizada (si aplica)

| Comando | Repo | Resultado (exit code / resumen) |
|---------|------|----------------------------------|
| `curl http://localhost:8000/api/v1/health/` | grabakar-backend | HTTP 200 |
| `docker compose exec django python manage.py spectacular --validate --fail-on-warn` | grabakar-backend | exit code `0` (OK) |
| `curl http://localhost:8000/api/v1/docs/` | grabakar-backend | HTTP 200 |
| `curl http://localhost:8000/api/v1/redoc/` | grabakar-backend | HTTP 200 |
| `curl http://localhost:8000/api/v1/schema/` | grabakar-backend | HTTP 200 |
| `docker compose exec django pytest api/tests/test_admin_rbac.py -q` | grabakar-backend | `9 passed, 1 warning` (exit code `0`) |

---
## 6. Brechas y seguimiento

| Severidad | Descripción | Archivo / área afectada | Issue / acción |
|-----------|-------------|---------------------------|----------------|
| S3 / Minor | Aviso de DRF pagination (unordered queryset) en test `test_leyes_read_tenant_admin` | `api/tests/test_admin_rbac.py` (y `api.models.LeyCaso`) | Registrar como backlog si no afecta funcionalidad |
| S2 / Major (operativo) | UI del panel web no verificada en esta ronda | `grabakar-admin` (login/Sidebar/UsuariosPage) | Ejecutar manualmente o con automatización y completar `U-01..U-04` |
| S2 / Major (operativo) | In-device APK testing no verificado en esta ronda | emulador / ADB | Completar cuando ADB esté disponible |
| S4 / Trivial (setup) | Si el contenedor `django` falla con `ModuleNotFoundError: drf_spectacular`, reconstruir imágenes (recomendado: `docker compose build django`) | `grabakar-backend` | No es funcionalidad rota verificada en esta ronda; es un requisito operativo para poder ejecutar el QA |

---
## 7. Conclusión

- **Estado:** Aprobado con observaciones (backend OpenAPI + API RBAC verificados; UI/Apk flows pendientes).
- **Condiciones de salida:**
  - OpenAPI validado y endpoints de documentación cargan correctamente.
  - RBAC backend (tenants/sucursales/leyes/usuarios) se comporta según expectativas en pruebas automatizadas.
- **Próximos pasos recomendados:**
  - Completar verificación UI del panel web (`grabakar-admin`) para personas `none/platform_admin/tenant_admin/panel_operator_readonly`.
  - Ejecutar in-device APK suite cuando ADB esté disponible para cubrir TC del APK.
