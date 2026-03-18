# TC_REPORTES — Test Cases: Reportes

**Module**: Reports Generation  
**Feature Docs**: [REPORTES.md](../features/REPORTES.md)  
**API Contract**: `GET /api/v1/reportes/diario/`, `GET /api/v1/reportes/mensual/`, `GET /api/v1/reportes/plataforma/`

---

## Per-Tenant Reports (JSON/CSV)

### TC-REP-001 — Reporte diario JSON
**Priority**: P0  
**Preconditions**: Logged in as supervisor, grabados exist for today  
**Steps**:
1. Navigate to reports section
2. Select daily report, today's date
3. Request JSON format

**Expected**:
- Response includes: `fecha`, `total_grabados`, `por_operador` (array), `por_tipo` (array), `registros`
- Counts match actual records for today

### TC-REP-002 — Reporte diario CSV
**Priority**: P1  
**Steps**: Request daily report in CSV format

**Expected**:
- CSV file downloaded with UTF-8 BOM
- Headers: uuid, patente, operador, fecha_registro, cantidad_impresiones, fecha_impresion, tipo_movimiento, tipo_vehiculo, device_id, estado_sync, fecha_sincronizacion
- Data matches JSON report

### TC-REP-003 — Reporte mensual JSON
**Priority**: P1  
**Steps**: Request monthly report for current month

**Expected**:
- Returns all records for the month
- Same structure as daily but for month range

### TC-REP-004 — Reporte mensual CSV
**Priority**: P1  
**Steps**: Monthly report, CSV format

**Expected**: Same as daily CSV but with month's data

### TC-REP-005 — Reporte con filtro por operador
**Priority**: P1  
**Steps**: Request report with `operador_id` filter

**Expected**:
- Only records from specified operador
- Other operadores excluded

### TC-REP-006 — Reporte sin registros
**Priority**: P2  
**Steps**: Request report for date with no records

**Expected**:
- Empty results array
- `total_grabados: 0`
- No errors

---

## Permissions

### TC-REP-010 — Operador no puede acceder reportes
**Priority**: P0  
**Preconditions**: Logged in as `operador`  
**Steps**: Navigate to / request reports endpoint

**Expected**:
- 403: _"No tienes permiso para esta operación"_
- Reports section hidden/disabled in UI

### TC-REP-011 — Supervisor puede descargar reportes
**Priority**: P0  
**Preconditions**: Logged in as `supervisor`  
**Steps**: Request any report

**Expected**: Successful response with data

### TC-REP-012 — Admin puede descargar reportes
**Priority**: P1  
**Steps**: As admin, request per-tenant reports

**Expected**: Successful response with data

### TC-REP-013 — Supervisor solo ve datos de su tenant
**Priority**: P0  
**Steps**: Supervisor requests report

**Expected**:
- Only records from their tenant
- No cross-tenant data leakage

---

## Platform XLSX Reports (Pending Implementation)

### TC-REP-020 — Reporte plataforma XLSX descarga
**Priority**: P0 (when implemented)  
**Preconditions**: Admin user, XLSX feature active  
**Steps**: `GET /reportes/plataforma/?fecha=YYYY-MM-DD&formato=xlsx`

**Expected**:
- XLSX file downloaded
- 3 tabs: Resumen, Resumen por dia, Patentes
- Content-Type: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

### TC-REP-021 — Tab 1: Resumen
**Priority**: P1  
**Steps**: Open Tab 1 of XLSX

**Expected**:
- Grouped by CLIENTE + SUCURSAL
- Column: TOTAL PATENTES (count)

### TC-REP-022 — Tab 2: Resumen por día
**Priority**: P1  
**Steps**: Open Tab 2

**Expected**:
- Cross-tab with day columns (1, 2, ..., N)
- Grouped by CLIENTE + SUCURSAL
- TOTAL column
- Based on `fecha_sincronizacion`

### TC-REP-023 — Tab 3: Patentes (detalle)
**Priority**: P1  
**Steps**: Open Tab 3

**Expected**:
- One row per grabado
- 13 columns: PATENTE, CANTIDAD, VIN, OT, RESPONSABLE, DISPOSITIVO, FECHA CREADA, FECHA SINCRONIZADA, NOMBRE, EMAIL, CLIENTE, SUCURSAL, TIPO CLIENTE

### TC-REP-024 — Reporte es month-to-date
**Priority**: P1  
**Steps**: Request report for March 15

**Expected**:
- Data covers March 1 through March 14 (fecha - 1 day)

### TC-REP-025 — No admin → 403
**Priority**: P0  
**Steps**: Non-admin requests plataforma report

**Expected**: 403 Permission Denied

---

## Celery Automated Reports

### TC-REP-030 — Tarea diaria genera reporte
**Priority**: P2  
**Steps**: Verify `generar_reporte_diario` Celery task  

**Expected**:
- Task runs at scheduled time
- Generates report for previous day

### TC-REP-031 — Tarea XLSX diaria (01:00 AM)
**Priority**: P2  
**Steps**: Verify `generar_reporte_xlsx_diario` scheduled at 01:00 AM Santiago time

**Expected**: XLSX generated for month-to-date

---

## Edge Cases

### TC-REP-040 — Registros tardíos (offline sync)
**Priority**: P1  
**Steps**:
1. Create grabado offline on March 10
2. Sync on March 15
3. Request March 15 report (XLSX version)

**Expected**:
- Record appears under March 15 in `fecha_sincronizacion`-based report
- Record appears under March 10 in `fecha_creacion_local`-based report

### TC-REP-041 — Reporte con fecha futura
**Priority**: P2  
**Steps**: Request report for a future date

**Expected**:
- Empty results or error
- No crash

### TC-REP-042 — Reporte cross-tenant sin tenant_id
**Priority**: P1  
**Steps**: Admin request to `/reportes/plataforma/` without `tenant_id`

**Expected**:
- Returns data from ALL tenants
- Only admin can access
