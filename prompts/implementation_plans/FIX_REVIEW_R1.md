# Fix Implementation Plan — Review Issues (R1)

Implementation guide for fixing all issues from the first code review. Follow step-by-step. Each fix includes the exact file, what to change, and how to verify.

> **Pre-requisite:** Read `AGENT_RULES.md` first. Follow PDCA. All user-facing text in Spanish (Chile).

---

## Fix C1: Vidrios sin UUID en sync payload (🔴 CRÍTICO)

**Problema:** `SyncManager.ts` envía vidrios sin `uuid`. Backend `ImpresionVidrio` tiene `uuid` como PK con `editable=False` → crash en `create()`.

**Archivos a modificar:**
- `grabakar-frontend/src/services/SyncManager.ts`
- `grabakar-frontend/src/pages/GrabadoFormPage.tsx`

### Paso 1: Generar UUID al crear ImpresionVidrio

```typescript
// src/pages/GrabadoFormPage.tsx
// En el loop donde se crean impresiones (línea ~93-103):
// ANTES:
await db.impresiones_vidrio.add({
  grabado_uuid: uuid,
  nombre_vidrio: vidrios[i].nombre,
  orden: vidrios[i].numero,
  // ...
});

// DESPUÉS: agregar campo vidrio_uuid
await db.impresiones_vidrio.add({
  vidrio_uuid: crypto.randomUUID(),  // ← NUEVO
  grabado_uuid: uuid,
  nombre_vidrio: vidrios[i].nombre,
  orden: vidrios[i].numero,
  cantidad_impresiones: 0,
  omitido: false,
  fecha_primera_impresion: null,
  fecha_ultima_impresion: null,
});
```

### Paso 2: Agregar campo `vidrio_uuid` al tipo ImpresionVidrio

```typescript
// src/types/models.ts — agregar a ImpresionVidrio:
export interface ImpresionVidrio {
  id?: number;
  vidrio_uuid: string;    // ← NUEVO: UUID para sync con backend
  grabado_uuid: string;
  // ... resto igual
}
```

### Paso 3: Agregar índice en Dexie

```typescript
// src/db/database.ts — actualizar schema:
this.version(2).stores({
  grabados: 'uuid, patente, fecha_creacion_local, estado_sync, tenant_id',
  impresiones_vidrio: '++id, grabado_uuid, nombre_vidrio, vidrio_uuid',  // ← agregar vidrio_uuid
  configuracion: 'clave',
});
```

### Paso 4: Enviar UUID en sync payload

```typescript
// src/services/SyncManager.ts — en enviarBatch(), en el map de vidrios (~línea 70-75):
vidrios: await db.impresiones_vidrio.where('grabado_uuid').equals(g.uuid).toArray().then(imp => imp.map(i => ({
  uuid: i.vidrio_uuid,       // ← NUEVO
  numero_vidrio: i.orden,
  nombre_vidrio: i.nombre_vidrio,
  fecha_impresion: i.fecha_ultima_impresion || undefined,
  cantidad_impresiones: i.cantidad_impresiones,
}))),
```

### Verificación C1
```bash
# Buscar que el payload incluye uuid:
grep -n 'uuid.*vidrio_uuid\|vidrio_uuid' src/services/SyncManager.ts src/pages/GrabadoFormPage.tsx src/types/models.ts
# Debe haber matches en los 3 archivos.
```

---

## Fix C2: DEVICE_ID efímero (🔴 CRÍTICO)

**Problema:** `DEVICE_ID` se genera con `Math.random()` fuera del componente. Se pierde al recargar la página.

**Archivo:** `grabakar-frontend/src/pages/GrabadoFormPage.tsx`

### Paso 1: Crear helper para device ID persistente

```typescript
// src/utils/deviceId.ts (NUEVO ARCHIVO)
import { db } from '../db/database';

let cachedDeviceId: string | null = null;

export async function getDeviceId(): Promise<string> {
  if (cachedDeviceId) return cachedDeviceId;
  const entry = await db.configuracion.get('device_id');
  if (entry) {
    cachedDeviceId = entry.valor as string;
    return cachedDeviceId;
  }
  const id = `device-${crypto.randomUUID()}`;
  await db.configuracion.put({ clave: 'device_id', valor: id });
  cachedDeviceId = id;
  return id;
}
```

### Paso 2: Usar en GrabadoFormPage

```typescript
// src/pages/GrabadoFormPage.tsx
// ELIMINAR línea: const DEVICE_ID = 'device-' + Math.random().toString(36).slice(2, 12);
// EN onSubmit, antes de crear el grabado:
import { getDeviceId } from '../utils/deviceId';

const onSubmit = async (data: FormData) => {
  // ...
  const deviceId = await getDeviceId();  // ← reemplaza DEVICE_ID
  const grabado: Grabado = {
    // ...
    device_id: deviceId,  // ← usar variable
    // ...
  };
};
```

### Verificación C2
```bash
grep -rn 'Math.random' src/pages/GrabadoFormPage.tsx
# Debe retornar 0 matches.
grep -rn 'getDeviceId' src/pages/GrabadoFormPage.tsx src/utils/deviceId.ts
# Debe retornar matches en ambos archivos.
```

---

## Fix F1: Flujo multi-vidrio bidireccional (🟡 FUNCIONAL)

**Problema:** `FlujoMultiVidrioPage.tsx` línea 64 permite `← Anterior`. Spec dice "Unidirectional flow: no going back to previous vidrio."

**Archivo:** `grabakar-frontend/src/pages/FlujoMultiVidrioPage.tsx`

### Cambio

```tsx
// ANTES (línea 64):
<button type="button" className="btn-link" onClick={() => idx === 0 ? (window.confirm('¿Salir del flujo?') && navigate('/home')) : setIdx(i => i - 1)}>
  {idx === 0 ? 'Atrás' : '← Anterior'}
</button>

// DESPUÉS:
<button type="button" className="btn-link" onClick={() => window.confirm('¿Salir del flujo? El progreso de vidrios ya registrados se mantendrá.') && navigate('/home')}>
  Salir
</button>
```

Eliminar toda lógica de `setIdx(i => i - 1)`. El botón siempre sale del flujo con confirmación.

### Verificación F1
```bash
grep -n 'i - 1\|Anterior' src/pages/FlujoMultiVidrioPage.tsx
# Debe retornar 0 matches.
```

---

## Fix F2: "Offline" en inglés (🟡 FUNCIONAL)

**Archivo:** `grabakar-frontend/src/components/ConnectivityBadge.tsx`

### Cambio

```tsx
// Línea 9, ANTES:
{online ? 'En línea' : 'Offline'}

// DESPUÉS:
{online ? 'En línea' : 'Sin conexión'}
```

### Verificación F2
```bash
grep -rni 'offline' src/components/ConnectivityBadge.tsx
# Solo debe aparecer en className, no en texto visible.
```

---

## Fix F3: PWA / Service Worker (🟡 FUNCIONAL)

### Paso 1: Instalar dependencia
```bash
cd grabakar-frontend
npm install -D vite-plugin-pwa
```

### Paso 2: Configurar en vite.config.ts

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
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
  server: { port: 5173, proxy: { '/api': { target: 'http://localhost:8000', changeOrigin: true } } },
  test: { globals: true, environment: 'jsdom', setupFiles: './src/test-setup.ts' },
});
```

### Verificación F3
```bash
npm run build
# Debe generar sw.js en dist/
ls dist/sw.js
```

---

## Fix F4: Purga nunca invocada (🟡 FUNCIONAL)

**Problema:** `purgarRegistrosAntiguos()` existe en `database.ts` pero nunca se llama.

**Archivo:** `grabakar-frontend/src/App.tsx`

### Cambio: invocar purga al inicio

```typescript
// En App.tsx, dentro del componente App, agregar useEffect:
import { purgarRegistrosAntiguos } from './db/database';

export default function App() {
  useEffect(() => { connectivityService.start(); }, []);
  useEffect(() => { purgarRegistrosAntiguos(); }, []);  // ← NUEVO
  // ... resto igual
}
```

### Verificación F4
```bash
grep -rn 'purgarRegistrosAntiguos' src/
# Debe aparecer en database.ts (definición) y App.tsx (invocación).
```

---

## Fix BE1: Tenant check en sync (⚠️ BACKEND)

**Archivo:** `grabakar-backend/api/services/sync_service.py`

### Cambio en `_validar_referencias`:

```python
@staticmethod
def _validar_referencias(data, usuario):  # ← agregar parámetro usuario
    if 'ley_caso_id' in data and not LeyCaso.objects.filter(id=data['ley_caso_id']).exists():
        raise ValueError(f"ley_caso_id {data['ley_caso_id']} no existe")
    if 'usuario_responsable_id' in data:
        resp = Usuario.objects.filter(id=data['usuario_responsable_id']).first()
        if not resp:
            raise ValueError(f"usuario_responsable_id {data['usuario_responsable_id']} no existe")
        if resp.tenant_id != usuario.tenant_id:
            raise ValueError("El usuario responsable no pertenece a tu tenant")
```

Actualizar las 2 llamadas a `_validar_referencias` en `_crear_grabado` y `_actualizar_grabado` para pasar `usuario`.

### Test

```python
# api/tests/test_sync.py — agregar:
def test_sync_rechaza_usuario_responsable_otro_tenant(self, auth_client, otro_tenant, ley, usuario_operador):
    from api.models import Usuario
    otro_user = Usuario.objects.create_user(
        username='otro', password='test', tenant=otro_tenant, nombre_completo='Otro', rol='operador',
    )
    batch = [_make_batch_item(otro_user, ley)]  # usuario de otro tenant
    r = auth_client.post('/api/v1/sync/upload/', {'device_id': 'dev1', 'batch': batch}, format='json')
    assert r.json()['fallidos'] == 1
```

---

## Tests: Archivos a crear

### Frontend Tests

#### `src/services/__tests__/AuthService.test.ts`

Crear con estos 7 tests (mockear fetch con `vi.fn()`):

1. `test_login_guarda_tokens` — después de login, IndexedDB tiene access_token, refresh_token, offline_token
2. `test_login_error_lanza_excepcion` — fetch 401 → throw Error con mensaje en español
3. `test_logout_elimina_tokens_conserva_grabados` — tokens borrados, grabados intactos
4. `test_sesion_valida_con_access_token` — isSessionValid retorna true si token_expiry > now
5. `test_sesion_valida_con_offline_token` — fallback a offline funciona
6. `test_sesion_invalida_ambos_expirados` — retorna false
7. `test_refresh_exitoso` — nuevo access_token guardado

#### `src/services/__tests__/SyncManager.test.ts`

Crear con estos 6 tests (mockear fetch, crear instancia fresca por test):

1. `test_obtener_pendientes` — registros con 'pendiente' se fetchean
2. `test_batch_max_100` — solo 100 enviados por request
3. `test_marca_sincronizado_tras_exito` — estado updated a 'sincronizado'
4. `test_marca_error_en_fallo` — errores individuales manejados
5. `test_backoff_incrementa` — 5s → 15s → 45s
6. `test_backoff_resetea_en_online` — vuelve a 5s

#### `src/db/database.test.ts` — agregar 4 tests

4. `test_purga_solo_sincronizados_antiguos` — purge only synced > 30 days
5. `test_purga_no_elimina_no_sincronizados` — pending records preserved
6. `test_detectar_duplicado_antiguo` — > 30 days returns undefined
7. `test_transaccion_atomica` — grabado + impresiones saved atomically

#### `src/utils/validation.test.ts` (NUEVO)

1. `test_vin_vacio_retorna_null` — input vacío es válido
2. `test_vin_17_chars_retorna_null` — longitud correcta OK
3. `test_vin_distinto_17_retorna_error` — retorna mensaje de error en español

#### `src/utils/deviceId.test.ts` (NUEVO)

1. `test_genera_y_persiste_device_id` — primera llamada genera y guarda
2. `test_segunda_llamada_retorna_misma_id` — no regenera

---

## Orden de ejecución

1. **C1** — UUID en vidrios (SyncManager + types + database + GrabadoFormPage)
2. **C2** — DEVICE_ID persistente (nuevo archivo deviceId.ts + GrabadoFormPage)
3. **F2** — "Offline" → "Sin conexión" (ConnectivityBadge)
4. **F1** — Eliminar navegación atrás en multi-vidrio
5. **F4** — Invocar purga en App.tsx
6. **F3** — Instalar y configurar vite-plugin-pwa
7. **BE1** — Tenant check en sync_service.py
8. **Tests** — Crear todos los archivos de test

## Verificación final

```bash
# Frontend
cd grabakar-frontend
npm run lint        # 0 errores
npx vitest run      # todos pasan
npm run build       # dist/ con sw.js
grep -rni 'offline' src/components/ | grep -v className  # 0 matches de texto "Offline"
grep -rn 'Math.random' src/pages/                        # 0 matches
grep -rn 'i - 1' src/pages/FlujoMultiVidrioPage.tsx     # 0 matches

# Backend
cd grabakar-backend
pytest -q           # todos pasan
```

## Commit convention

```
fix(sync): enviar UUID de vidrios en payload de sincronización
fix(grabado): persistir device_id en IndexedDB
fix(i18n): cambiar texto 'Offline' a 'Sin conexión'
fix(flujo-vidrio): eliminar navegación hacia atrás
feat(pwa): configurar vite-plugin-pwa
fix(db): invocar purga automática al inicio
fix(sync-service): validar tenant de usuario responsable en sync
test(frontend): agregar tests para AuthService, SyncManager, database, validation
```
