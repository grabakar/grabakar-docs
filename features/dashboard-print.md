# Multi-Tier Dashboard & Streamlined Engraving Workflow

> **Convención GrabaKar (marzo 2026)**
> Este documento originalmente asumía rutas bajo **`grabakar-frontend`** (APK). La convención acordada es: **dashboards, gráficos, billing y vista tipo factura** se implementan en **`grabakar-admin`** (panel web). La **APK** se centra en flujo de grabado, sesión offline y sincronización; ver [APK_SESION_OFFLINE_INVARIANTE.md](APK_SESION_OFFLINE_INVARIANTE.md).
> Personas del panel: [producto/PANEL_ADMIN_PERSONAS_Y_PERMISOS.md](../producto/PANEL_ADMIN_PERSONAS_Y_PERMISOS.md).
> Plan de documentación: [planificacion/DOCUMENTACION_PLANIFICADA.md](../planificacion/DOCUMENTACION_PLANIFICADA.md).

---

Implement role-based dashboards for three user tiers (admin, supervisor/company, operador) with data-driven analytics, a billing engine, and an enhanced "Print & Next" engraving experience. All UI text in Spanish (Chile). No hardcoded brand names.

**Reimpresión:** en flujo multi-vidrio, la reimpresión solo en la **pantalla final** (no en cada vidrio); ver backlog P3-05.

---

## Proposed Changes

### Backend — Model Changes

#### [MODIFY] [models.py](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/models.py)

Add two fields to support billing and goal tracking:

- `Tenant.precio_por_grabado` — `DecimalField(max_digits=10, decimal_places=2, default=0)`. Price per engraving for billing calculation.
- `Usuario.meta_mensual` — `PositiveIntegerField(default=0)`. Monthly target for operator goal tracking. `0` = no target.

Run `makemigrations` + `migrate` after changes.

---

### Backend — New Dashboard Endpoints

#### [MODIFY] [views/](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/views/__init__.py)

Add two new view files:

#### [NEW] [dashboard_company.py](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/views/dashboard_company.py)

`GET /api/v1/dashboard/empresa/` — Scoped to the logged-in user's tenant (supervisor/admin only).

Returns:
- `grabados_hoy`, `grabados_mes`, `grabados_total` (scoped to tenant)
- `por_operador` — count per operator this month
- `serie_diaria` — daily counts for the current month
- `serie_mes_anterior` — daily counts for the previous month (trend comparison)
- `proyeccion_mensual` — linear extrapolation from current velocity to month-end
- `historial_mensual` — last 6 months totals

#### [NEW] [dashboard_operador.py](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/views/dashboard_operador.py)

`GET /api/v1/dashboard/operador/` — Returns data for the logged-in user (any role).

Returns:
- `grabados_hoy`, `grabados_mes` — user's own counts
- `meta_mensual` — their target
- `proyeccion_fin_mes` — linear projection of where they'll end
- `porcentaje_avance` — `grabados_mes / meta_mensual * 100`
- `serie_diaria` — user's daily output this month

#### [NEW] [billing.py](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/views/billing.py)

`GET /api/v1/billing/resumen/` — Admin-only. Returns billing summary per tenant.

Returns array of:
```json
{
  "tenant_id": 1,
  "tenant_nombre": "AutoMax",
  "precio_por_grabado": 1500.00,
  "grabados_periodo": 320,
  "total_a_pagar": 480000.00
}
```

Query params: `mes` (YYYY-MM, default current month)

#### [MODIFY] [urls.py](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/urls.py)

Register the three new endpoints:
```python
path('dashboard/empresa/', DashboardCompanyView.as_view()),
path('dashboard/operador/', DashboardOperadorView.as_view()),
path('billing/resumen/', BillingResumenView.as_view()),
```

---

### Frontend — Dependencies

#### [MODIFY] [package.json](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/package.json)

Install `recharts` for charting:
```bash
npm install recharts
```

Recharts is lightweight, declarative, built on React components, and has zero config. Perfect for the minimalist charts requirement.

---

### Frontend — Types

#### [MODIFY] [types/models.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/types/models.ts)

Add TypeScript interfaces for the new API responses:
- `DashboardEmpresaData`
- `DashboardOperadorData`
- `BillingResumen`

---

### Frontend — API Service

#### [NEW] [services/DashboardService.ts](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/services/DashboardService.ts)

Fetch functions for the three new endpoints. Uses the existing `AuthService` token pattern.

---

### Frontend — Pages (3 Dashboard Pages)

#### [NEW] [pages/AdminDashboardPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/AdminDashboardPage.tsx)

**Level 1 — System Admin (Master View)**

- KPI cards: global engravings today/month/total, active tenants, active users, sync health
- Area chart: 30-day daily engraving trend
- Bar chart: per-tenant breakdown
- **Billing table**: list all tenants with `precio_por_grabado`, `grabados_periodo`, `total_a_pagar`
- **Print View button**: opens a print-optimized page with tenant logo, billing table formatted as invoice
- Links to existing user management (admin CRUD routes)

#### [NEW] [pages/CompanyDashboardPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/CompanyDashboardPage.tsx)

**Level 2 — Company (Enterprise View)**

- KPI cards: company engravings today/month, operators active
- Line chart: daily engravings this month **overlaid** with previous month (trend comparison)
- Forecast line: dashed linear projection to month-end
- Table: daily service log (scrollable, shows date/operator/count)
- Bar chart: last 6 months history

#### [NEW] [pages/OperadorDashboardPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/OperadorDashboardPage.tsx)

**Level 3 — Operator (Operational View)**

- Large progress ring or gauge showing `grabados_mes / meta_mensual`
- Numeric display: "45 de 100 grabados"
- Projection badge: "Proyección fin de mes: 78" with color (green if on track, yellow if behind, red if <50%)
- Minimalist bar chart: daily output this month

#### [NEW] [pages/PrintInvoicePage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/PrintInvoicePage.tsx)

**Print-optimized invoice page** (triggered from Admin Dashboard):
- Tenant logo at top
- Billing period and date
- Table with company, service count, unit price, total
- Grand total
- `@media print` styles for clean PDF output

---

### Frontend — Routing & Navigation

#### [MODIFY] [App.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/App.tsx)

Add routes:
```tsx
<Route path="/dashboard" element={<PrivateRoute><DashboardRouter /></PrivateRoute>} />
<Route path="/dashboard/admin" element={<PrivateRoute><AdminDashboardPage /></PrivateRoute>} />
<Route path="/dashboard/empresa" element={<PrivateRoute><CompanyDashboardPage /></PrivateRoute>} />
<Route path="/dashboard/operador" element={<PrivateRoute><OperadorDashboardPage /></PrivateRoute>} />
<Route path="/print/factura" element={<PrivateRoute><PrintInvoicePage /></PrivateRoute>} />
```

`DashboardRouter` component: reads `usuario.rol` and redirects to the appropriate dashboard.

#### [MODIFY] [components/AppHeader.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/components/AppHeader.tsx)

Add "Dashboard" navigation item, visible to all authenticated users.

---

### Frontend — Print & Next Enhancement

#### [MODIFY] [pages/FlujoMultiVidrioPage.tsx](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/pages/FlujoMultiVidrioPage.tsx)

The existing flow already operates as a single-step-at-a-time glass printing sequence. Enhancements:
- Combine "Imprimir" + advance into a **single "Imprimir y Siguiente"** button
- Auto-advance to next glass after print
- On last glass: button says "Imprimir y Finalizar"
- Larger touch targets for tablet
- Remove the separate "Siguiente" step — merging it into the print action
- Distraction-free: hide header nav, only show patent + glass name + the button

---

### Frontend — Styles

#### [MODIFY] [index.css](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-frontend/src/index.css)

Add CSS for:
- Dashboard layout (KPI card grid, chart containers)
- Progress ring/gauge component
- Print styles (`@media print`) for invoice page
- Enhanced flujo page — larger button, distraction-free

---

## User Review Required

> [!IMPORTANT]
> **Charting library choice**: I'm proposing `recharts` as the charting library. It's lightweight (~45KB gzipped), React-native, and widely used. Alternative: `chart.js` via `react-chartjs-2` (slightly smaller but less React-idiomatic). Let me know if you have a preference.

> [!IMPORTANT]
> **Billing model**: The plan adds `precio_por_grabado` as a flat rate per engraving on the [Tenant](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/models.py#15-25) model. If you need tiered pricing, volume discounts, or different prices by movement type, please let me know and I'll adjust the model.

> [!IMPORTANT]
> **Monthly target**: `meta_mensual` on [Usuario](file:///c:/Users/jvill/Grabakar/grabakar-infra/repos/grabakar-backend/api/models.py#40-56) would be set via the admin panel (existing admin CRUD). If you want operators to set their own goals, let me know.

---

## Verification Plan

### Automated Tests

**Backend tests** — run with:
```bash
docker compose exec backend pytest api/tests/ -v
```

New test files:
- `test_dashboard_company.py` — test scoped queries, permission checks (only supervisor/admin)
- `test_dashboard_operador.py` — test user-scoped data, projection math
- `test_billing.py` — test pricing calculation, admin-only access

Existing tests in `api/tests/` provide conftest fixtures and patterns to follow.

### Browser Verification

After implementation, I will use the browser tool to:
1. Log in as admin → verify Admin Dashboard loads with KPI cards and charts
2. Log in as supervisor → verify Company Dashboard with scoped data
3. Log in as operador → verify Operator Dashboard with goal tracking
4. Test the Print & Next flow — verify single-button advances through glasses
5. Test the print invoice page renders correctly

### Manual Verification
- Please verify that `docker compose up -d` runs successfully after changes
- If you have test users in different roles, test each dashboard with the appropriate login
- Verify the billing table calculates correctly with your real pricing data
