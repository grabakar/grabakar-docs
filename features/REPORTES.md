# REPORTES — Reporte XLSX Multi-Tab Plataforma

**Fase**: 2
**Estado**: Implementado. Endpoints JSON/CSV per-tenant y XLSX plataforma funcionales.

## Contexto

El sistema genera reportes en formato XLSX con 3 pestañas que cubren **toda la plataforma** (cross-tenant), replicando la estructura de reportes operativos corporativos.

## Alcance

### Ya implementado (per-tenant y plataforma)

- `GET /api/v1/reportes/diario/` — JSON o CSV, filtrado por `fecha_creacion_local`, per-tenant
- `GET /api/v1/reportes/mensual/` — misma lógica para rango mensual
- `GET /api/v1/reportes/plataforma/` — XLSX descargarble, cross-tenant
- Permisos: solo `supervisor` y `admin` (plataforma requiere `admin`)
- Celery task: generadores diarios automatizados


Generación de reportes XLSX multi-tab que replica la estructura del reporte externo actual:

**Tab 1 — Resumen**: agregación por CLIENTE + SUCURSAL con total de patentes.

**Tab 2 — Resumen por dia**: cross-tab con columnas por día del mes, agrupado por CLIENTE + SUCURSAL. Crece a medida que avanza el mes.

**Tab 3 — Patentes**: detalle plano, una fila por grabado. 13 columnas: PATENTE, CANTIDAD, VIN, OT, RESPONSABLE, DISPOSITIVO, FECHA CREADA, FECHA SINCRONIZADA, NOMBRE, EMAIL, CLIENTE, SUCURSAL, TIPO CLIENTE.

## Datos clave

- Los reportes son **month-to-date**: generados diariamente, cubren desde el día 1 del mes hasta el día anterior a la fecha del reporte.
- Todos los registros se agrupan por **`fecha_sincronizacion`** (fecha de recepción en servidor), no por `fecha_creacion_local`. Verificado contra reportes reales.
- Los registros offline (creados meses atrás, sincronizados hoy) aparecen en el día de sync.

## Modelo de datos requerido

El reporte XLSX necesita campos que no existen en el modelo actual:

| Campo | Modelo | Descripción |
|---|---|---|
| `Sucursal` | Nuevo modelo | Sucursal/branch dentro de un Tenant |
| `tipo_cliente` | `Tenant` | Clasificación del cliente (ej: "CONCESIÓN") |
| `responsable_texto` | `Grabado` | Texto libre del operador físico (distinto del usuario logueado) |

## Endpoint

```
GET /api/v1/reportes/plataforma/?fecha=YYYY-MM-DD&tenant_id=N&formato=xlsx
```

- `fecha`: fecha del reporte (default: hoy). Cubre 1ro del mes hasta `fecha - 1`.
- `tenant_id`: opcional. Sin él, genera reporte cross-tenant.
- `formato`: `xlsx` (default).
- Permiso: solo `admin`.
- Response: descarga de archivo XLSX.

## Generación automática

Celery beat: tarea diaria `generar_reporte_xlsx_diario()` a las 01:00 AM (America/Santiago). Genera el reporte month-to-date del día anterior para toda la plataforma.

No se necesita tarea mensual separada — la tarea diaria del día 1 cubre automáticamente el mes anterior completo.

## Notas

- El `ReporteService` existente (per-tenant, JSON/CSV, `fecha_creacion_local`) se mantiene sin cambios.
- El nuevo `ReporteXLSXService` es independiente y usa `fecha_sincronizacion`.
- Requiere `openpyxl` como dependencia.
