# TC_LOGIN_SESION — Test Cases: Login y Gestión de Sesión

**Module**: Authentication & Session Management  
**Feature Docs**: [LOGIN_SESION.md](../features/LOGIN_SESION.md), [SEGURIDAD.md](../tecnico/SEGURIDAD.md)  
**API Contract**: `POST /api/v1/auth/login/`, `POST /api/v1/auth/refresh/`

---

## Happy Path

### TC-LOGIN-001 — Login exitoso con credenciales válidas
**Priority**: P0  
**Preconditions**: Backend running, internet available, user `operador1/secret123` exists  
**Steps**:
1. Open app on emulator
2. Verify login screen is displayed
3. Enter username `operador1`
4. Enter password `secret123`
5. Tap "Iniciar Sesión"

**Expected**:
- Navigates to Home screen within 3 seconds
- User name displayed in header
- Tenant branding (logo, colors) applied
- No error messages shown
- Connection indicator shows green/online

### TC-LOGIN-002 — Login pre-llenado de campos del operador
**Priority**: P1  
**Preconditions**: Logged in successfully once before  
**Steps**:
1. Log out
2. Return to login screen

**Expected**:
- Username field may retain last used value (if remember me enabled)
- Password field is always empty
- Tenant logo shown if cached config exists

### TC-LOGIN-003 — Login carga config de tenant completa
**Priority**: P0  
**Preconditions**: Backend has tenant with custom branding  
**Steps**:
1. Login as user belonging to tenant "VidrioPro Chile"
2. Navigate to Home

**Expected**:
- `tenant_config` cached locally (verify via app state)
- `leyes_activas` cacheadas para auto-asignar ley_caso en nuevo grabado (campo oculto en formulario)
- `vidrios_config` loaded (auto: 6 vidrios, moto: 1)
- CSS variables set to tenant colors

---

## Negative / Validation Tests

### TC-LOGIN-010 — Login con username vacío
**Priority**: P1  
**Steps**:
1. Leave username empty
2. Enter password
3. Tap "Iniciar Sesión"

**Expected**:
- Button is disabled (cannot tap) OR error: _"El nombre de usuario es obligatorio."_
- No network request made

### TC-LOGIN-011 — Login con password vacío
**Priority**: P1  
**Steps**:
1. Enter username
2. Leave password empty
3. Tap "Iniciar Sesión"

**Expected**:
- Button is disabled OR error: _"La contraseña debe tener al menos 6 caracteres."_

### TC-LOGIN-012 — Login con password < 6 caracteres
**Priority**: P2  
**Steps**:
1. Enter username `operador1`
2. Enter password `abc` (3 chars)
3. Tap "Iniciar Sesión"

**Expected**:
- Validation error about minimum password length
- No network request or returns 401

### TC-LOGIN-013 — Login con credenciales inválidas
**Priority**: P0  
**Steps**:
1. Enter username `operador1`
2. Enter password `wrongpassword`
3. Tap "Iniciar Sesión"

**Expected**:
- Error: _"Usuario o contraseña incorrectos."_
- Does NOT reveal whether username or password was wrong
- User remains on login screen
- Password field is cleared

### TC-LOGIN-014 — Login con usuario inexistente
**Priority**: P1  
**Steps**:
1. Enter username `nonexistent_user`
2. Enter password `anypassword123`
3. Tap "Iniciar Sesión"

**Expected**:
- Same generic error: _"Usuario o contraseña incorrectos."_
- Response time similar to valid-user-wrong-password (timing attack prevention)

### TC-LOGIN-015 — Login con usuario inactivo/desactivado
**Priority**: P1  
**Preconditions**: User `desactivado1` exists with `activo=False`  
**Steps**:
1. Login with `desactivado1` credentials

**Expected**:
- Error: _"Tu cuenta ha sido desactivada. Contacta al administrador."_ (or generic 401)

### TC-LOGIN-016 — Login con tenant desactivado
**Priority**: P1  
**Preconditions**: User exists but their tenant has `activo=False`  
**Steps**:
1. Login with user from inactive tenant

**Expected**:
- 403 with message about deactivated account/organization

---

## Offline Login Tests

### TC-LOGIN-020 — Login sin conexión a internet
**Priority**: P0  
**Preconditions**: Device in airplane mode, no cached session  
**Steps**:
1. Enable airplane mode on emulator
2. Open app
3. Enter valid credentials
4. Tap "Iniciar Sesión"

**Expected**:
- Error: _"Se requiere conexión a internet para iniciar sesión."_
- Connection indicator shows red/offline

### TC-LOGIN-021 — Auto-login con offline token válido
**Priority**: P0  
**Preconditions**: User logged in previously, offline token not expired (< 72h)  
**Steps**:
1. Close app
2. Enable airplane mode
3. Reopen app

**Expected**:
- App loads directly to Home (no login screen)
- Cached tenant config applied
- Offline indicator shown
- User can access full local functionality

### TC-LOGIN-022 — Offline token expirado sin conexión
**Priority**: P0  
**Preconditions**: Offline token has expired (> 72h since login)  
**Steps**:
1. (Simulate by advancing device clock > 72h)
2. Enable airplane mode
3. Open app

**Expected**:
- Redirected to login screen
- Message: _"Tu sesión offline ha expirado. Conéctate a internet para iniciar sesión nuevamente."_
- Cannot access any features

### TC-LOGIN-023 — Offline token expirado con conexión disponible
**Priority**: P1  
**Preconditions**: Offline token expired  
**Steps**:
1. Open app with connection available

**Expected**:
- Redirected to login screen
- Can login normally
- New tokens issued and stored

---

## Token Lifecycle Tests

### TC-LOGIN-030 — Access token auto-refresh antes de expirar
**Priority**: P0  
**Preconditions**: Logged in, access token approaching expiry (< 5 min remaining)  
**Steps**:
1. Wait until access token has < 5 min remaining
2. Observe app behavior (background refresh)

**Expected**:
- New access token obtained silently via `POST /auth/refresh/`
- User sees no interruption
- App continues functioning

### TC-LOGIN-031 — Refresh token expirado con offline token válido
**Priority**: P1  
**Steps**:
1. Simulate refresh token expiry (> 7 days)
2. App tries to refresh access token → fails with 401
3. Check offline token status

**Expected**:
- If offline token valid → continue in offline mode
- If offline token also expired → redirect to login

### TC-LOGIN-032 — Refresh token expirado sin offline token
**Priority**: P1  
**Steps**:
1. Simulate both refresh and offline token expiry

**Expected**:
- Redirect to login
- Clear error message

---

## Logout Tests

### TC-LOGIN-040 — Logout estándar
**Priority**: P0  
**Preconditions**: User logged in, some unsynchronized records exist  
**Steps**:
1. Open menu/settings
2. Tap "Cerrar Sesión" / Logout
3. Confirm if prompted

**Expected**:
- Tokens (access, refresh, offline) are deleted from local storage
- Redirected to login screen
- **Unsynchronized records preserved** in IndexedDB (not deleted!)
- Backend invalidates refresh token if online

### TC-LOGIN-041 — Logout preserva registros pendientes de sync
**Priority**: P0  
**Preconditions**: 3 grabados with `estado_sync: "pendiente"` exist  
**Steps**:
1. Logout
2. Login again with same user
3. Check local grabados

**Expected**:
- All 3 pending grabados still present in IndexedDB
- Sync can resume normally

### TC-LOGIN-042 — Logout offline
**Priority**: P2  
**Preconditions**: User offline with valid offline token  
**Steps**:
1. Logout while offline

**Expected**:
- Local tokens deleted
- Backend logout call fails silently (no error shown)
- Redirected to login with offline message

---

## Brute Force / Rate Limiting Tests

### TC-LOGIN-050 — 5 intentos fallidos consecutivos
**Priority**: P1  
**Steps**:
1. Attempt login with wrong password 5 times consecutively

**Expected**:
- After 5th attempt: _"Demasiados intentos fallidos. Intenta nuevamente en X minutos."_
- Login button disabled for lockout period (5 min)

### TC-LOGIN-051 — Backend rate limiting (10/min per IP)
**Priority**: P2  
**Steps**:
1. Send > 10 login requests in under 1 minute

**Expected**:
- 429 response after 10th request
- Appropriate error message displayed

---

## Multi-Tab / Multi-Instance Tests

### TC-LOGIN-060 — Logout sincronizado entre pestañas (PWA)
**Priority**: P3  
**Preconditions**: App open in 2 browser tabs (PWA mode)  
**Steps**:
1. Logout in tab 1

**Expected**:
- Tab 2 also redirected to login (via BroadcastChannel)

---

## Edge Cases

### TC-LOGIN-070 — Password con caracteres especiales
**Priority**: P2  
**Steps**:
1. Login with password containing `áéíóú ñ @ # $ %`

**Expected**:
- Login works if password is correct
- Special characters handled properly in API request

### TC-LOGIN-071 — Cambio de password en otro dispositivo
**Priority**: P2  
**Preconditions**: User logged in on device A, password changed on device B  
**Steps**:
1. On device A, wait for access token to expire
2. Refresh attempt will fail

**Expected**:
- Refresh fails with 401
- Offline token still valid until 72h
- User remains operational offline

### TC-LOGIN-072 — Reloj del dispositivo desincronizado
**Priority**: P3  
**Steps**:
1. Set device clock 48h ahead
2. Check offline token behavior

**Expected**:
- Offline token may appear expired prematurely
- Known limitation documented

### TC-LOGIN-073 — Almacenamiento lleno al guardar tokens
**Priority**: P3  
**Preconditions**: Device storage nearly full  
**Steps**:
1. Attempt login

**Expected**:
- If storage fails: _"Error al guardar sesión. Libera espacio en el dispositivo."_

### TC-LOGIN-074 — Conexión intermitente durante login
**Priority**: P2  
**Steps**:
1. Start login with connection
2. Connection drops mid-request

**Expected**:
- Network error displayed clearly
- User can retry
- No partial state saved

### TC-LOGIN-075 — Password visibility toggle
**Priority**: P2  
**Steps**:
1. Enter password
2. Tap eye/visibility toggle icon

**Expected**:
- Password characters visible when toggled
- Characters hidden when toggled back
- Toggle state does not submit form

### TC-LOGIN-076 — Login button disabled until fields filled
**Priority**: P1  
**Steps**:
1. Open login screen with both fields empty

**Expected**:
- "Iniciar Sesión" button is disabled/grayed out
- Becomes enabled only when both username AND password have values
