# Frontend Agent — Implementation Plan

Technical implementation guide for all frontend tasks in `grabakar-frontend`. Follow step-by-step.

---

## Repository Setup (P0-02)

### Step 1: Initialize Project

```bash
npx -y create-vite@latest ./ -- --template react-ts
npm install
npm install react-router-dom dexie dexie-react-hooks react-hook-form
npm install -D vitest @testing-library/react @testing-library/jest-dom \
  @testing-library/user-event jsdom fake-indexeddb \
  eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  eslint-plugin-react-hooks prettier
```

### Step 2: Folder Structure

```
src/
├── pages/
│   ├── LoginPage.tsx
│   ├── HomePage.tsx
│   ├── GrabadoFormPage.tsx
│   └── FlujoMultiVidrioPage.tsx
├── components/
│   ├── AppHeader.tsx
│   ├── ConnectivityBadge.tsx
│   ├── SyncIndicator.tsx
│   ├── GrabadoCard.tsx
│   └── VidrioStep.tsx
├── services/
│   ├── AuthService.ts
│   ├── SyncManager.ts
│   ├── ConnectivityService.ts
│   ├── PrintService.ts          # Stub
│   └── ScannerService.ts        # Stub
├── hooks/
│   ├── useAuth.ts
│   ├── useSyncStatus.ts
│   ├── useAppConfig.ts
│   └── useConnectivity.ts
├── types/
│   ├── models.ts                # Grabado, ImpresionVidrio, Usuario, etc.
│   ├── api.ts                   # API request/response types
│   └── config.ts                # TenantBranding, AppConfig
├── utils/
│   ├── validation.ts
│   ├── patente.ts               # normalization, regex
│   └── crypto.ts                # token encryption
├── db/
│   └── database.ts              # Dexie instance + schema
├── contexts/
│   ├── AuthContext.tsx
│   └── ThemeContext.tsx
├── App.tsx
├── main.tsx
└── index.css
```

### Step 3: Vite Config

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test-setup.ts',
  },
});
```

### Step 4: Test Setup

```typescript
// src/test-setup.ts
import '@testing-library/jest-dom';
import 'fake-indexeddb/auto';
```

### Step 5: TypeScript Config

```json
// tsconfig.json — ensure strict mode
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

### Step 6: `.cursorrules`

```
# grabakar-frontend — Agent Context

Read `AGENT_RULES.md` and `FRONTEND_AGENT.md` from grabakar-docs/prompts/ before any work.
Read the relevant feature spec from grabakar-docs/features/ for your task.
Read `FRONTEND_IMPLEMENTATION.md` from grabakar-docs/prompts/implementation_plans/ for exact steps.

## Stack
React 18+ | TypeScript strict | Vite | Dexie.js | React Router | Vitest + RTL

## Key Rules
- All user-facing text in Spanish (Chile).
- NEVER hardcode "GrabaKar". Use `useAppConfig().appName`.
- Types in `src/types/`. No inline type definitions.
- Services never import React components. Pure logic.
- CSS variables for theming. No CSS frameworks.
- Tablet-first responsive: tablet > mobile > desktop.
- Offline-first: every flow must work without internet.
- Tests for all business logic (sync, auth, validations). Min 70% coverage.
- Conventional commits in Spanish.
- Branch naming: `feature/<fase>-<nombre>`

## PDCA
CHECK (read state) > ACT (diagnose) > PLAN (define steps) > DO (implement)

## Docs Location
- Feature specs: grabakar-docs/features/
- API contracts (mock these): grabakar-docs/tecnico/API_CONTRACTS.md
- Data model (for types): grabakar-docs/tecnico/MODELO_DATOS.md
- Implementation plans: grabakar-docs/prompts/implementation_plans/
```

### Verification

```bash
npm run dev    # → localhost:5173 loads
npm run lint   # → 0 errors
npm test       # → 0 tests, 0 errors
npm run build  # → dist/ created
```

---

## Dexie Database Layer (P1-06)

### Database Class

```typescript
// src/db/database.ts
import Dexie, { Table } from 'dexie';
import { Grabado, ImpresionVidrio, Configuracion } from '../types/models';

class GrabakarDB extends Dexie {
  grabados!: Table<Grabado, string>;
  impresiones_vidrio!: Table<ImpresionVidrio, number>;
  configuracion!: Table<Configuracion, string>;

  constructor() {
    super('grabakar');
    this.version(1).stores({
      grabados: 'uuid, patente, fecha_creacion_local, estado_sync, tenant_id',
      impresiones_vidrio: '++id, grabado_uuid, nombre_vidrio',
      configuracion: 'clave',
    });
  }
}

export const db = new GrabakarDB();
```

### Types

```typescript
// src/types/models.ts
export interface Grabado {
  uuid: string;
  patente: string;
  patente_confirmacion: string;
  vin_chasis: string | null;
  orden_trabajo: string | null;
  usuario_responsable_id: number;
  usuario_responsable_nombre: string;
  ley_caso_id: number;
  ley_caso_nombre: string;
  tipo_movimiento: 'venta' | 'demo' | 'capacitacion';
  tipo_vehiculo: 'auto' | 'moto';
  formato_impresion: 'horizontal' | 'vertical';
  es_duplicado: boolean;
  fecha_creacion_local: string;
  estado_sync: 'pendiente' | 'sincronizado' | 'error';
  fecha_sincronizacion: string | null;
  device_id: string;
  tenant_id: number;
  completado: boolean;
  error_sync?: string;
}

export interface ImpresionVidrio {
  id?: number;
  grabado_uuid: string;
  nombre_vidrio: string;
  orden: number;
  cantidad_impresiones: number;
  omitido: boolean;
  fecha_primera_impresion: string | null;
  fecha_ultima_impresion: string | null;
}

export interface Configuracion {
  clave: string;
  valor: unknown;
}

export interface Usuario {
  id: number;
  username: string;
  nombre_completo: string;
  rol: 'operador' | 'supervisor' | 'admin';
  tenant_id: number;
}

export interface TenantBranding {
  nombre: string;
  logo_url: string;
  color_primario: string;
  color_secundario: string;
  configuracion_json: Record<string, unknown>;
  fecha_cache: string;
}
```

### Purge Function

```typescript
// src/db/database.ts (add to same file)
export async function purgarRegistrosAntiguos(): Promise<number> {
  const hace30Dias = new Date();
  hace30Dias.setDate(hace30Dias.getDate() - 30);
  const aPurgar = await db.grabados
    .where('estado_sync').equals('sincronizado')
    .and(r => new Date(r.fecha_creacion_local) < hace30Dias)
    .toArray();
  const uuids = aPurgar.map(g => g.uuid);
  if (uuids.length === 0) return 0;
  await db.transaction('rw', [db.grabados, db.impresiones_vidrio], async () => {
    await db.impresiones_vidrio.where('grabado_uuid').anyOf(uuids).delete();
    await db.grabados.where('uuid').anyOf(uuids).delete();
  });
  return uuids.length;
}
```

### Duplicate Detection

```typescript
// src/db/database.ts (add to same file)
export async function detectarDuplicado(patente: string): Promise<Grabado | undefined> {
  const hace30Dias = new Date();
  hace30Dias.setDate(hace30Dias.getDate() - 30);
  return db.grabados
    .where('patente').equals(patente)
    .and(r => new Date(r.fecha_creacion_local) >= hace30Dias)
    .first();
}
```

### Tests: `src/db/__tests__/database.test.ts`

Test each function:
1. `test_guardar_y_leer_grabado` — CRUD works
2. `test_purga_solo_sincronizados_antiguos` — purge only synced > 30 days
3. `test_purga_no_elimina_no_sincronizados` — pending records preserved
4. `test_detectar_duplicado_existe` — returns match within 30 days
5. `test_detectar_duplicado_antiguo` — > 30 days returns undefined
6. `test_detectar_duplicado_no_existe` — returns undefined
7. `test_transaccion_atomica` — grabado + impresiones saved atomically

---

## Auth Service (P1-07)

### AuthService Class

```typescript
// src/services/AuthService.ts
import { db } from '../db/database';
import { Usuario, TenantBranding } from '../types/models';

const API_BASE = '/api/v1';

class AuthService {
  async login(username: string, password: string): Promise<void> {
    const res = await fetch(`${API_BASE}/auth/login/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password }),
    });
    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error || 'Error de autenticación');
    }
    const data = await res.json();
    // Store tokens
    await db.configuracion.put({ clave: 'access_token', valor: data.access_token });
    await db.configuracion.put({ clave: 'refresh_token', valor: data.refresh_token });
    await db.configuracion.put({ clave: 'offline_token', valor: data.offline_token });
    await db.configuracion.put({ clave: 'token_expiry', valor: Date.now() + (data.token_expiry * 1000) });
    await db.configuracion.put({ clave: 'offline_token_expiry', valor: Date.now() + (data.offline_token_expiry * 1000) });
    // Store user
    await db.configuracion.put({ clave: 'usuario', valor: data.usuario });
    // Store tenant branding
    await db.configuracion.put({
      clave: 'tenant_branding',
      valor: { ...data.tenant, configuracion_json: {}, fecha_cache: new Date().toISOString() },
    });
    // Store leyes and vidrios config
    await db.configuracion.put({ clave: 'leyes_activas', valor: data.leyes_activas });
    await db.configuracion.put({ clave: 'vidrios_config', valor: data.vidrios_config });
  }

  async logout(): Promise<void> {
    // Remove tokens and user, keep unsynced grabados
    await db.configuracion.delete('access_token');
    await db.configuracion.delete('refresh_token');
    await db.configuracion.delete('offline_token');
    await db.configuracion.delete('token_expiry');
    await db.configuracion.delete('offline_token_expiry');
    await db.configuracion.delete('usuario');
    // Keep: tenant_branding, leyes_activas, vidrios_config (for login screen branding)
  }

  async getAccessToken(): Promise<string | null> {
    const entry = await db.configuracion.get('access_token');
    return (entry?.valor as string) ?? null;
  }

  async isSessionValid(): Promise<boolean> {
    const expiryEntry = await db.configuracion.get('token_expiry');
    if (expiryEntry && (expiryEntry.valor as number) > Date.now()) return true;
    // Fallback to offline token
    const offlineExpiry = await db.configuracion.get('offline_token_expiry');
    return offlineExpiry ? (offlineExpiry.valor as number) > Date.now() : false;
  }

  async isOfflineMode(): Promise<boolean> {
    const expiryEntry = await db.configuracion.get('token_expiry');
    const accessValid = expiryEntry && (expiryEntry.valor as number) > Date.now();
    if (accessValid) return false;
    const offlineExpiry = await db.configuracion.get('offline_token_expiry');
    return offlineExpiry ? (offlineExpiry.valor as number) > Date.now() : false;
  }

  async refreshToken(): Promise<boolean> {
    const refreshEntry = await db.configuracion.get('refresh_token');
    if (!refreshEntry) return false;
    try {
      const res = await fetch(`${API_BASE}/auth/refresh/`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refresh: refreshEntry.valor }),
      });
      if (!res.ok) return false;
      const data = await res.json();
      await db.configuracion.put({ clave: 'access_token', valor: data.access });
      await db.configuracion.put({ clave: 'token_expiry', valor: Date.now() + (data.token_expiry ?? 10800) * 1000 });
      return true;
    } catch {
      return false;
    }
  }

  async getStoredUser(): Promise<Usuario | null> {
    const entry = await db.configuracion.get('usuario');
    return (entry?.valor as Usuario) ?? null;
  }
}

export const authService = new AuthService();
```

### AuthContext

```typescript
// src/contexts/AuthContext.tsx
import { createContext, useContext, useState, useEffect, ReactNode, useCallback } from 'react';
import { authService } from '../services/AuthService';
import { Usuario } from '../types/models';

interface AuthState {
  usuario: Usuario | null;
  isAuthenticated: boolean;
  isOffline: boolean;
  loading: boolean;
}

interface AuthContextType extends AuthState {
  login: (username: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState<AuthState>({ usuario: null, isAuthenticated: false, isOffline: false, loading: true });

  useEffect(() => {
    (async () => {
      const valid = await authService.isSessionValid();
      if (valid) {
        const usuario = await authService.getStoredUser();
        const offline = await authService.isOfflineMode();
        setState({ usuario, isAuthenticated: true, isOffline: offline, loading: false });
      } else {
        setState({ usuario: null, isAuthenticated: false, isOffline: false, loading: false });
      }
    })();
    // Auto-refresh every 60s
    const interval = setInterval(async () => {
      const expiryEntry = await (await import('../db/database')).db.configuracion.get('token_expiry');
      if (expiryEntry) {
        const remaining = (expiryEntry.valor as number) - Date.now();
        if (remaining > 0 && remaining < 5 * 60 * 1000 && navigator.onLine) {
          await authService.refreshToken();
        }
      }
    }, 60_000);
    return () => clearInterval(interval);
  }, []);

  const login = useCallback(async (username: string, password: string) => {
    await authService.login(username, password);
    const usuario = await authService.getStoredUser();
    setState({ usuario, isAuthenticated: true, isOffline: false, loading: false });
  }, []);

  const logout = useCallback(async () => {
    await authService.logout();
    setState({ usuario: null, isAuthenticated: false, isOffline: false, loading: false });
  }, []);

  return <AuthContext.Provider value={{ ...state, login, logout }}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

### Tests: `src/services/__tests__/AuthService.test.ts`

1. `test_login_guarda_tokens` — after login, IndexedDB has tokens
2. `test_logout_elimina_tokens_conserva_grabados` — tokens removed, grabados kept
3. `test_sesion_valida_con_access_token` — isSessionValid true
4. `test_sesion_valida_con_offline_token` — fallback works
5. `test_sesion_invalida_ambos_expirados` — returns false
6. `test_refresh_exitoso` — new access token stored
7. `test_refresh_fallido` — returns false

---

## SyncManager (P1-08)

### Implementation

```typescript
// src/services/SyncManager.ts
import { db } from '../db/database';
import { authService } from './AuthService';
import { Grabado } from '../types/models';

export interface SyncStatus {
  pendientes: number;
  sincronizando: boolean;
  ultimaSync: Date | null;
  error: string | null;
}

type SyncListener = (status: SyncStatus) => void;

class SyncManager {
  private static instance: SyncManager;
  private sincronizando = false;
  private intervaloBackoff = 5000;
  private readonly MAX_BACKOFF = 1_800_000;
  private readonly BATCH_SIZE = 100;
  private listeners = new Set<SyncListener>();
  private retryTimeout: ReturnType<typeof setTimeout> | null = null;

  static getInstance(): SyncManager {
    if (!SyncManager.instance) SyncManager.instance = new SyncManager();
    return SyncManager.instance;
  }

  async iniciarSync(): Promise<void> {
    if (this.sincronizando) return;
    this.sincronizando = true;
    this.notificar();
    try {
      let pendientes = await this.obtenerPendientes();
      while (pendientes.length > 0) {
        const batch = pendientes.slice(0, this.BATCH_SIZE);
        await this.enviarBatch(batch);
        pendientes = await this.obtenerPendientes();
      }
      this.intervaloBackoff = 5000;
    } catch {
      this.programarReintento();
    } finally {
      this.sincronizando = false;
      this.notificar();
    }
  }

  private async obtenerPendientes(): Promise<Grabado[]> {
    return db.grabados.where('estado_sync').equals('pendiente').toArray();
  }

  private async enviarBatch(batch: Grabado[]): Promise<void> {
    const token = await authService.getAccessToken();
    const payload = await Promise.all(
      batch.map(async g => ({
        ...g,
        impresiones: await db.impresiones_vidrio.where('grabado_uuid').equals(g.uuid).toArray(),
      }))
    );
    const res = await fetch('/api/v1/sync/upload/', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
      body: JSON.stringify({ registros: payload }),
    });
    if (!res.ok) {
      if (res.status === 401) {
        const refreshed = await authService.refreshToken();
        if (refreshed) return this.enviarBatch(batch);
      }
      throw new Error(`Sync failed: ${res.status}`);
    }
    const resultado = await res.json();
    for (const item of resultado.resultados) {
      if (item.status === 'ok' || item.status === 'ignorado') {
        await db.grabados.update(item.uuid, { estado_sync: 'sincronizado', fecha_sincronizacion: new Date().toISOString() });
      } else if (item.status === 'error') {
        await db.grabados.update(item.uuid, { estado_sync: 'error', error_sync: item.mensaje });
      }
    }
  }

  private programarReintento(): void {
    if (this.retryTimeout) clearTimeout(this.retryTimeout);
    this.retryTimeout = setTimeout(() => this.iniciarSync(), this.intervaloBackoff);
    this.intervaloBackoff = Math.min(this.intervaloBackoff * 3, this.MAX_BACKOFF);
  }

  subscribe(fn: SyncListener): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }

  private async notificar(): Promise<void> {
    const pendientes = await db.grabados.where('estado_sync').equals('pendiente').count();
    const ultimaSyncEntry = await db.configuracion.get('ultima_sync');
    const status: SyncStatus = {
      pendientes,
      sincronizando: this.sincronizando,
      ultimaSync: ultimaSyncEntry ? new Date(ultimaSyncEntry.valor as string) : null,
      error: null,
    };
    this.listeners.forEach(fn => fn(status));
  }

  resetBackoff(): void {
    this.intervaloBackoff = 5000;
  }
}

export const syncManager = SyncManager.getInstance();
```

### Sync Triggers Setup

In `App.tsx` or a top-level component:
```typescript
useEffect(() => {
  // Online event
  const onOnline = () => { syncManager.resetBackoff(); syncManager.iniciarSync(); };
  window.addEventListener('online', onOnline);
  // Periodic (every 5 min)
  const interval = setInterval(() => { if (navigator.onLine) syncManager.iniciarSync(); }, 300_000);
  // Visibility change
  const onVisible = () => { if (document.visibilityState === 'visible' && navigator.onLine) syncManager.iniciarSync(); };
  document.addEventListener('visibilitychange', onVisible);
  return () => { window.removeEventListener('online', onOnline); clearInterval(interval); document.removeEventListener('visibilitychange', onVisible); };
}, []);
```

### Hook

```typescript
// src/hooks/useSyncStatus.ts
import { useState, useEffect } from 'react';
import { syncManager, SyncStatus } from '../services/SyncManager';

export function useSyncStatus(): SyncStatus {
  const [status, setStatus] = useState<SyncStatus>({ pendientes: 0, sincronizando: false, ultimaSync: null, error: null });
  useEffect(() => syncManager.subscribe(setStatus), []);
  return status;
}
```

### Tests: `src/services/__tests__/SyncManager.test.ts`

1. `test_encola_pendientes` — records with 'pendiente' are fetched
2. `test_batch_max_100` — only 100 sent per request
3. `test_marca_sincronizado_tras_exito` — status updated
4. `test_marca_error_en_fallo` — individual errors handled
5. `test_backoff_incrementa` — 5s → 15s → 45s
6. `test_backoff_resetea_en_online` — resets to 5s

---

## Theme Provider (P1-13)

### Implementation

```typescript
// src/contexts/ThemeContext.tsx
import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
import { db } from '../db/database';
import { TenantBranding } from '../types/models';

interface AppConfig {
  appName: string;
  logoUrl: string;
  colorPrimario: string;
  colorSecundario: string;
}

const DEFAULTS: AppConfig = {
  appName: import.meta.env.VITE_APP_NAME || 'GrabaKar',
  logoUrl: '/assets/grabakar-logo.svg',
  colorPrimario: '#1A56DB',
  colorSecundario: '#F97316',
};

const AppConfigContext = createContext<AppConfig>(DEFAULTS);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [config, setConfig] = useState<AppConfig>(DEFAULTS);
  useEffect(() => {
    db.configuracion.get('tenant_branding').then(cached => {
      const b = (cached?.valor as TenantBranding) ?? null;
      const c: AppConfig = b ? { appName: b.nombre, logoUrl: b.logo_url || DEFAULTS.logoUrl, colorPrimario: b.color_primario, colorSecundario: b.color_secundario } : DEFAULTS;
      const root = document.documentElement.style;
      root.setProperty('--color-primary', c.colorPrimario);
      root.setProperty('--color-secondary', c.colorSecundario);
      setConfig(c);
    });
  }, []);
  return <AppConfigContext.Provider value={config}>{children}</AppConfigContext.Provider>;
}

export function useAppConfig(): AppConfig { return useContext(AppConfigContext); }
```

### CSS Variables

```css
/* src/index.css */
:root {
  --color-primary: #1A56DB;
  --color-secondary: #F97316;
  --color-bg: #F9FAFB;
  --color-text: #1F2937;
  --color-error: #DC2626;
  --color-success: #16A34A;
  --color-warning: #D97706;
  --font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
}

* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: var(--font-family); background: var(--color-bg); color: var(--color-text); }

.btn-primary {
  background: var(--color-primary);
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 16px;
  cursor: pointer;
  width: 100%;
}
.btn-secondary {
  background: transparent;
  color: var(--color-primary);
  border: 2px solid var(--color-primary);
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 16px;
  cursor: pointer;
  width: 100%;
}
```

---

## Pages

### Login Page (P1-09)

Key elements:
- Logo from `useAppConfig().logoUrl`, app name from `useAppConfig().appName`
- Username + password fields (password has visibility toggle)
- "Iniciar Sesión" button disabled until both fields filled
- `ConnectivityBadge` in corner (green dot = online, red = offline)
- Error messages in Spanish per `features/LOGIN_SESION.md` validation table
- On submit: call `useAuth().login()`, navigate to `/home` on success
- If offline + valid offline token: auto-redirect to `/home`

### Home Page (P1-10)

Key elements:
- `AppHeader` with tenant logo, app name, `SyncIndicator`, hamburger menu
- List of grabados from IndexedDB, ordered by `fecha_creacion_local` DESC
- Each `GrabadoCard` shows: patente (bold), fecha, tipo_movimiento, sync status icon
- FAB "+" button → navigates to `/grabado/nuevo`
- Menu items: "Cola de sincronización", "Cerrar sesión"
- Resume incomplete grabado: check for `completado: false` on mount

### Grabado Form (P1-11)

Key elements using `react-hook-form`:
- `patente`: input, `autoCapitalize`, transform to uppercase in `onChange`, maxLength 8
- `patente_confirmacion`: input, validate match on `onBlur`, **disable paste**
- `vin_chasis`: optional, validate exactly 17 chars if present
- `orden_trabajo`: optional, max 50 chars
- `usuario_responsable`: read-only, pre-filled from `useAuth().usuario`
- `ley_caso`: select from cached `leyes_activas`
- `tipo_movimiento`: select (Venta, Demo, Capacitación)
- `tipo_vehiculo`: select (Auto, Moto)
- `formato_impresion`: select (Horizontal, Vertical)
- On save: generate UUID v4, save to IndexedDB with `estado_sync: 'pendiente'`, check duplicates first
- Post-save: lock `patente` + `vin_chasis`, change button to "Continuar a Grabado de Vidrios"

### Patente Normalization

```typescript
// src/utils/patente.ts
export function normalizarPatente(value: string): string {
  return value.replace(/[\s\-]/g, '').toUpperCase();
}
export function validarPatente(value: string): string | null {
  const normalized = normalizarPatente(value);
  if (!/^[A-Z0-9]{6,8}$/.test(normalized)) return 'La patente debe tener entre 6 y 8 caracteres alfanuméricos.';
  return null;
}
```

### Flujo Multi-Vidrio (P1-12)

Key elements:
- Receives `grabadoUuid` from navigation
- On mount: load vidrios config from IndexedDB based on `tipo_vehiculo`, create all `ImpresionVidrio` records
- Each step shows:
  - Header: "Vidrio 3 de 6" with progress dots
  - **Patente in large font** (min 48px, monospace, bold)
  - Vidrio name
  - "IMPRIMIR" button (primary, large) — increments `cantidad_impresiones` with 500ms debounce
  - "Omitir este vidrio" link — confirmation dialog
  - "Siguiente Vidrio" / "Finalizar" button
- On "Finalizar": set `Grabado.completado = true`, navigate to Home
- On "Atrás" (first vidrio): confirmation dialog
- Unidirectional flow: no going back to previous vidrio

---

## Service Worker (PWA)

Use `vite-plugin-pwa`:
```bash
npm install -D vite-plugin-pwa
```

Configure in `vite.config.ts`:
```typescript
import { VitePWA } from 'vite-plugin-pwa';

plugins: [
  react(),
  VitePWA({
    registerType: 'autoUpdate',
    workbox: {
      globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
      runtimeCaching: [{
        urlPattern: /^https?:\/\/.*\/api\/v1\/config\//,
        handler: 'NetworkFirst',
        options: { cacheName: 'api-config', expiration: { maxEntries: 1, maxAgeSeconds: 86400 } },
      }],
    },
  }),
],
```

---

## ConnectivityService

```typescript
// src/services/ConnectivityService.ts
type ConnectivityListener = (online: boolean) => void;

class ConnectivityService {
  private online = navigator.onLine;
  private listeners = new Set<ConnectivityListener>();
  private pingInterval: ReturnType<typeof setInterval> | null = null;

  start(): void {
    window.addEventListener('online', () => this.check());
    window.addEventListener('offline', () => this.setOnline(false));
    this.schedulePing();
  }

  private schedulePing(): void {
    if (this.pingInterval) clearInterval(this.pingInterval);
    const interval = this.online ? 30_000 : 60_000;
    this.pingInterval = setInterval(() => this.check(), interval);
  }

  private async check(): Promise<void> {
    try {
      const res = await fetch('/api/v1/health/', { method: 'GET', cache: 'no-store' });
      this.setOnline(res.ok);
    } catch {
      this.setOnline(false);
    }
  }

  private setOnline(value: boolean): void {
    if (this.online !== value) {
      this.online = value;
      this.listeners.forEach(cb => cb(value));
      this.schedulePing();
    }
  }

  isOnline(): boolean { return this.online; }
  onChange(cb: ConnectivityListener): () => void { this.listeners.add(cb); return () => this.listeners.delete(cb); }
}

export const connectivityService = new ConnectivityService();
```

---

## CI/CD (P0-05)

Create `.github/workflows/ci.yml` — exact content from `tecnico/TESTING.md` (frontend.yml).

---

## Testing Checklist

| Check | Command |
|-------|---------|
| Lint | `npm run lint` |
| Tests | `npx vitest run --coverage` |
| Build | `npm run build` |
| No hardcoded brand | `grep -rn '"GrabaKar"' src/ --include="*.tsx" --include="*.ts"` (only in ThemeContext defaults) |
| Types check | `npx tsc --noEmit` |

---

## Parallel Work Rules

- Frontend tasks P1-06, P1-13 can start immediately after P0-02.
- P1-07 depends on P1-06. P1-08 depends on P1-06 + P1-07.
- P1-09 through P1-12 can be partially parallelized (see backlog dependency graph).
- Frontend agent MUST NOT touch backend files.
- Mock ALL API calls using contracts from `tecnico/API_CONTRACTS.md`.
- When done, commit with `feat(<scope>): <description in Spanish>` and push to `feature/<fase>-<nombre>`.
