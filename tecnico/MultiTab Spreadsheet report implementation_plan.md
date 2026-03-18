# Multi-Tab XLSX Report Implementation

Implement multi-tab XLSX report generation with new data model fields (`Sucursal`, `tipo_cliente`, `responsable_texto`), a new `ReporteXLSXService` producing 3-tab workbooks, API endpoints supporting XLSX download, and Celery scheduled tasks.

## Proposed Changes

### Data Model — [api/models.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/models.py)

#### [NEW] Sucursal model
```python
class Sucursal(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE, related_name='sucursales')
    nombre = models.CharField(max_length=150)
    activa = models.BooleanField(default=True)
    class Meta:
        unique_together = ['tenant', 'nombre']
        indexes = [models.Index(fields=['tenant', 'activa'])]
```

#### [MODIFY] [models.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/models.py)
- Add `tipo_cliente = CharField(max_length=50)` to [Tenant](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/models.py#5-14)
- Add `sucursal = FK(Sucursal, nullable)` to [Usuario](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/models.py#16-29)
- Add `sucursal = FK(Sucursal, nullable)` and `responsable_texto = CharField(max_length=150)` to [Grabado](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/models.py#40-82)
- Add index on `fecha_sincronizacion` for report filtering

---

### Serializers — [api/serializers/grabados.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/serializers/grabados.py)

#### [MODIFY] [grabados.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/serializers/grabados.py)
- Add `responsable_texto` and `sucursal` to [GrabadoSerializer](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/serializers/grabados.py#35-57) fields
- Add `sucursal_id` to [GrabadoCreateSerializer](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/serializers/grabados.py#59-83) fields

---

### Sync Service — [api/services/sync_service.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/services/sync_service.py)

#### [MODIFY] [sync_service.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/services/sync_service.py)
- Accept `responsable_texto` and `sucursal_id` in [_crear_grabado](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/tests/test_reportes.py#77-105) / [_actualizar_grabado](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/services/sync_service.py#63-77)
- Auto-assign `sucursal` from usuario if not provided in batch data

---

### Admin — [api/admin.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/admin.py)

#### [MODIFY] [admin.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/admin.py)
- Register `Sucursal` model with `SucursalAdmin`
- Add `sucursal` to `GrabadoAdmin.list_display` and `list_filter`
- Add `sucursal` to `UsuarioAdmin.list_display`
- Add `tipo_cliente` to `TenantAdmin.list_display`

---

### Dependencies

#### [MODIFY] [requirements.txt](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/requirements.txt)
- Add `openpyxl>=3.1.0`

---

### XLSX Report Service

#### [NEW] [reporte_xlsx_service.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/services/reporte_xlsx_service.py)

`ReporteXLSXService` with methods:
- `generar_xlsx(fecha_inicio, fecha_fin, tenant_id=None, filtro_fecha='sync')` → `BytesIO`
- `_build_queryset(...)` — base queryset with joins
- `_tab_resumen(ws, qs)` — GROUP BY tenant+sucursal → count
- `_tab_resumen_por_dia(ws, qs, fecha_inicio, fecha_fin)` — day-pivot columns
- `_tab_patentes(ws, qs)` — flat detail, 13 columns
- XLSX styling: bold headers, background color, auto-width, total row

---

### API Endpoints

#### [MODIFY] [reportes.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/views/reportes.py)
- Add `formato=xlsx` support to [ReporteDiarioView](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/views/reportes.py#14-86) and [ReporteMensualView](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/views/reportes.py#88-170)
- Add [filtro_fecha](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/tests/test_reportes.py#260-271) query param (`sync`/`creacion`)

#### [NEW] `ReportePlataformaView` in same file
- `GET /api/v1/reportes/plataforma/?fecha_inicio=...&fecha_fin=...&formato=xlsx`
- Admin-only, cross-tenant

#### [MODIFY] [urls.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/urls.py)
- Add `path('reportes/plataforma/', ReportePlataformaView.as_view())`

---

### Celery Tasks

#### [MODIFY] [reportes_tasks.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/tasks/reportes_tasks.py)
- Add `generar_reporte_xlsx_diario()` — runs daily at 01:00 AM Santiago
- Add `generar_reporte_xlsx_mensual()` — runs 1st of each month

---

### Fixtures

#### [MODIFY] [initial.json](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/fixtures/initial.json)
- Add sample `Sucursal` records for demo tenant
- Add `tipo_cliente` to existing tenant fixture

---

### Tests

#### [NEW] [test_reportes_xlsx.py](file:///Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend/api/tests/test_reportes_xlsx.py)

Unit tests for `ReporteXLSXService`:
- Multiple tenants + sucursales aggregation (Tab 1)
- Day-pivot correctness (Tab 2)
- Flat detail with all 13 columns (Tab 3)
- Platform-wide vs single tenant
- Empty date range
- `filtro_fecha=sync` vs `filtro_fecha=creacion`

Integration tests for endpoints:
- XLSX download response headers and content-type
- 403 for operador
- Platform endpoint admin-only access

## Verification Plan

### Automated Tests

Run the existing test suite to verify no regressions from model changes:
```bash
cd /Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend
.venv/bin/python -m pytest api/tests/ -v --tb=short 2>&1 | head -100
```

Run the new XLSX-specific tests:
```bash
cd /Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend
.venv/bin/python -m pytest api/tests/test_reportes_xlsx.py -v --tb=short
```

Run all tests together for full regression check:
```bash
cd /Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend
.venv/bin/python -m pytest api/tests/ -v --tb=short
```
