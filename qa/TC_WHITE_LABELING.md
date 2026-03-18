# TC_WHITE_LABELING — Test Cases: Multi-Tenant Branding / White-Label

**Module**: White-Labeling / Multi-Tenant Branding  
**Feature Docs**: [WHITE_LABELING.md](../features/WHITE_LABELING.md)

---

## Branding Application

### TC-WL-001 — Tenant branding aplicado tras login
**Priority**: P0  
**Preconditions**: Tenant "VidrioPro Chile" has custom `color_primario: "#FF0000"`, `color_secundario: "#00FF00"`, logo_url, nombre  
**Steps**:
1. Login as user from "VidrioPro Chile"
2. Observe Home screen

**Expected**:
- Logo shows "VidrioPro Chile" logo (not GrabaKar default)
- App name shows "VidrioPro Chile"
- Primary buttons use `#FF0000`
- Secondary elements use `#00FF00`
- CSS variables: `--color-primary: #FF0000`, `--color-secondary: #00FF00`

### TC-WL-002 — Branding default sin config cacheada
**Priority**: P0  
**Preconditions**: First-ever app launch, no cached config  
**Steps**: Open app

**Expected**:
- Default logo: `/assets/grabakar-logo.svg`
- Default name: "GrabaKar"
- Default primary: `#1A56DB`
- Default secondary: `#F97316`

### TC-WL-003 — Branding persiste offline
**Priority**: P0  
**Steps**:
1. Login (online) → config cached
2. Close app
3. Open app offline

**Expected**:
- Tenant branding applied from IndexedDB cache
- Colors, logo, name all correct

### TC-WL-004 — CSS variables aplicadas correctamente
**Priority**: P1  
**Steps**: Inspect elements after login

**Expected**:
- `--color-primary` set on `document.documentElement`
- Buttons, headers, accents using `var(--color-primary)`
- Text brand elements using tenant color

### TC-WL-005 — Logo fallback cuando URL falla
**Priority**: P1  
**Preconditions**: Tenant logo_url points to broken/expired URL  
**Steps**: Login and observe header

**Expected**:
- Broken image replaced by default GrabaKar logo
- No broken image icon shown
- `onError` handler triggers gracefully

### TC-WL-006 — Refresh de config manual
**Priority**: P2  
**Steps**:
1. Login (config cached)
2. Admin changes tenant colors on backend
3. Trigger config refresh (`GET /config/`)

**Expected**:
- IndexedDB updated with new config
- CSS variables re-applied immediately
- UI reflects new colors without re-login

---

## Zero Hardcoded Brand

### TC-WL-010 — No strings "GrabaKar" en componentes React
**Priority**: P0  
**Steps**: Grep source code for hardcoded "GrabaKar" strings

**Expected**:
- Zero occurrences in React component JSX/TSX
- App name always from `useAppConfig().appName`

### TC-WL-011 — Nombre del tenant en header
**Priority**: P1  
**Steps**: Login, check header/nav bar

**Expected**:
- Shows tenant name (e.g., "VidrioPro Chile")
- NOT "GrabaKar" (unless default tenant)

---

## Multiple Tenants on Same Device

### TC-WL-020 — Cambio de tenant por login diferente
**Priority**: P1  
**Steps**:
1. Login as user from Tenant A (red branding)
2. Logout
3. Login as user from Tenant B (blue branding)

**Expected**:
- Tenant B branding fully replaces Tenant A
- Colors, logo, name all from Tenant B
- No remnants of Tenant A branding

### TC-WL-021 — Config sobrescrita en mismo dispositivo
**Priority**: P2  
**Steps**: Same as TC-WL-020

**Expected**:
- IndexedDB `tenant_branding` key overwritten
- Previous tenant config not accessible

---

## Validation

### TC-WL-030 — Color hex inválido en config
**Priority**: P2  
**Steps**: Backend has `color_primario: "not-a-color"`

**Expected**:
- CSS variable set (may not display correctly)
- App does not crash
- Fallback to default might apply

### TC-WL-031 — Logo URL vacía
**Priority**: P2  
**Steps**: Tenant has `logo_url: ""`

**Expected**:
- Default logo displayed
- No broken image

### TC-WL-032 — Caracteres especiales en nombre de tenant
**Priority**: P2  
**Steps**: Tenant name: `Vidrio & Patente <Chile>`

**Expected**:
- Properly escaped in HTML
- No XSS vulnerability
- React handles escaping by default

### TC-WL-033 — Config JSON malformada
**Priority**: P2  
**Steps**: `configuracion_json` in IndexedDB is corrupt/invalid

**Expected**:
- Fallback to `{}` (empty config)
- App continues normally

---

## Edge Cases

### TC-WL-040 — Colores con bajo contraste
**Priority**: P3  
**Steps**: Tenant configures `color_primario: "#FFFFFF"` (white on white)

**Expected**:
- App loads (no validation blocking)
- Buttons may be invisible (admin responsibility)
- Known limitation

### TC-WL-041 — Config cambia mientras usuario está offline
**Priority**: P2  
**Steps**:
1. Login, go offline
2. Admin changes branding
3. User remains offline

**Expected**:
- User sees old branding until next login or config refresh
- No interruption to offline workflow
