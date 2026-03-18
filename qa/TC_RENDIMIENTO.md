# TC_RENDIMIENTO — Test Cases: Performance & Stress

**Module**: Performance, Stress, and Resource Management

---

## App Startup Performance

### TC-PERF-001 — Tiempo de arranque en frío (APK)
**Priority**: P1  
**Steps**: Force-stop app, reopen from launcher

**Expected**:
- App usable (login or Home) within 4 seconds
- No white screen > 2 seconds

### TC-PERF-002 — Tiempo de arranque en caliente
**Priority**: P2  
**Steps**: Switch away from app, then return

**Expected**: App resume within 1 second

### TC-PERF-003 — Startup con muchos registros en IndexedDB
**Priority**: P1  
**Preconditions**: 500+ grabados en IndexedDB  
**Steps**: Open app

**Expected**:
- Startup not noticeably slower
- Home loads within 5 seconds

---

## Form Performance

### TC-PERF-010 — Formulario de grabado responsive
**Priority**: P1  
**Steps**: Type rapidly in patente field

**Expected**:
- No input lag
- Normalization (uppercase) instant
- No dropped characters

### TC-PERF-011 — Dropdown loading time
**Priority**: P2  
**Steps**: Tap on any dropdown (tipo movimiento, tipo vehículo, etc.)

**Expected**: Options appear within 200ms from cached config

---

## Sync Performance

### TC-PERF-020 — Sync de 100 registros
**Priority**: P0  
**Preconditions**: 100 grabados pendientes  
**Steps**: Trigger sync

**Expected**:
- Completes within 30 seconds on 4G connection
- No UI freezing during sync
- Progress indication updates

### TC-PERF-021 — Sync de 500 registros (accumulated)
**Priority**: P1  
**Preconditions**: 500 grabados from days without connection  
**Steps**: Restore connection

**Expected**:
- 5 batches of 100
- Each batch succeeds
- App remains responsive during sync
- Total sync < 3 minutes on 4G

### TC-PERF-022 — Sync no bloquea UI
**Priority**: P0  
**Steps**: Start sync, simultaneously navigate app

**Expected**:
- Can create new grabados while sync runs
- Can navigate to different screens
- No UI freezes or "Application Not Responding" (ANR)

---

## Health Endpoint Performance

### TC-PERF-030 — Health check < 500ms
**Priority**: P1  
**Steps**: Time `GET /api/v1/health/`

**Expected**: Response in < 500ms

---

## IndexedDB Performance

### TC-PERF-040 — Duplicate detection speed
**Priority**: P1  
**Preconditions**: 1000 grabados in IndexedDB  
**Steps**: Check duplicate for a patente

**Expected**: Result within 200ms (indexed queries)

### TC-PERF-041 — Purge speed (30-day cleanup)
**Priority**: P2  
**Preconditions**: 500 old synced records  
**Steps**: Trigger purge

**Expected**:
- Completes within 5 seconds
- No UI interruption

### TC-PERF-042 — Data persistence under volume
**Priority**: P1  
**Steps**: Create 200 grabados in a session

**Expected**:
- All saved correctly
- No IndexedDB errors
- App remains responsive

---

## Memory & Resources

### TC-PERF-050 — Uso de memoria estable
**Priority**: P2  
**Steps**: Use app for 30 minutes, creating grabados

**Expected**:
- Memory usage stable (no leak)
- No increasing memory over time
- App does not get killed by OS

### TC-PERF-051 — Battery impact monitoring
**Priority**: P3  
**Steps**: Use app for 2 hours continuously

**Expected**:
- No excessive battery drain
- Reasonable CPU usage during idle

### TC-PERF-052 — Network usage efficient
**Priority**: P2  
**Steps**: Monitor network during sync

**Expected**:
- No unnecessary requests when all synced
- Health pings only every 30s (not more frequent)
- Sync payloads reasonably sized

---

## Stress Testing

### TC-PERF-060 — 1000 grabados offline acumulados
**Priority**: P1  
**Steps**:
1. Create 1000 grabados offline over several days
2. Reconnect

**Expected**:
- All 1000 eventually synced (10 batches of 100)
- No data loss
- App remains functional throughout

### TC-PERF-061 — Rapid grabado creation (20 in 5 minutes)
**Priority**: P1  
**Steps**: Create 20 grabados as fast as possible

**Expected**:
- All saved correctly
- Each has unique UUID
- No data corruption

### TC-PERF-062 — App open for 8 hours (work shift)
**Priority**: P2  
**Steps**: Leave app open, periodically creating grabados, for 8 hours

**Expected**:
- App remains responsive
- Token refresh works correctly
- Sync continues functioning
- No memory issues

### TC-PERF-063 — Low-end device simulation
**Priority**: P2  
**Steps**: Run on emulator with limited RAM (2GB)

**Expected**:
- App functions correctly
- May be slower but no crashes
- No ANR dialogs
