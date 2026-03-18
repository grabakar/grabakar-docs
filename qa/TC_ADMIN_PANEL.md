# TC_ADMIN_PANEL — Test Cases: Panel de SuperAdministración

**Module**: Admin API & Platform Management  
**API Contract**: `GET/POST /api/v1/admin/*`
**Roles**: Solo `IsPlatformAdmin` (Usuarios con `is_staff=True` o rol específico superadmin)

---

## 1. Seguridad y Accesos

### TC-ADM-001 — Restricción de acceso para Operadores
**Priority**: P0  
**Steps**:
1. Iniciar sesión en backend con token de usuario rol `operador`.
2. Ejecutar GET `/api/v1/admin/dashboard/`.

**Expected**:
- Response HTTP 403 Forbidden.
- El rol no debe filtrar data inter-tenant.

### TC-ADM-002 — Restricción de acceso para Supervisores
**Priority**: P0  
**Steps**:
1. Iniciar sesión con rol `supervisor`.
2. Ejecutar GET `/api/v1/admin/tenants/`.

**Expected**:
- Response HTTP 403 Forbidden.

### TC-ADM-003 — Acceso exitoso para Platform Admin (Staff)
**Priority**: P0  
**Steps**:
1. Iniciar sesión con usuario `is_staff=True`.
2. Ejecutar GET `/api/v1/admin/dashboard/`.

**Expected**:
- Response 200 OK con métricas globales.

---

## 2. Gestión de Tenants y Sucursales

### TC-ADM-010 — Creación de Tenant con validación de color hex
**Priority**: P1  
**Steps**:
1. POST a `/api/v1/admin/tenants/`.
2. Payload con `color_primario`: `rojo`.

**Expected**:
- HTTP 400 Bad Request.
- Error indicando: _"Formato hex inválido (#RRGGBB)."_

### TC-ADM-011 — Creación exitosa de Tenant
**Priority**: P0  
**Steps**:
1. POST a `/api/v1/admin/tenants/`.
2. Payload válido con logos y colores hex correctos.

**Expected**:
- HTTP 201 Created.
- Retorna el ID del nuevo Tenant con contadores iniciales (sucursales_count=0, usuarios_count=0).

### TC-ADM-012 — Validación cruzada nombre de sucursal única por Tenant
**Priority**: P1  
**Steps**:
1. Crear Sucursal "Norte" en Tenant ID 1.
2. Intentar crear otra Sucursal "Norte" en Tenant ID 1.

**Expected**:
- HTTP 400 Bad Request.
- Error indicando que ya existe una sucursal con ese nombre en dicho Tenant.

---

## 3. Gestión de Usuarios (ABM)

### TC-ADM-020 — Creación de usuario vinculando Sucursal que NO pertenece al Tenant
**Priority**: P0  
**Steps**:
1. Recuperar Tenant ID 1 y Sucursal ID 99 (perteneciente a Tenant 2).
2. POST `/api/v1/admin/usuarios/` asignando `tenant_id=1` y `sucursal_id=99`.

**Expected**:
- HTTP 400 Bad Request.
- Valiation Error: _"La sucursal no pertenece al tenant indicado."_

### TC-ADM-021 — Creación exitosa de usuario operador
**Priority**: P0  
**Steps**:
1. POST `/api/v1/admin/usuarios/` con clave y parámetros consistentes.

**Expected**:
- HTTP 201 Created.
- Contraseña es hasheada automáticamente (no expuesta en response).

### TC-ADM-022 — Password Reset Administrativo
**Priority**: P1  
**Steps**:
1. POST `/api/v1/admin/usuarios/{id}/reset_password/`.
2. Payload: `{"new_password": "super_secure_pass_1"}`.

**Expected**:
- HTTP 200 OK: _"Contraseña actualizada exitosamente."_
- Login subsiguiente con la nueva clave funciona.

---

## 4. Dashboards y Analíticas

### TC-ADM-030 — Filtro transversal de Grabados (Sin leak multitenant en admin)
**Priority**: P1  
**Steps**:
1. GET `/api/v1/admin/grabados/`.

**Expected**:
- HTTP 200 OK.
- La paginación devuelve grabados de múltiples tenants mezclados, probando la capacidad de auditoría global del SuperAdmin.

### TC-ADM-031 — Analytics de Sincronización por Fecha y Tenant
**Priority**: P1  
**Steps**:
1. GET `/api/v1/admin/analytics/sync/?dias=15&tenant_id=1`.

**Expected**:
- Payload contiene métricas agrupadas (`resumen` y `serie` temporal) _únicamente_ para el Tenant especificado, dentro del margen de 15 días traseros.

---

## 5. Auditoría de Integridad

### TC-ADM-040 — Visualización anidada de Impresión de Vidrios
**Priority**: P2  
**Steps**:
1. Obtener UUID de un grabado existente con impresiones.
2. GET `/api/v1/admin/grabados/{uuid}/`.

**Expected**:
- La respuesta HTTP 200 debe incluir el array anidado `vidrios` con el desglose de `nombre_vidrio` y `cantidad_impresiones`, tal cual está mapeado en el iterador de `ImpresionVidrioAdminSerializer`.
