# TC_GRABADO_PATENTE — Test Cases: Creación de Registro de Grabado

**Module**: Grabado (Plate Engraving Record)  
**Feature Docs**: [GRABADO_PATENTE.md](../features/GRABADO_PATENTE.md)  
**API Contract**: `POST /api/v1/grabados/`, `POST /api/v1/sync/upload/`

---

## Happy Path

### TC-GRAB-001 — Crear grabado completo (auto, venta)
**Priority**: P0  
**Preconditions**: Logged in as operador, online or offline  
**Steps**:
1. Tap "Nuevo Grabado" on Home
2. Verify `usuario_responsable` is pre-filled and read-only
3. Enter patente: `BBDF12`
4. Enter patente confirmation: `BBDF12`
5. Enter VIN: `1HGCM82633A004352`
6. Enter orden de trabajo: `OT-2026-001`
7. Select tipo movimiento: "Venta"
8. Select tipo vehículo: "Auto"
9. Select formato impresión: "Horizontal"
10. Tap "Guardar"

**Expected**:
- UUID v4 generated for the record
- Record saved in IndexedDB with `estado_sync: "pendiente"`
- Toast: _"Grabado guardado correctamente."_
- Fields `patente` and `vin_chasis` become read-only
- Button changes to "Continuar a Grabado de Vidrios"
- Date field shows current local date/time

### TC-GRAB-002 — Crear grabado mínimo (sin campos opcionales)
**Priority**: P0  
**Steps**:
1. Tap "Nuevo Grabado"
2. Enter patente: `ZZZZ99`
3. Confirm patente: `ZZZZ99`
4. Leave VIN empty
5. Leave orden de trabajo empty
6. Select required dropdowns (tipo movimiento, tipo vehículo, formato)
7. Tap "Guardar"

**Expected**:
- Grabado created successfully
- `vin_chasis: null`, `orden_trabajo: null`
- All other behavior same as TC-GRAB-001

### TC-GRAB-003 — Crear grabado para moto
**Priority**: P1  
**Steps**:
1. Create grabado with `tipo_vehiculo: "Moto"`
2. Tap "Continuar a Grabado de Vidrios"

**Expected**:
- Grabado saved normally
- Multi-vidrio flow shows only 1 glass (Parabrisas)

### TC-GRAB-004 — Crear grabado offline
**Priority**: P0  
**Preconditions**: Device in airplane mode, offline token valid  
**Steps**:
1. Enable airplane mode
2. Create grabado with all fields
3. Tap "Guardar"

**Expected**:
- Grabado saved in IndexedDB
- `estado_sync: "pendiente"`
- No network errors shown
- Full form functionality available

---

## Patente Validation

### TC-GRAB-010 — Patente normalización a mayúsculas
**Priority**: P0  
**Steps**:
1. Enter patente: `bb-df 12`
2. Confirm patente: `bb-df 12`

**Expected**:
- Displayed as `BBDF12` (uppercase, no spaces/hyphens)
- Stored as `BBDF12`

### TC-GRAB-011 — Patente con formato correcto (6 chars)
**Priority**: P1  
**Steps**: Enter patente `ABCD12`, confirm, save

**Expected**: Accepted, saved successfully

### TC-GRAB-012 — Patente con formato correcto (8 chars)
**Priority**: P1  
**Steps**: Enter patente `ABCD1234`, confirm, save

**Expected**: Accepted, saved successfully

### TC-GRAB-013 — Patente demasiado corta (< 6 chars)
**Priority**: P0  
**Steps**: Enter patente `ABC12` (5 chars)

**Expected**:
- Error: _"La patente debe tener entre 6 y 8 caracteres alfanuméricos."_
- "Guardar" button disabled

### TC-GRAB-014 — Patente demasiado larga (> 8 chars)
**Priority**: P1  
**Steps**: Enter patente `ABCDE12345` (10 chars)

**Expected**:
- Input limited to `maxLength=8` OR error on validation
- Cannot save

### TC-GRAB-015 — Patente con caracteres no alfanuméricos
**Priority**: P1  
**Steps**: Enter patente `AB@#12`

**Expected**:
- Error after normalization (removal of hyphens/spaces then regex check)
- Non-alphanumeric chars rejected

### TC-GRAB-016 — Confirmación de patente no coincide
**Priority**: P0  
**Steps**:
1. Enter patente: `BBDF12`
2. Enter confirmation: `BBDF13`

**Expected**:
- Error on blur: _"Las patentes no coinciden."_
- "Guardar" button disabled

### TC-GRAB-017 — Paste deshabilitado en campo confirmación
**Priority**: P2  
**Steps**:
1. Enter patente: `BBDF12`
2. Try to paste `BBDF12` into confirmation field

**Expected**:
- Paste action blocked (Poka-Yoke: force manual re-typing)

### TC-GRAB-018 — Patente solo espacios/guiones
**Priority**: P2  
**Steps**: Enter patente `-- --` (only delimiters)

**Expected**:
- After normalization becomes empty string
- Error: required / too short

---

## VIN Validation

### TC-GRAB-020 — VIN correcto (17 caracteres)
**Priority**: P1  
**Steps**: Enter VIN `1HGCM82633A004352`

**Expected**: Accepted

### TC-GRAB-021 — VIN incorrecto (≠ 17 caracteres)
**Priority**: P1  
**Steps**: Enter VIN `1HGCM826` (8 chars)

**Expected**:
- Error: _"El VIN/Chasis debe tener 17 caracteres alfanuméricos."_

### TC-GRAB-022 — VIN con caracteres especiales
**Priority**: P2  
**Steps**: Enter VIN `1HGCM82633A00@35#`

**Expected**:
- Error (non-alphanumeric characters)

### TC-GRAB-023 — VIN vacío (opcional)
**Priority**: P1  
**Steps**: Leave VIN empty, save

**Expected**: Accepted, `vin_chasis: null`

---

## Duplicate Detection

### TC-GRAB-030 — Duplicado detectado (< 30 días)
**Priority**: P0  
**Preconditions**: Grabado with patente `BBDF12` created 5 days ago in IndexedDB  
**Steps**:
1. Start new grabado
2. Enter patente `BBDF12`, confirm

**Expected**:
- Alert: _"La patente BBDF12 ya fue grabada el DD/MM/YYYY. ¿Deseas continuar de todos modos?"_
- Two buttons: "Cancelar" and "Continuar"

### TC-GRAB-031 — Duplicado aceptado por usuario
**Priority**: P0  
**Preconditions**: TC-GRAB-030 alert shown  
**Steps**: Tap "Continuar"

**Expected**:
- Grabado saved with `es_duplicado: true`
- All other behavior normal

### TC-GRAB-032 — Duplicado cancelado por usuario
**Priority**: P1  
**Steps**: Tap "Cancelar" on duplicate alert

**Expected**:
- Return to form
- Patente field cleared or editable
- No record created

### TC-GRAB-033 — No duplicado (> 30 días)
**Priority**: P1  
**Preconditions**: Grabado with patente `BBDF12` created 35 days ago  
**Steps**: Enter patente `BBDF12`

**Expected**:
- No duplicate alert shown
- `es_duplicado: false`

### TC-GRAB-034 — No duplicado (different patente)
**Priority**: P1  
**Steps**: Enter patente `ZZZZ99` (no previous record)

**Expected**:
- No duplicate alert
- `es_duplicado: false`

### TC-GRAB-035 — Duplicado con normalización
**Priority**: P2  
**Preconditions**: Grabado with patente `BBDF12` exists  
**Steps**: Enter patente `bb-df 12` (different formatting)

**Expected**:
- After normalization to `BBDF12`, duplicate is detected
- Alert shown

---

## Post-Save Behavior

### TC-GRAB-040 — Campos read-only después de guardar
**Priority**: P0  
**Steps**:
1. Save a grabado
2. Try to edit `patente` field
3. Try to edit `vin_chasis` field

**Expected**:
- Both fields are read-only / disabled
- No way to change patente after saving

### TC-GRAB-041 — Botón cambia a "Continuar a Grabado de Vidrios"
**Priority**: P0  
**Steps**: Save a grabado

**Expected**:
- "Guardar" button replaced by "Continuar a Grabado de Vidrios"
- Button uses tenant primary color

### TC-GRAB-042 — Navegación a flujo multi-vidrio
**Priority**: P0  
**Steps**: Tap "Continuar a Grabado de Vidrios"

**Expected**:
- Navigates to multi-vidrio page
- Passes `uuid` of created grabado
- Multi-vidrio loads correct glass list

### TC-GRAB-043 — UUID visible en datos del registro
**Priority**: P2  
**Steps**: After saving, check record details

**Expected**:
- UUID v4 format visible (8-4-4-4-12 hex pattern)

---

## Formulario / UI

### TC-GRAB-050 — Formulario muestra todos los campos requeridos
**Priority**: P1  
**Steps**: Open "Nuevo Grabado" screen

**Expected**:
- Fields visible: patente, confirmación patente, VIN (optional), orden trabajo (optional), tipo movimiento, tipo vehículo, formato impresión
- Ley/Caso no se muestra; se asigna automáticamente con la primera ley de `leyes_activas`
- `usuario_responsable` pre-filled, read-only
- Date shown (auto-filled)

### TC-GRAB-051 — Selectores cargan opciones desde config cacheada
**Priority**: P1  
**Steps**: Open form after login

**Expected**:
- `ley_caso`: no mostrado en UI; valor asignado automáticamente con la primera ley de `leyes_activas` (tenant config)
- `tipo_movimiento`: shows Venta, Demo, Capacitación
- `tipo_vehiculo`: shows Auto, Moto
- `formato_impresion`: shows Horizontal, Vertical

### TC-GRAB-052 — Botón "Guardar" deshabilitado con campos inválidos
**Priority**: P1  
**Steps**: Open form, leave required fields empty

**Expected**:
- "Guardar" button disabled until all required validations pass

### TC-GRAB-053 — Validación en onBlur
**Priority**: P2  
**Steps**:
1. Enter invalid patente `AB`
2. Tab/blur away from field

**Expected**:
- Error appears immediately on blur, not waiting for submit

---

## Edge Cases

### TC-GRAB-060 — App se cierra durante guardado
**Priority**: P1  
**Steps**:
1. Fill all fields
2. Tap "Guardar"
3. Force-close app immediately

**Expected**:
- Dexie transaction guarantees atomicity
- On reopen: either complete record exists, or no partial record

### TC-GRAB-061 — IndexedDB llena
**Priority**: P2  
**Preconditions**: Device storage nearly full  
**Steps**: Try to save a grabado

**Expected**:
- Error: _"Almacenamiento lleno. Sincroniza los registros pendientes o libera espacio."_

### TC-GRAB-062 — Usuario desactivado mientras offline
**Priority**: P2  
**Steps**:
1. Admin deactivates user on backend
2. User (still offline) creates grabados
3. User reconnects

**Expected**:
- Grabados saved locally
- Sync fails with 403
- Records marked `estado_sync: "error"` with reason

### TC-GRAB-063 — Orden de trabajo > 50 caracteres
**Priority**: P2  
**Steps**: Enter 51+ characters in orden de trabajo

**Expected**:
- Error: _"La orden de trabajo no puede exceder 50 caracteres."_

### TC-GRAB-064 — Formulario con orientación landscape/portrait
**Priority**: P3  
**Steps**: Rotate device during form entry

**Expected**:
- All fields remain visible and accessible
- Data not lost on rotation
- Layout adapts gracefully
