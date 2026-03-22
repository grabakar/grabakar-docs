# TC_EDGE_CASES — Test Cases: Cross-Cutting Edge Cases

**Module**: Edge Cases and Boundary Conditions
**Scope**: Scenarios that span multiple modules or test unusual system states

---

## Device & Environment

### TC-EDGE-001 — Rotación de pantalla en cada pantalla
**Priority**: P2
**Steps**: Rotate device 90° on Login, Home, Grabado form, Multi-vidrio, each glass
**Expected**: Layout adapts, no data loss, no crashes

### TC-EDGE-002 — Back button en cada pantalla
**Priority**: P1
**Steps**: Press Android back button on every screen
**Expected**:
- Login: minimize app
- Home: minimize app or nothing
- Grabado form: prompt if unsaved changes
- Multi-vidrio: abandon confirmation
- Reports: back to Home

### TC-EDGE-003 — App killed by OS (memory pressure)
**Priority**: P1
**Steps**: Open many heavy apps to trigger OS kill
**Expected**:
- On reopen, app resumes from last state or Home
- No data loss (IndexedDB is persistent)
- Unfinished grabado recovery prompt

### TC-EDGE-004 — Double-tap on any button
**Priority**: P2
**Steps**: Rapidly double-tap Save, Imprimir, Finalizar, etc.
**Expected**: Action only fires once (debounce/disable after first tap)

### TC-EDGE-005 — App update / reinstall
**Priority**: P2
**Steps**: Install new APK over existing
**Expected**:
- IndexedDB data preserved
- Tokens preserved
- Unsynced records not lost

---

## Data Integrity

### TC-EDGE-010 — Crear el mismo grabado en 2 dispositivos
**Priority**: P1
**Steps**:
1. User logs in on device A and B
2. Both create grabados for same patente BBDF12
3. Both sync

**Expected**:
- Different UUIDs → both accepted by server
- Both records exist with different UUIDs
- No data conflict

### TC-EDGE-011 — UUID collision (theoretical)
**Priority**: P3
**Steps**: (Cannot realistically reproduce)

**Expected**:
- Backend rejects INSERT with existing UUID → 409
- Probability: negligible (2^122 space)

### TC-EDGE-012 — Registro parcial en IndexedDB
**Priority**: P1
**Steps**: Kill app during Dexie transaction

**Expected**:
- Transaction is atomic: either complete or rolled back
- No partial grabado records in IndexedDB

### TC-EDGE-013 — Datos corruptos en IndexedDB
**Priority**: P2
**Steps**: (Simulate by manually corrupting IndexedDB entries)

**Expected**:
- App handles gracefully (try/catch)
- Offers to clear corrupted data
- Does not crash

---

## Network Edge Cases

### TC-EDGE-020 — Conexión WiFi cautiva (captive portal)
**Priority**: P1
**Steps**: Connect to WiFi with captive portal (hotel/coffee shop)

**Expected**:
- `navigator.onLine` might report `true`
- Ping check (`/health/`) fails → correctly reports offline
- No false positive sync attempts

### TC-EDGE-021 — Conexión muy lenta (2G)
**Priority**: P2
**Steps**: Emulate slow network (2G speeds)

**Expected**:
- Sync works but takes longer
- Timeouts handled gracefully
- UI remains responsive
- No partial sync corruption

### TC-EDGE-022 — Conexión se pierde durante login
**Priority**: P1
**Steps**: Start login, then quickly toggle airplane mode

**Expected**:
- Network error displayed
- No partial auth state saved
- Can retry when online again

### TC-EDGE-023 — Header Content-Type inesperado del server
**Priority**: P3
**Steps**: Backend returns HTML instead of JSON (e.g., nginx error page)

**Expected**:
- Error handled gracefully
- No crash on JSON parse
- Appropriate error message

---

## Clock & Time

### TC-EDGE-030 — Reloj del dispositivo en el futuro
**Priority**: P2
**Steps**: Set device clock 48h ahead, create grabado, sync

**Expected**:
- Server may reject if `fecha_creacion_local > now() + 24h`
- Or accept (depends on backend validation)
- Token expiry may behave unexpectedly

### TC-EDGE-031 — Reloj del dispositivo en el pasado
**Priority**: P2
**Steps**: Set device clock 24h behind

**Expected**:
- Records get old timestamps
- Offline token may appear valid longer than expected
- Known limitation

### TC-EDGE-032 — Cambio de zona horaria
**Priority**: P2
**Steps**: Change device timezone during use

**Expected**:
- Existing records keep original timestamp
- ISO 8601 with offset handles this
- No data corruption

---

## Storage Edge Cases

### TC-EDGE-040 — IndexedDB quota exceeded
**Priority**: P1
**Steps**: Fill device storage, then try to save

**Expected**:
- `QuotaExceededError` caught
- Error: _"Almacenamiento lleno. Sincroniza los registros o libera espacio."_
- Purge may free space

### TC-EDGE-041 — localStorage full
**Priority**: P2
**Steps**: Fill localStorage, try to save tokens

**Expected**:
- Error: _"Error al guardar sesión. Libera espacio en el dispositivo."_

### TC-EDGE-042 — Large cantidad of impresiones
**Priority**: P2
**Steps**: Tap "Imprimir" 100 times on one glass

**Expected**:
- `cantidad_impresiones: 100` (with debounce, fewer actual)
- Counter stores correctly
- No overflow or crash

---

## Multi-User / Multi-Tenant

### TC-EDGE-050 — Operador reasignado a otro tenant
**Priority**: P2
**Steps**:
1. User operates under Tenant A
2. Admin moves user to Tenant B
3. User logs in again

**Expected**:
- New login returns Tenant B config
- Old cached Tenant A config overwritten
- Old unsynchronized records from Tenant A — sync attempt may fail (tenant mismatch)

### TC-EDGE-051 — Tenant desactivado mientras operador offline
**Priority**: P2
**Steps**:
1. User goes offline
2. Admin deactivates tenant
3. User works offline for hours
4. User reconnects

**Expected**:
- Offline token valid until expiry
- Sync fails with auth error
- Records stay in `"pendiente"` or `"error"`
- On next login attempt: 403

### TC-EDGE-052 — Ley desactivada entre creación y sync
**Priority**: P2
**Steps**:
1. User creates grabado with Ley A offline
2. Admin deactivates Ley A
3. User syncs

**Expected**:
- Server rejects with: _"ley_caso_id no pertenece al tenant"_ or ley inactive
- Record marked `estado_sync: "error"`

---

## Input Extremes

### TC-EDGE-060 — Patente all zeros: `000000`
**Priority**: P2
**Steps**: Enter patente `000000`

**Expected**: Accepted (6 alphanumeric chars, valid)

### TC-EDGE-061 — Patente all letters: `AAAAAA`
**Priority**: P2
**Steps**: Enter patente `AAAAAA`

**Expected**: Accepted

### TC-EDGE-062 — VIN edge case: all zeros
**Priority**: P2
**Steps**: Enter VIN `00000000000000000` (17 zeros)

**Expected**: Accepted by format validation (17 alphanumeric)

### TC-EDGE-063 — Orden de trabajo with unicode
**Priority**: P3
**Steps**: Enter OT with emojis or Chinese characters

**Expected**:
- Accepted or rejected based on charset validation
- No crash

### TC-EDGE-064 — Empty strings vs null
**Priority**: P2
**Steps**: Submit optional fields as empty string vs not sending field

**Expected**:
- Backend handles both `""` and `null`
- No 500 errors
