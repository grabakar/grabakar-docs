# TC_USABILIDAD — Test Cases: Usability & UX

**Module**: Usability, Accessibility, and User Experience

---

## Navigation & Layout

### TC-UX-001 — Navegación intuitiva desde Home
**Priority**: P1  
**Steps**: Login, observe Home screen

**Expected**:
- Clear "Nuevo Grabado" button (primary CTA)
- List of recent grabados visible
- Sync indicator visible
- User info / logout accessible
- Connection status visible

### TC-UX-002 — Acceso con una mano en tablet
**Priority**: P2  
**Steps**: Use app with one hand on tablet

**Expected**:
- Primary actions (save, print, next) reachable in bottom half of screen
- No critical actions only in top corners

### TC-UX-003 — Responsive landscape / portrait
**Priority**: P1  
**Steps**: Rotate device during use

**Expected**:
- Layout adapts smoothly
- No cropped content
- No loss of state on rotation
- Patente large text remains legible in both orientations

### TC-UX-004 — Touch targets ≥ 48x48dp
**Priority**: P2  
**Steps**: Check all tappable elements

**Expected**:
- All buttons, links, and interactive elements have minimum 48x48dp touch area
- Follows Android accessibility guidelines

---

## Form UX

### TC-UX-010 — Formulario de grabado: flujo natural
**Priority**: P1  
**Steps**: Fill grabado form top-to-bottom

**Expected**:
- Fields in logical order
- Keyboard auto-focus moves to next field
- Keyboard type matches field (text, numeric, etc.)
- Validation errors appear near their field

### TC-UX-011 — Error messages claros y en español
**Priority**: P0  
**Steps**: Trigger various validation errors

**Expected**:
- All error messages in Spanish
- Messages are actionable (tell user what to fix)
- Messages appear near the problematic field
- No technical jargon

### TC-UX-012 — Toast notifications visible y temporales
**Priority**: P2  
**Steps**: Save a grabado, print on a glass

**Expected**:
- Toast appears for 3-5 seconds
- Doesn't block interaction
- Readable text size and contrast

### TC-UX-013 — Loading states for all actions
**Priority**: P1  
**Steps**: Save grabado, login, sync

**Expected**:
- Loading indicator (spinner/progress) shown during async operations
- Buttons disabled during loading to prevent double-submit
- No "dead" periods with no visual feedback

---

## Offline Experience

### TC-UX-020 — Indicador de conectividad siempre visible
**Priority**: P0  
**Steps**: Observe app in both online and offline states

**Expected**:
- Green badge = online
- Red badge = offline
- Visible on every screen (not just Home)
- Transition is immediate (< 3 seconds)

### TC-UX-021 — Mensajes offline claros
**Priority**: P1  
**Steps**: Try online-only actions while offline

**Expected**:
- Clear Spanish messages explaining limitation
- Suggest reconnecting or provide alternative
- No cryptic error codes

### TC-UX-022 — No confusión sobre estado de registros
**Priority**: P1  
**Steps**: Create grabados offline, check list

**Expected**:
- Pending records clearly marked (icon/badge "pendiente")
- Synced records clearly marked
- Error records clearly marked with reason

---

## Poka-Yoke (Error Prevention)

### TC-UX-030 — Doble digitación de patente
**Priority**: P0  
**Steps**: Enter patente, then must re-enter to confirm

**Expected**:
- Two separate fields: "Patente" and "Confirmar Patente"
- Cannot save until they match
- Paste disabled on confirmation field

### TC-UX-031 — Patente grande en multi-vidrio
**Priority**: P0  
**Steps**: Enter multi-vidrio flow

**Expected**:
- Patente shown in huge text (≥ 48px)
- Monospace, bold font
- Easy to verify at arm's length
- Visible before each print action

### TC-UX-032 — Confirmaciones para acciones destructivas
**Priority**: P1  
**Steps**: Try to skip a glass, abandon flow, logout

**Expected**:
- Confirmation dialog for all irreversible actions
- Dialog clearly states consequences
- "Cancelar" option always available

---

## Accessibility

### TC-UX-040 — Contraste de texto suficiente
**Priority**: P2  
**Steps**: Check text against backgrounds

**Expected**:
- Minimum 4.5:1 contrast ratio for normal text
- Minimum 3:1 for large text
- Tenant colors may not meet this (known limitation)

### TC-UX-041 — Font sizes readable
**Priority**: P1  
**Steps**: Check all text on a real device/emulator

**Expected**:
- Body text ≥ 14sp
- Labels ≥ 12sp
- No text too small to read on tablet

### TC-UX-042 — Screen reader compatible (basic)
**Priority**: P3  
**Steps**: Enable TalkBack on emulator

**Expected**:
- All buttons have accessible labels
- Form fields have labels
- Images have alt text

---

## Feedback & Confirmation

### TC-UX-050 — Confirmación visual al guardar grabado
**Priority**: P0  
**Steps**: Save a grabado

**Expected**:
- Toast: _"Grabado guardado correctamente."_
- Visual change (button text changes)
- Clear path forward ("Continuar a Grabado de Vidrios")

### TC-UX-051 — Confirmación visual al finalizar flujo
**Priority**: P0  
**Steps**: Finalize multi-vidrio flow

**Expected**:
- Toast: _"Grabado finalizado correctamente."_
- Returns to Home
- New record visible in list

### TC-UX-052 — Error feedback no bloqueante
**Priority**: P1  
**Steps**: Trigger a sync error

**Expected**:
- Error shown but doesn't block app use
- User can dismiss and continue working
- Error details available on tap
