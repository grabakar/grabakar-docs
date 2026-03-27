# Print Flow Redesign + Grabaplate Rename

> **Valores actuales de ESC/POS (escala por defecto, calibración, tests):** no usar este documento como referencia de bytes — ver **[tecnico/IMPRESION_ESC_POS.md](../../tecnico/IMPRESION_ESC_POS.md)** y `grabakar-frontend/src/utils/escpos.ts`.

After saving a grabado, the user enters a **print screen** where they only need to press "Imprimir" to print the **patente only**. The vidrio-by-vidrio flow is removed. A counter tracks prints, with **no minimum required** before the user can close.

## User Review Required

> [!IMPORTANT]
> The `impresiones_vidrio` DB table and per-vidrio tracking will **no longer be used** in the print flow. Records will still be created (to avoid breaking sync), but the UI won't step through them one by one. The grabado's `cantidad_impresiones` will track the total count. If you want to eliminate `impresiones_vidrio` entirely, let me know.

> [!WARNING]
> The receipt will **only print the patente** — no vidrio name, no UUID, no date. This matches your requirement that only patente is legally mandated. Confirm if you want any other info kept.

## Proposed Changes

### ESC/POS Receipt

#### [MODIFY] [escpos.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/utils/escpos.ts)

- **New function** `buildPatenteReceipt(patente: string)` — replaces [buildCertificadoEscPos](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/utils/escpos.ts#11-44)
- Receipt content: **only the patente**, centered
- Character size: `GS ! 0x44` → **5× width + 5× height** (ajuste para aproximar ~8mm x ~3.5cm en papel 55mm)
- Keep [escposSanitizeLine](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/utils/escpos.ts#5-10) for sanitization

```
[ESC] @ (init)
[ESC] a 1 (center)
[GS] ! 0x44 (5× width, 5× height)
PATENTE\n
[GS] ! 0x00 (reset)
\n\n
[GS] V 0 (cut)
```

---

### Print Service

#### [MODIFY] [PrintService.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/services/PrintService.ts)

- New function `imprimirPatente(patente: string)` replacing [imprimirCertificado](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/services/PrintService.ts#21-41)
- Simplified params — only needs `patente`

---

### Print Page (replaces FlujoMultiVidrioPage)

#### [MODIFY] [FlujoMultiVidrioPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/FlujoMultiVidrioPage.tsx)

Rename/rewrite as **ImpresionPage** — counter-based flow:

- Shows patente in large text
- **Counter**: "Impresiones: N"
- **"Imprimir" button**: sends ESC/POS, increments counter in state + Dexie (`grabado.cantidad_impresiones`)
- **"Cerrar" button**: always visible
  - No minimum print threshold
- On close: mark `grabado.completado = true`, navigate to `/home`

---

### Grabado Form — Navigation Update

#### [MODIFY] [GrabadoFormPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/GrabadoFormPage.tsx)

- After save, navigate to `/impresion/${uuid}` instead of `/flujo-vidrio/${uuid}`
- Button text changes from "Continuar a Grabado de Vidrios" → "Imprimir"

---

### Grabado Detail Page (re-print from history)

#### [NEW] [GrabadoDetailPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/GrabadoDetailPage.tsx)

- Shows grabado details: patente, fecha, tipo movimiento, etc.
- **"Reimprimir" button** → navigates to `/impresion/${uuid}` (same print page, reuses ImpresionPage)
- Back button to home

---

### Grabado Card Update

#### [MODIFY] [GrabadoCard.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/components/GrabadoCard.tsx)

- Completed grabados link to `/grabado/${uuid}` (detail page with re-print)

---

### VidrioStep Removal

#### [DELETE] [VidrioStep.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/components/VidrioStep.tsx)

- No longer needed — the print page is self-contained

---

### Model Update

#### [MODIFY] [models.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/types/models.ts)

- Add `cantidad_impresiones: number` to [Grabado](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/types/models.ts#1-23) interface

---

### Database

#### [MODIFY] [database.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/db/database.ts)

- Ensure `cantidad_impresiones` defaults to `0` on [Grabado](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/types/models.ts#1-23)

---

### Routes

#### [MODIFY] [App.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/App.tsx)

- Add `/impresion/:uuid` → `ImpresionPage`
- Add `/grabado/:uuid` → `GrabadoDetailPage`
- Remove `/flujo-vidrio/:uuid` route (or redirect to `/impresion/:uuid`)

---

### Styles

#### [MODIFY] [index.css](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/index.css)

- Add styles for `.impresion-page`, `.impresion-counter`, `.impresion-actions`
- Add styles for `.grabado-detail` page
- Remove `.flujo-*` and `.vidrio-step` styles (or keep for backward compat)

---

### Rename: Grabakar → Grabaplate

#### [MODIFY] [database.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/db/database.ts)

- `class GrabakarDB` → `class GraplateDB`
- DB name `'grabakar'` → `'grabaplate'`

> [!CAUTION]
> Changing the Dexie DB name will create a **new empty database**. Existing local data would be lost. If that's OK (data syncs from backend), no problem. Otherwise we keep the internal name as-is and only rename UI-facing strings.

#### [MODIFY] [index.html](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/index.html)

- `<title>grabakar-frontend</title>` → `<title>Grabaplate</title>`

> [!NOTE]
> The app name displayed in the header ([AppHeader.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/components/AppHeader.tsx)) comes from `useAppConfig()` which reads `tenant_branding` from the backend. That rename would need to happen on the backend/admin side, not in frontend code.

---

## Verification Plan

### Automated Tests

Run: `npx vitest run` from `c:\Users\jvill\Grabakar\grabakar-infra\repos\grabakar-frontend`

1. **Update** `escpos.test.ts` — test `buildPatenteReceipt` outputs `GS ! 0x44` (5× width + 5× height) and contains only patente text
2. **Rewrite** `FlujoMultiVidrioPage.test.tsx` → test new ImpresionPage:
   - Counter starts at 0
   - Clicking "Imprimir" increments counter
  - "Cerrar" does not enforce a minimum print threshold

### Manual Verification

> Since Bluetooth printing requires a physical Android device and printer, the ESC/POS output format and 9mm height should be verified by you on a real device. The automated tests verify the binary payload structure.
