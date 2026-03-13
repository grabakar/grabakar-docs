# REPORT_XLSX_IMPLEMENTATION — Multi-Tab XLSX Report Generation

**Status**: Plan
**Priority**: P2
**Depends on**: Sync functional, data model migration applied

---

## 1. Problem Statement

The existing report system (`ReporteService`) produces flat JSON/CSV scoped to a single tenant. The business requires **multi-tab XLSX reports** that:

- Span **all tenants** (platform-wide) or a single tenant
- Match the structure of the current auto-generated Excel reports (3 tabs)
- Generated daily, covering **month-to-date** (1st of month through yesterday)
- Include fields not present in the current data model

### Reference Reports Analyzed

- `Reporte grabakar diario 2026_Feb_25.xlsx` — month-to-date (Feb 1–24), 1490 records, 37 branches
- `Reporte grabakar diario 2026_Mar_03.xlsx` — month-to-date (Mar 1–2), 68 records, 21 branches

Both files are **month-to-date** reports generated daily. The "daily" name refers to generation frequency, not the time window. Each report covers day 1 of the current month through `report_date - 1`. Day columns in Tab 2 grow as the month progresses.

### Key Finding: Grouping by Sync Date

Records are attributed to their **`fecha_sincronizacion`** (server receipt date), NOT `fecha_creacion_local`. Verified:

- Feb 25 report, day 2 column = 79 records. Records with `fecha_sincronizacion` = 2026-02-02 = 79. Exact match.
- Feb 25 report, day 9 column = 276 records. Records with `fecha_sincronizacion` = 2026-02-09 = 276. Exact match.
- Mar 03 report, day 2 column = 68 records. All 68 have `fecha_sincronizacion` = 2026-03-02.
- Mar 03 report includes records with `fecha_creacion_local` from Oct 2025 through Mar 2026 — all synced on Mar 2.

This means `fecha_sincronizacion` is the canonical grouping/filtering field for all report tabs.

---

## 2. Data Model Gaps

### 2.1 Missing: Sucursal (Branch)

The Excel groups by CLIENTE + SUCURSAL. A single client (e.g., "Aspillaga") has many branches ("Coquimbo", "El Belloto", "Quillota", etc.). The current Tenant model has no branch concept.

**Option A — Sucursal model (recommended)**:

```python
class Sucursal(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE, related_name='sucursales')
    nombre = models.CharField(max_length=150)
    activa = models.BooleanField(default=True)

    class Meta:
        unique_together = ['tenant', 'nombre']
        indexes = [models.Index(fields=['tenant', 'activa'])]
```

Then add `sucursal = models.ForeignKey(Sucursal, on_delete=models.SET_NULL, null=True, blank=True)` to **Usuario** (the branch is where the operator works) and optionally to **Grabado** (denormalized for reporting speed).

**Option B — Sucursal as CharField on Usuario**: Simpler, but no referential integrity. Reporting requires `GROUP BY` on free text.

**Decision**: Option A. The branch dimension is critical for all aggregation tabs.

### 2.2 Missing: tipo_cliente (Client Classification)

Every row in the Excel has `TIPO CLIENTE` (e.g., "CONCESION"). This is a tenant-level attribute.

```python
# Add to Tenant model
tipo_cliente = models.CharField(max_length=50, blank=True, default='')
```

### 2.3 Missing: responsable_texto (Physical Operator)

The Excel has two person fields:
- **NOMBRE** = the registered device user (`Usuario.nombre_completo`)
- **RESPONSABLE** = free-text, the person who physically performed the engraving (typed at job time, often different from NOMBRE, sometimes blank)

The current `Grabado.usuario_responsable` FK maps to NOMBRE. RESPONSABLE has no field.

```python
# Add to Grabado model
responsable_texto = models.CharField(max_length=150, blank=True, default='')
```

### 2.4 Email

`Usuario` extends `AbstractUser` which includes `email`. Already in the model but not populated or included in reports. No model change needed — just ensure email is set during user creation and included in report queries.

### Summary of Model Changes

| Change | Model | Field | Type | Migration |
|---|---|---|---|---|
| New model | `Sucursal` | — | — | `0002_sucursal` |
| New FK | `Usuario` | `sucursal` | FK(Sucursal), nullable | `0002_sucursal` |
| New FK | `Grabado` | `sucursal` | FK(Sucursal), nullable | `0002_sucursal` |
| New field | `Tenant` | `tipo_cliente` | CharField(50) | `0002_sucursal` |
| New field | `Grabado` | `responsable_texto` | CharField(150) | `0002_sucursal` |

---

## 3. Target XLSX Structure

### Tab 1: Resumen

| CLIENTE | SUCURSAL | TOTAL PATENTES |
|---|---|---|
| Aspillaga | Coquimbo | 33 |
| Aspillaga | El Belloto | 42 |
| ... | ... | ... |
| **Total** | | **1490** |

Query: `Grabado` grouped by `tenant__nombre`, `sucursal__nombre`, count of records in date range.

### Tab 2: Resumen por dia

| CLIENTE | SUCURSAL | 1 | 2 | 3 | ... | N | TOTAL |
|---|---|---|---|---|---|---|---|
| Aspillaga | Coquimbo | 0 | 4 | 1 | ... | 1 | 33 |

Query: Same grouping but with `fecha_sincronizacion__day` annotation. Columns = each day from 1 to `report_date - 1`. Dynamic column count grows throughout the month (1 on the 2nd, 28–31 by month end).

### Tab 3: Patentes

| PATENTE1 | CANTIDAD | VIN | OT | RESPONSABLE | DISPOSITIVO | FECHA CREADA | FECHA SINCRONIZADA | NOMBRE | EMAIL | CLIENTE | SUCURSAL | TIPO CLIENTE |
|---|---|---|---|---|---|---|---|---|---|---|---|---|

Query: Flat select with joins to Usuario, Tenant, Sucursal. One row per Grabado.

Field mapping:

| Column | Source |
|---|---|
| PATENTE1 | `Grabado.patente` |
| CANTIDAD | `SUM(ImpresionVidrio.cantidad_impresiones)` for that grabado |
| VIN | `Grabado.vin_chasis` |
| OT | `Grabado.orden_trabajo` |
| RESPONSABLE | `Grabado.responsable_texto` |
| DISPOSITIVO | `Grabado.device_id` |
| FECHA CREADA | `Grabado.fecha_creacion_local` (date only) |
| FECHA SINCRONIZADA | `Grabado.fecha_sincronizacion` (date only) |
| NOMBRE | `Grabado.usuario_responsable.nombre_completo` |
| EMAIL | `Grabado.usuario_responsable.email` |
| CLIENTE | `Grabado.tenant.nombre` |
| SUCURSAL | `Grabado.sucursal.nombre` |
| TIPO CLIENTE | `Grabado.tenant.tipo_cliente` |

---

## 4. Implementation Tasks

### Phase A — Data Model Migration

**A1. Create Sucursal model + migrate fields**

File: `api/models.py`

- Add `Sucursal` model
- Add `Tenant.tipo_cliente`
- Add `Usuario.sucursal` FK (nullable)
- Add `Grabado.sucursal` FK (nullable)
- Add `Grabado.responsable_texto`
- Run `makemigrations`, `migrate`

**A2. Update serializers**

File: `api/serializers/grabados.py`

- Add `responsable_texto` and `sucursal` to `GrabadoSerializer`, `GrabadoCreateSerializer`
- Add sucursal to sync upload handling

**A3. Update SyncService**

File: `api/services/sync_service.py`

- Accept `responsable_texto` and `sucursal_id` in batch upload
- Auto-assign sucursal from usuario if not provided

**A4. Update admin**

File: `api/admin.py`

- Register `Sucursal`
- Add sucursal to `GrabadoAdmin`, `UsuarioAdmin`

**A5. Seed data**

File: `api/fixtures/initial.json`

- Add sample Sucursal records for demo tenant

---

### Phase B — XLSX Report Service

**B1. Add openpyxl dependency**

File: `requirements.txt`

- Add `openpyxl>=3.1.0`

**B2. Create XLSX report service**

File: `api/services/reporte_xlsx_service.py`

Core class: `ReporteXLSXService`

```
Methods:
  generar_xlsx(report_date, tenant_id=None) -> BytesIO
    - report_date: date — report covers month-to-date (1st of month through report_date - 1)
    - tenant_id=None means platform-wide (all tenants)
    - Returns in-memory XLSX workbook

  _build_queryset(report_date, tenant_id) -> QuerySet
    - Filters: fecha_sincronizacion >= month_start AND < report_date
    - Joins to Usuario, Tenant, Sucursal

  _tab_resumen(ws, qs)
    - GROUP BY tenant__nombre, sucursal__nombre → count
    - Write headers + data rows + total row

  _tab_resumen_por_dia(ws, qs, report_date)
    - GROUP BY tenant__nombre, sucursal__nombre, TruncDay(fecha_sincronizacion)
    - Pivot into day-of-month columns (1 through report_date.day - 1)
    - Write headers (dynamic day columns) + data rows + totals per day

  _tab_patentes(ws, qs)
    - Flat detail, one row per grabado
    - Annotate with SUM(cantidad_impresiones)
    - Write all 13 columns
```

Query patterns (ORM, no raw SQL):

```python
# Base queryset — filter by fecha_sincronizacion (month-to-date)
qs = Grabado.objects.filter(
    fecha_sincronizacion__gte=month_start,  # 1st of current month
    fecha_sincronizacion__lt=report_date,   # up to (not including) report date
)
if tenant_id:
    qs = qs.filter(tenant_id=tenant_id)

# Tab 1 - Resumen
qs.values('tenant__nombre', 'sucursal__nombre') \
  .annotate(total=Count('uuid')) \
  .order_by('tenant__nombre', 'sucursal__nombre')

# Tab 2 - Resumen por dia (group by sync date day)
qs.values('tenant__nombre', 'sucursal__nombre', day=TruncDay('fecha_sincronizacion')) \
  .annotate(total=Count('uuid')) \
  .order_by('tenant__nombre', 'sucursal__nombre', 'day')

# Tab 3 - Patentes
qs.select_related('usuario_responsable', 'tenant', 'sucursal') \
  .prefetch_related('impresiones') \
  .annotate(cantidad=Sum('impresiones__cantidad_impresiones')) \
  .order_by('fecha_sincronizacion', 'patente')
```

**B3. XLSX styling**

Minimal styling to match the reference Excel:
- Title row (merged cells, bold) per tab
- Header row with bold + background color
- Auto-width columns
- Total row with bold
- No empty rows at top (or 5 empty rows to match original)

---

### Phase C — API Endpoint

**C1. Platform-wide XLSX report endpoint (admin only)**

New view or extend existing:

```
GET /api/v1/reportes/plataforma/?fecha=YYYY-MM-DD&formato=xlsx
```

- `fecha` = report date (default: today). Report covers 1st of that month through `fecha - 1 day`.
- `tenant_id` = optional. If omitted, platform-wide (all tenants). If provided, single-tenant.
- Permission: admin only (cross-tenant when no tenant_id)
- Calls `ReporteXLSXService.generar_xlsx(report_date, tenant_id)`
- Response: XLSX file download
- Filename: `Reporte grabakar diario YYYY_Mon_DD.xlsx`

URL registration in `api/urls.py`:

```python
path('reportes/plataforma/', ReportePlataformaView.as_view()),
```

The existing `ReporteDiarioView` and `ReporteMensualView` remain unchanged (JSON/CSV, per-tenant, `fecha_creacion_local`). The XLSX format is only on this new endpoint because it uses a different date strategy (`fecha_sincronizacion`) and different output structure (3-tab workbook).

---

### Phase D — Celery Scheduled Task

**D1. Update reportes_tasks.py**

File: `api/tasks/reportes_tasks.py`

```
Task: generar_reporte_xlsx_diario()
  - Runs daily via celery-beat (e.g., 01:00 AM America/Santiago)
  - report_date = today → generates month-to-date (1st through yesterday)
  - Saves to configured storage path or sends via email
  - Filename convention: Reporte grabakar diario YYYY_Mon_DD.xlsx
```

No separate monthly task needed — the daily task on the 1st of each month naturally covers the entire previous month (report_date = Mar 1 → covers Feb 1–28).

**D2. Configure celery-beat schedule**

Via django-celery-beat DB scheduler (already in docker-compose). Add periodic tasks via fixture or admin.

---

### Phase E — Tests

**E1. Unit tests for ReporteXLSXService**

File: `api/tests/test_reportes_xlsx.py`

- Test with multiple tenants and sucursales
- Test Tab 1 aggregation correctness
- Test Tab 2 day-pivot correctness
- Test Tab 3 flat detail completeness
- Test tenant_id=None (platform-wide) vs specific tenant
- Test empty date range
- Test offline edge case (fecha_creacion_local != fecha_sincronizacion)

**E2. Integration tests for endpoints**

- Test XLSX download response headers and content-type
- Test permission (403 for operador)
- Test platform endpoint admin-only

---

## 5. Execution Order

```
A1 → A2 → A3 → A4 → A5 (model changes, one migration)
  ↓
B1 → B2 → B3 (XLSX service)
  ↓
C1 (API endpoint)
  ↓
D1 → D2 (Celery task + beat config)
  ↓
E1 → E2 (tests)
```

A-series is blocking. B and C can partially overlap. D is independent of C. E runs last.

---

## 6. Files Changed / Created

| Action | Path |
|---|---|
| Modified | `api/models.py` |
| Modified | `api/serializers/grabados.py` |
| Modified | `api/services/sync_service.py` |
| Modified | `api/admin.py` |
| Modified | `api/views/reportes.py` |
| Modified | `api/urls.py` |
| Modified | `api/tasks/reportes_tasks.py` |
| Modified | `api/fixtures/initial.json` |
| Modified | `requirements.txt` |
| Created | `api/services/reporte_xlsx_service.py` |
| Created | `api/tests/test_reportes_xlsx.py` |
| Created | `api/migrations/0002_sucursal_*.py` (auto) |

---

## 7. Offline Edge Cases

Handled by design. Since all queries filter/group by `fecha_sincronizacion` (see Section 1 — Key Finding), offline records automatically appear in the report of the day they sync, regardless of `fecha_creacion_local`. This matches verified Excel behavior.

Example from data: Boris Araya (Indumotora, Local Santa Rosa) has 15 records with `fecha_creacion_local` spanning 2025-09-30 to 2026-01-21, all with `fecha_sincronizacion` = 2026-03-02. In the Mar 03 report, all 15 appear under day 2 of March.

**No `filtro_fecha` param needed** — the existing reports exclusively use sync date. If a creation-date view is needed later, it can be added as a separate endpoint.

---

## 8. Relationship with Existing ReporteService

The current `api/services/reporte_service.py` provides JSON/CSV reports per-tenant filtered by `fecha_creacion_local`. The new `ReporteXLSXService` is a separate service that:

- Filters by `fecha_sincronizacion` (different from existing)
- Supports cross-tenant (platform-wide) output
- Produces multi-tab XLSX (different format)

Keep both services. The existing one serves the per-tenant API (JSON/CSV for supervisors). The new one serves platform-level XLSX exports. Do not modify `ReporteService` — it remains stable for its current consumers.

---

## 9. Open Questions

1. **Storage**: Should generated XLSX files be persisted (S3/GCS/local) or generated on-demand per request? On-demand is simpler; storage is needed if email delivery is required.
2. **Email delivery**: Should the daily Celery task email the report? If so, to whom? This requires SMTP config and a recipient list per tenant.
3. **Historical backfill**: Once the Sucursal model exists, existing Grabado records have `sucursal=NULL`. Need a backfill strategy (admin UI, CSV import, or manual SQL).
4. **The 30-day purge**: The current system purges synced records after 30 days. The Feb 25 report contains data from September 2025. If purge runs, historical reports can't be regenerated. Consider: (a) extend retention, (b) store generated reports, (c) archive to cold storage before purge.
