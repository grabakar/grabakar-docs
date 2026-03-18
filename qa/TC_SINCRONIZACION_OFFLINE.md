# TC_SINCRONIZACION_OFFLINE — Test Cases: Sincronización Offline-First

**Module**: Offline Sync Engine  
**Feature Docs**: [SINCRONIZACION_OFFLINE.md](../features/SINCRONIZACION_OFFLINE.md)  
**API Contract**: `POST /api/v1/sync/upload/`, `GET /api/v1/health/`

---

## Happy Path

### TC-SYNC-001 — Sync automática al recuperar conexión
**Priority**: P0  
**Preconditions**: 3 grabados created offline, `estado_sync: "pendiente"`  
**Steps**:
1. Disable airplane mode (restore connection)
2. Wait for automatic sync

**Expected**:
- Sync indicator shows spinning icon
- All 3 records synced successfully
- `estado_sync` → `"sincronizado"` for all
- `fecha_sincronizacion` set
- Pending count goes to 0

### TC-SYNC-002 — Sync post-guardado con conexión
**Priority**: P0  
**Preconditions**: Online  
**Steps**:
1. Create and save a new grabado

**Expected**:
- Immediate sync attempt after save
- Record transitions to `"sincronizado"` within seconds

### TC-SYNC-003 — Sync batch processing (multiple records)
**Priority**: P0  
**Preconditions**: 10 grabados pendientes  
**Steps**: Restore connection

**Expected**:
- Records sent as batch to `POST /sync/upload/`
- Response shows `procesados: 10, exitosos: 10, fallidos: 0`
- All records updated in IndexedDB

### TC-SYNC-004 — Sync status endpoint
**Priority**: P1  
**Steps**: Call `GET /sync/status/?device_id=xxx` after sync

**Expected**:
- Returns `ultima_sincronizacion`, `registros_sincronizados`, `registros_pendientes_servidor: 0`

---

## Connectivity Detection

### TC-SYNC-010 — Indicador online/offline
**Priority**: P0  
**Steps**:
1. Toggle airplane mode on emulator

**Expected**:
- Badge/indicator changes to red (offline) / green (online)
- Transition happens within 3 seconds

### TC-SYNC-011 — Detección por ping periódico
**Priority**: P1  
**Steps**:
1. Observe periodic `GET /health/` calls (every 30s)

**Expected**:
- Ping made every ~30 seconds when `navigator.onLine === true`
- If 3 consecutive pings fail → considered offline

### TC-SYNC-012 — Online event trigger
**Priority**: P1  
**Steps**:
1. Go offline, create records
2. Go online

**Expected**:
- `online` event triggers sync after 3-second debounce
- No premature sync on brief connectivity flickers

### TC-SYNC-013 — App resume trigger
**Priority**: P1  
**Steps**:
1. Minimize app (go to another app)
2. Return to GrabaKar app (visibility change)

**Expected**:
- Connection check performed
- If online and pending records → sync initiated

### TC-SYNC-014 — Periodic sync trigger
**Priority**: P2  
**Steps**: Leave app open, online, with pending records for > 5 minutes

**Expected**:
- Sync attempted periodically (every 5 min)

---

## Error Handling & Retry

### TC-SYNC-020 — Reintento con backoff exponencial
**Priority**: P0  
**Steps**:
1. Create offline records
2. Go online but with backend down

**Expected**:
- Retry intervals: 5s → 15s → 45s → ~2m → ~5m → ~20m → 30m (max)
- Backoff multiplier ×3

### TC-SYNC-021 — Reset de backoff al reconectar
**Priority**: P1  
**Steps**:
1. Trigger backoff (backend errors)
2. Go offline, then back online

**Expected**:
- Backoff resets to 5s initial interval

### TC-SYNC-022 — Errores parciales en batch
**Priority**: P0  
**Preconditions**: Batch of 5 records, 1 has invalid `ley_caso_id`  
**Steps**: Trigger sync

**Expected**:
- 4 records: `estado_sync: "sincronizado"`
- 1 record: `estado_sync: "error"` with error detail
- Other records NOT affected by the one failure

### TC-SYNC-023 — Token expirado durante sync
**Priority**: P1  
**Steps**:
1. Start syncing large batch
2. Access token expires mid-batch

**Expected**:
- 401 intercepted
- Token auto-refreshed
- Batch retried

### TC-SYNC-024 — Sync con respuesta "ignorado"
**Priority**: P1  
**Preconditions**: Record already on server with newer `fecha_creacion_local`  
**Steps**: Sync same UUID

**Expected**:
- Response: `status: "ignorado"`
- Local record marked as `"sincronizado"` (server already has it)

---

## Batch Limits

### TC-SYNC-030 — Batch máximo 100 registros
**Priority**: P1  
**Preconditions**: 250 pending records  
**Steps**: Trigger sync

**Expected**:
- Sent in batches: 100 + 100 + 50
- All eventually synced

### TC-SYNC-031 — Batch vacío rechazado
**Priority**: P2  
**Steps**: Send empty batch to API

**Expected**:
- 400: _"El batch no puede estar vacío."_

### TC-SYNC-032 — Batch > 100 rechazado
**Priority**: P2  
**Steps**: Send > 100 records in single batch

**Expected**:
- 400: _"El batch no puede exceder 100 registros."_

---

## Conflict Resolution

### TC-SYNC-040 — Dispositivo gana si más reciente
**Priority**: P0  
**Preconditions**: Server has record with `fecha: T1`, device sends same UUID with `fecha: T2 > T1`  
**Steps**: Sync

**Expected**:
- Server updates record
- Response: `status: "ok"`

### TC-SYNC-041 — Servidor mantiene si dispositivo más antiguo
**Priority**: P1  
**Preconditions**: Server has record with `fecha: T2`, device sends same UUID with `fecha: T1 < T2`  
**Steps**: Sync

**Expected**:
- Server ignores update
- Response: `status: "ignorado"`
- Local record marked as synced

### TC-SYNC-042 — Registros UTC timezone handling
**Priority**: P2  
**Steps**: Create record in timezone `America/Santiago` (-03:00)  
**Expected**:
- `fecha_creacion_local` includes timezone offset in ISO 8601
- Server compares in UTC correctly

---

## Data Purge

### TC-SYNC-050 — Purga de registros sincronizados > 30 días
**Priority**: P1  
**Preconditions**: IndexedDB has synced records from 35 days ago  
**Steps**: App startup (triggers purge)

**Expected**:
- Records > 30 days with `estado_sync: "sincronizado"` deleted from IndexedDB
- Associated `ImpresionVidrio` records also deleted

### TC-SYNC-051 — Registros no sincronizados nunca se purgan
**Priority**: P0  
**Preconditions**: IndexedDB has unsynced records from 60 days ago  
**Steps**: App startup

**Expected**:
- Records with `estado_sync: "pendiente"` or `"error"` are NEVER purged
- Regardless of age

### TC-SYNC-052 — Purga periódica (cada 24h)
**Priority**: P2  
**Steps**: Leave app running for 24+ hours

**Expected**:
- Purge runs again after 24 hours

---

## Sync Indicator UI

### TC-SYNC-060 — Badge muestra registros pendientes
**Priority**: P1  
**Steps**: Create 5 grabados offline

**Expected**:
- Badge shows `(5)` in warning color

### TC-SYNC-061 — Icono giratorio durante sincronización
**Priority**: P1  
**Steps**: Trigger sync

**Expected**:
- Sync icon animates/spins during active sync
- Stops when sync completes

### TC-SYNC-062 — Tap en indicador muestra detalle
**Priority**: P2  
**Steps**: Tap on sync indicator

**Expected**:
- Detail: _"X registros pendientes | Última sincronización: hace Y min"_

### TC-SYNC-063 — Icono rojo con error
**Priority**: P1  
**Preconditions**: Sync error occurred  
**Steps**: Check sync indicator

**Expected**:
- Red icon with tooltip showing error description

---

## Edge Cases

### TC-SYNC-070 — Conexión inestable (online/offline rápido)
**Priority**: P1  
**Steps**: Toggle connection rapidly (on-off-on within 2 seconds)

**Expected**:
- 3-second debounce prevents premature sync
- Sync only starts on stable connection

### TC-SYNC-071 — Sync interrumpida (app se cierra)
**Priority**: P1  
**Steps**:
1. Start syncing
2. Force close app mid-request

**Expected**:
- Records remain `"pendiente"` (not partially synced)
- On reopen, sync retries normally
- Server is idempotent (same UUID safe to resend)

### TC-SYNC-072 — Reloj del dispositivo adelantado
**Priority**: P2  
**Preconditions**: Device clock set 48h in the future  
**Steps**: Create grabado, sync

**Expected**:
- Server may reject if `fecha_creacion_local > now() + 24h`
- Error message returned

### TC-SYNC-073 — Backend caído por horas
**Priority**: P1  
**Steps**: Backend unreachable for 2+ hours  
**Expected**:
- Backoff reaches 30 min max
- Records accumulate in IndexedDB without limit
- App remains fully operational for local operations

### TC-SYNC-074 — Dispositivo sin espacio para IndexedDB
**Priority**: P2  
**Steps**: Fill device storage, try to save new grabado

**Expected**:
- `QuotaExceededError` caught
- Error: _"Almacenamiento lleno. Sincroniza los registros o libera espacio."_
- Purge of old records may free space

### TC-SYNC-075 — Múltiples dispositivos con mismo usuario
**Priority**: P2  
**Steps**:
1. User logged in on device A and B
2. Both create grabados offline
3. Both sync

**Expected**:
- Different UUIDs, no collision
- Server accepts all records
