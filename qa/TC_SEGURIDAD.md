# TC_SEGURIDAD — Test Cases: Seguridad y Control de Acceso

**Module**: Security & Access Control
**Feature Docs**: [SEGURIDAD.md](../tecnico/SEGURIDAD.md), [ROLES_USUARIOS.md](../producto/ROLES_USUARIOS.md)

---

## Role-Based Access Control

### TC-SEC-001 — Operador: acceso a funciones propias
**Priority**: P0
**Steps**: Login as `operador`

**Expected**:
- ✓ Create grabado
- ✓ Multi-vidrio flow
- ✓ View own records
- ✓ Manual sync
- ✗ View other operators' records → hidden/disabled
- ✗ Download reports → hidden/403
- ✗ Manage users → hidden/403
- ✗ Configure tenant → hidden/403

### TC-SEC-002 — Supervisor: acceso ampliado
**Priority**: P0
**Steps**: Login as `supervisor`

**Expected**:
- ✓ All operador permissions
- ✓ View records from ALL operators in tenant
- ✓ Download reports (CSV/JSON)
- ✓ Dashboard
- ✗ Manage users → hidden/403
- ✗ Configure tenant → hidden/403

### TC-SEC-003 — Admin: acceso total
**Priority**: P0
**Steps**: Login as `admin`

**Expected**:
- ✓ All supervisor permissions
- ✓ Manage users (create, deactivate, change role)
- ✓ Configure tenant (branding, logo, colors, name)
- ✓ Configure glass list per vehicle type
- ✓ Configure offline token expiration

---

## Tenant Isolation

### TC-SEC-010 — Operador no ve grabados de otro tenant
**Priority**: P0
**Preconditions**: 2 tenants, records in both
**Steps**: Login as user from Tenant A, list grabados

**Expected**:
- Only Tenant A records returned
- Zero records from Tenant B
- No filtering parameter needed (automatic)

### TC-SEC-011 — Sync no acepta datos de otro tenant
**Priority**: P0
**Steps**: Upload sync batch with `tenant_id` different from user's tenant

**Expected**:
- Rejected with error: _"tenant_id no corresponde al usuario autenticado"_

### TC-SEC-012 — Reportes aislados por tenant
**Priority**: P0
**Steps**: Supervisor downloads report

**Expected**:
- Only their tenant's data in CSV/JSON
- No cross-tenant leakage

### TC-SEC-013 — Config endpoint solo retorna propio tenant
**Priority**: P1
**Steps**: `GET /config/`

**Expected**:
- Only current user's tenant config
- Cannot query other tenant configs

---

## API Security

### TC-SEC-020 — Request sin autenticación → 401
**Priority**: P0
**Steps**: Call any protected endpoint without `Authorization` header

**Expected**: 401 Unauthorized

### TC-SEC-021 — Token inválido → 401
**Priority**: P0
**Steps**: Send request with `Authorization: Bearer invalid_token_xyz`

**Expected**: 401 with `TOKEN_EXPIRED` or invalid token error

### TC-SEC-022 — Token expirado → 401
**Priority**: P1
**Steps**: Use an expired access token

**Expected**: 401 `TOKEN_EXPIRED`

### TC-SEC-023 — CORS: request from unauthorized origin
**Priority**: P1
**Steps**: Make API request from origin not in `CORS_ALLOWED_ORIGINS`

**Expected**: CORS error (request blocked by browser)

---

## Anti-Reprint Fraud

### TC-SEC-030 — Patente no editable después de guardar
**Priority**: P0
**Steps**:
1. Save a grabado
2. Attempt to modify `patente` field

**Expected**:
- Field is read-only
- No way to change patente and reprint on different glass

### TC-SEC-031 — Cada impresión incrementa contador (auditado)
**Priority**: P0
**Steps**: Print 3 times on a glass

**Expected**:
- `cantidad_impresiones: 3`
- Each impression timestamped
- Audit trail preserved

### TC-SEC-032 — No se puede crear grabado con UUID existente
**Priority**: P1
**Steps**: POST grabado with same UUID as existing record

**Expected**: 409 `DUPLICATE_UUID` (direct creation) or conflict resolution in sync

---

## Rate Limiting

### TC-SEC-040 — Login rate limit: 10/min por IP
**Priority**: P1
**Steps**: Send 15 login requests in 1 minute

**Expected**: Requests 11-15 return 429 Too Many Requests

### TC-SEC-041 — Sync rate limit: 60/min por user
**Priority**: P2
**Steps**: Send > 60 sync requests in 1 minute

**Expected**: 429 after 60th request

### TC-SEC-042 — General rate limit: 120/min por user
**Priority**: P2
**Steps**: Send > 120 requests in 1 minute to any endpoint

**Expected**: 429 response

---

## Data Validation (Injection Prevention)

### TC-SEC-050 — SQL injection via patente
**Priority**: P0
**Steps**: Enter patente: `'; DROP TABLE api_grabado;--`

**Expected**:
- Rejected by validation (non-alphanumeric)
- Even if bypassed, ORM parameterizes queries
- No SQL executed

### TC-SEC-051 — XSS via patente field
**Priority**: P0
**Steps**: Enter patente: `<script>alert('xss')</script>`

**Expected**:
- Rejected by validation
- If displayed, React escapes by default
- No script execution

### TC-SEC-052 — XSS via tenant name
**Priority**: P1
**Steps**: Tenant name contains `<img onerror=alert(1)>`

**Expected**:
- React escapes HTML entities
- No XSS executed

---

## Token Security

### TC-SEC-060 — Offline token cifrado en dispositivo
**Priority**: P1
**Steps**: Inspect storage for offline token

**Expected**:
- Token encrypted with AES-GCM
- Not readable in plain text from IndexedDB

### TC-SEC-061 — Logout limpia todos los tokens
**Priority**: P0
**Steps**: Logout, inspect local storage and IndexedDB

**Expected**:
- `access_token`, `refresh_token`, `offline_token` all deleted
- No tokens accessible after logout

### TC-SEC-062 — Refresh token rotation
**Priority**: P1
**Steps**: Use refresh token to get new access token

**Expected**:
- Old refresh token blacklisted
- New refresh token issued
- Cannot reuse old refresh token

---

## HTTPS & Production Security

### TC-SEC-070 — HTTPS redirect in production
**Priority**: P1
**Preconditions**: Production environment
**Steps**: Access app via HTTP

**Expected**: Redirected to HTTPS

### TC-SEC-071 — HSTS header present
**Priority**: P2
**Steps**: Check response headers in production

**Expected**: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`

### TC-SEC-072 — DEBUG=False in production
**Priority**: P0
**Steps**: Trigger a 500 error in production

**Expected**:
- Generic error page (no stack trace)
- No sensitive information leaked
