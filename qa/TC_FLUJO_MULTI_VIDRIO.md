# TC_FLUJO_MULTI_VIDRIO — Test Cases: Flujo Iterativo de Grabado por Vidrio

**Module**: Multi-Glass Poka-Yoke Flow
**Feature Docs**: [FLUJO_MULTI_VIDRIO.md](../features/FLUJO_MULTI_VIDRIO.md)

---

## Happy Path

### TC-MV-001 — Flujo completo auto (6 vidrios)
**Priority**: P0
**Preconditions**: Grabado created with `tipo_vehiculo: "auto"`, tapped "Continuar a Grabado de Vidrios"
**Steps**:
1. Verify screen shows "Vidrio 1 de 6" with glass name "Parabrisas"
2. Verify patente displayed in large format (≥ 48px, monospace, bold)
3. Tap "Imprimir" → verify toast: _"Impresión registrada (1)"_
4. Tap "Siguiente Vidrio"
5. Repeat for vidrios 2-5 (Luneta, Delantera Izq, Delantera Der, Trasera Izq)
6. On vidrio 6 (Trasera Derecha), verify button says "Finalizar" instead of "Siguiente Vidrio"
7. Tap "Imprimir" then "Finalizar"

**Expected**:
- Progress: "Vidrio N de 6" updates correctly at each step
- Progress dots update (● ● ● ○ ○ ○)
- Toast: _"Grabado finalizado correctamente."_
- Navigates to Home
- `Grabado.completado = true` in IndexedDB
- 6 `ImpresionVidrio` records created, each with `cantidad_impresiones: 1`

### TC-MV-002 — Flujo completo moto (1 vidrio)
**Priority**: P0
**Preconditions**: Grabado with `tipo_vehiculo: "moto"`
**Steps**:
1. Enter multi-vidrio flow
2. Verify "Vidrio 1 de 1" with name "Parabrisas"
3. Verify button says "Finalizar" (not "Siguiente Vidrio")
4. Tap "Imprimir" then "Finalizar"

**Expected**:
- Single glass flow
- 1 `ImpresionVidrio` record created
- Grabado marked complete

### TC-MV-003 — Flujo offline
**Priority**: P0
**Preconditions**: Device offline
**Steps**: Complete entire multi-vidrio flow without internet

**Expected**:
- All records saved in IndexedDB
- No errors related to connectivity
- Full functionality available

---

## Print Button Behavior

### TC-MV-010 — Imprimir incrementa contador
**Priority**: P0
**Steps**:
1. On any glass screen, tap "Imprimir" 3 times

**Expected**:
- First tap: toast _"Impresión registrada (1)"_
- Second tap: toast _"Impresión registrada (2)"_
- Third tap: toast _"Impresión registrada (3)"_
- `ImpresionVidrio.cantidad_impresiones = 3`

### TC-MV-011 — Reimprimir vidrio (múltiples impresiones)
**Priority**: P1
**Steps**:
1. Tap "Imprimir" twice on same glass

**Expected**:
- Counter increments each time
- `fecha_primera_impresion` set on first tap only
- `fecha_ultima_impresion` updates on each tap
- `omitido` remains `false`

### TC-MV-012 — Debounce de doble-tap (500ms)
**Priority**: P1
**Steps**:
1. Rapidly double-tap "Imprimir" button (< 200ms apart)

**Expected**:
- Only 1 impression registered (debounce prevents accidental double)
- Toast shows "(1)" not "(2)"

### TC-MV-013 — Imprimir sin Bluetooth (Fase 1)
**Priority**: P1
**Steps**: Tap "Imprimir"

**Expected**:
- Event registered (counter incremented)
- No Bluetooth connection attempted
- No error about missing printer

---

## Skip Glass Behavior

### TC-MV-020 — Omitir vidrio con confirmación
**Priority**: P0
**Steps**:
1. On glass "Luneta", tap "Omitir este vidrio"
2. Confirm in dialog: _"¿Omitir el grabado de Luneta? Se registrará como no impreso."_

**Expected**:
- `ImpresionVidrio` for Luneta: `omitido: true`, `cantidad_impresiones: 0`
- Advances to next glass automatically

### TC-MV-021 — Cancelar omisión de vidrio
**Priority**: P1
**Steps**:
1. Tap "Omitir este vidrio"
2. Tap "Cancelar" in confirmation dialog

**Expected**:
- Stays on current glass
- No changes to the record

### TC-MV-022 — Omitir todos los vidrios
**Priority**: P1
**Steps**: Skip all 6 glasses without printing any

**Expected**:
- Puede finalizar sin bloqueo por mínimo de impresiones
- `Grabado.completado = true`

### TC-MV-023 — Omitir algunos, imprimir otros
**Priority**: P1
**Steps**:
1. Print on glasses 1, 3, 5
2. Skip glasses 2, 4, 6
3. Finalize

**Expected**:
- Finalizes successfully (sin mínimo de impresiones)
- Correct `omitido` and `cantidad_impresiones` per glass

---

## Navigation

### TC-MV-030 — Botón "Atrás" en primer vidrio
**Priority**: P1
**Steps**:
1. On vidrio 1, tap back button (or Android hardware back)
2. Confirmation: _"¿Abandonar el flujo de grabado? Los vidrios ya registrados se conservarán."_
3. Tap "Abandonar"

**Expected**:
- Returns to previous screen (grabado form or Home)
- Any `ImpresionVidrio` records already created are preserved

### TC-MV-031 — Cancelar abandono del flujo
**Priority**: P1
**Steps**: Tap back, then "Cancelar"

**Expected**:
- Stays on current glass in flow
- No data lost

### TC-MV-032 — Android hardware back button
**Priority**: P1
**Steps**: Press hardware back button on emulator (← key)

**Expected**:
- Same behavior as in-app back button
- Shows abandon confirmation

### TC-MV-033 — No se puede retroceder a vidrio anterior
**Priority**: P1
**Steps**:
1. Print on vidrio 1, go to vidrio 2
2. Try to go back to vidrio 1

**Expected**:
- Cannot navigate backward in the flow (unidirectional)
- Back button shows abandon confirmation, not previous glass

---

## Progress Indicator

### TC-MV-040 — Indicador de progreso correcto
**Priority**: P1
**Steps**: Navigate through all 6 glasses

**Expected**:
- Header updates: "Vidrio 1 de 6" → "Vidrio 2 de 6" → ... → "Vidrio 6 de 6"
- Progress dots: ● fills as you advance

### TC-MV-041 — Botón "Finalizar" solo en último vidrio
**Priority**: P0
**Steps**: Navigate to the last glass

**Expected**:
- Button text is "Finalizar" (not "Siguiente Vidrio")
- All previous glasses show "Siguiente Vidrio"

---

## Patente Display (Poka-Yoke)

### TC-MV-050 — Patente en formato grande
**Priority**: P0
**Steps**: Enter multi-vidrio flow

**Expected**:
- Patente displayed in at least 48px font
- Monospace font
- Bold weight
- Highly legible (high contrast against background)

### TC-MV-051 — Patente correcta en todos los vidrios
**Priority**: P0
**Steps**: Navigate through all glasses

**Expected**:
- Same patente shown on every glass screen
- Patente matches the one entered in grabado form

### TC-MV-052 — Patente legible en landscape y portrait
**Priority**: P2
**Steps**: Rotate device on a glass screen

**Expected**:
- Patente remains fully visible and legible
- Layout adapts gracefully

---

## Edge Cases

### TC-MV-060 — App se cierra en medio del flujo
**Priority**: P1
**Steps**:
1. Print on 3 glasses
2. Force-close app
3. Reopen app

**Expected**:
- Prompt: _"Tienes un grabado en progreso para la patente BBDF12. ¿Deseas continuar?"_
- If yes: resume at first glass without impression
- If no: go to Home (unfinished grabado preserved)

### TC-MV-061 — Config de vidrios cambia entre login y grabado
**Priority**: P2
**Preconditions**: Login cached config has auto=6 glasses, then config updated on server to auto=7
**Steps**: Start grabado flow

**Expected**:
- Uses config from login time (6 glasses)
- Does NOT recalculate vidrios mid-flow

### TC-MV-062 — Tipo vehículo con 0 vidrios configurados
**Priority**: P2
**Preconditions**: Config has 0 glasses for vehicle type
**Steps**: Start multi-vidrio flow

**Expected**:
- Error: _"No hay vidrios configurados para este tipo de vehículo. Contacta al administrador."_

### TC-MV-063 — Impresiones rápidas concurrentes
**Priority**: P2
**Steps**: Tap "Imprimir" as fast as possible 10 times

**Expected**:
- With debounce, approximately 5 impressions registered (500ms debounce)
- No crashes
- Counter consistent
