# Registro QA — batería automatizada (2026-03-21)

## Metadatos

| Campo | Valor |
|-------|--------|
| **Alcance** | Verificación GitHub + suite automatizada (backend pytest, OpenAPI, admin build, frontend test/build) |
| **Ambiente** | local |
| **Fecha** | 2026-03-21 |
| **Evidencia** | `qa_reports/2026-03-21T00-29-48Z/` |

---

## 1. GitHub Issues (QA)

| Repo | Consulta | Resultado |
|------|-----------|-----------|
| `grabakar/grabakar-frontend` | `label:qa-auto`, todos los estados | **#1** `[QA-AUTO] TC-MV-010: Imprimir no incrementa contador` → **CLOSED** (2026-03-21) |
| `grabakar/grabakar-frontend` | issues abiertos | **ninguno** |
| `grabakar/grabakar-backend` | `label:qa-auto` | **ningún issue** (repo sin etiquetas QA específicas en esta consulta) |

**Conclusión:** el único bug reportado por QA automatizado en frontend está **corregido y cerrado**. No hay issues `qa-auto` abiertos en frontend.

---

## 2. Backend — `grabakar-backend` (Docker `django`)

| Comando | Resultado |
|---------|-----------|
| `pytest -q` (suite completa) | **4 failed, 76 passed** (exit ≠ 0) |
| `python manage.py spectacular --validate --fail-on-warn` | **OK** (exit 0) |

### Fallos pytest (detalle)

1. `api/tests/test_reportes.py::TestReporteServiceDiario::test_reporte_diario_filtra_por_fecha` — `reporte_hoy["total_grabados"]` esperaba 1, obtuvo 0.
2. `api/tests/test_reportes_xlsx.py::TestReporteXLSXService::test_generar_xlsx_basico` — celda Resumen no coincide con expectativa de layout (`Total` vs `Test Tenant`).
3. `api/tests/test_reportes_xlsx.py::TestReporteXLSXService::test_filtro_fecha_sync_vs_creacion` — conteo por `fecha_sync` no cumple.
4. `api/tests/test_reportes_xlsx.py::TestReporteXLSXService::test_plataforma_cross_tenant` — celda total general `None`.

**Evidencia:** `qa_reports/2026-03-21T00-29-48Z/pytest_full.txt`

**Seguimiento:** issue creado: **grabakar/grabakar-backend#1** — corregir implementación o tests hasta que `pytest -q` quede en verde.

---

## 3. Panel admin — `grabakar-admin`

| Comando | Resultado |
|---------|-----------|
| `npm run build` (`tsc --noEmit && vite build`) | **OK** |
| `npm run lint` | **No ejecutable** — `eslint` no está en `PATH` del script |
| ESLint vía `npx eslint` | **Falló** — proyecto sin `eslint.config.*` compatible con ESLint 10 descargado por npx |

**Evidencia:** `admin_build.txt`, `admin_lint.txt`, `admin_lint_npx.txt`

**Recomendación para QA/dev:** ejecutar `npm install` en `grabakar-admin` y usar `./node_modules/.bin/eslint src` (o alinear `package.json` con flat config ESLint 9+).

---

## 4. App móvil / PWA — `grabakar-frontend`

| Comando | Resultado |
|---------|-----------|
| `npm test` (Vitest) | **OK** — 7 archivos, 26 tests |
| `npm run build` | **OK** |

**Evidencia:** `frontend_vitest.txt`, `frontend_build.txt`

---

## 5. Pendiente para “batería completa” (no automatizado aquí)

- **In-device / APK:** requiere ADB + emulador; seguir [QA_IN_DEVICE_RUNBOOK.md](QA_IN_DEVICE_RUNBOOK.md).
- **Panel web RBAC manual:** seguir [TC_RBAC_PANEL_ADMIN_QA.md](TC_RBAC_PANEL_ADMIN_QA.md) y [REGISTRO_QA_2026_RBAC_OPENAPI.md](REGISTRO_QA_2026_RBAC_OPENAPI.md) para cierre UI.
- **Backend reportes:** resolver los 4 fallos de pytest antes de dar **GO** en reportes/XLSX.

---

## 6. Conclusión

| Área | Estado |
|------|--------|
| Issues QA frontend | Cerrado (#1); sin abiertos |
| OpenAPI | OK |
| RBAC API (subconjunto previo) | OK en ronda anterior; esta batería no re-ejecutó solo RBAC |
| Pytest backend completo | **NO GO** (4 fallos en reportes) |
| Admin build | OK |
| Admin lint | Bloqueado / no verificado |
| Frontend unit + build | OK |

**Veredicto batería automatizada:** **Aprobado con observaciones** — bloqueante para release hasta corregir tests/servicio de reportes en backend.
