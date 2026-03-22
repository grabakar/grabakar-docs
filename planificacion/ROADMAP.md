# ROADMAP — Plan de Fases

## Estado actual (marzo 2026)

| Fase | Alcance | Backend | Frontend | Admin Panel |
|------|---------|:-------:|:--------:|:-----------:|
| **0 — Infraestructura** | Repos, Docker, CI/CD | ✅ | ✅ | ✅ |
| **1 — MVP Core** | Auth, grabado, multi-vidrio, sync API | ✅ | ✅ | ✅ |
| **2 — Sync + Reportes** | Auto-sync, XLSX, dashboard supervisor | ✅ | ⏳ | ✅ |
| **P3.x — Mejoras producto** | RBAC panel, RUT, reimpresión, dashboards | ✅ parcial | ⏳ | ⏳ |
| **3 — Bluetooth** | Impresión ESC/POS nativa | 🔲 | 🔲 | — |
| **4 — Cámara OCR** | Escaneo patente/VIN | 🔲 | 🔲 | — |
| **5 — Mobile Ionic** | APK distribuible | 🔲 | 🔲 | — |

---

## Fase 0 — Infraestructura ✅

- Repositorios: backend (Django), frontend (React/Vite), admin panel, docs
- Docker Compose local: PostgreSQL, Redis, backend, frontend dev
- CI backend: ruff + pytest (GitHub Actions)
- GCP staging: Cloud Run + e2-micro VM + Cloud Storage

**Pendiente (P0-05):** CI/CD frontend y admin panel (GitHub Actions workflows).

---

## Fase 1 — MVP Core ✅

- Autenticación: login, JWT, token offline (3h/7d/72h), sesión persistente
- Grabado CRUD: formulario, validación, UUID local, doble digitación (Poka-Yoke)
- Flujo multi-vidrio: iteración por vidrio, registro de impresiones, omisión
- Local storage: IndexedDB vía Dexie.js (grabados, impresiones, configuración)
- Sync API: endpoint upload, resolución de conflictos (device wins), cola pendientes
- Home: lista de grabados recientes, indicador de sync
- White-label base: branding por tenant (logo, colores vía CSS variables)
- Admin panel: CRUD tenants, sucursales, usuarios, grabados, leyes

---

## Fase 2 — Sync Automático + Reportes

**Backend ✅** — Reportes JSON/CSV/XLSX, Celery tasks, modelo Sucursal, dashboard analytics.

**Frontend ⏳** (pendiente):
- Auto-sync: `SyncManager` detección de conectividad + upload en background (P2-05)
- Dashboard supervisor: métricas de operadores, estados de sync (P2-06)

---

## Mejoras de Producto (P3.x)

Items cross-cutting implementados o pendientes independientemente de la secuencia de fases:

| ID | Descripción | Estado |
|----|-------------|--------|
| P3-01 | RBAC panel admin (3 personas: platform_admin, tenant_admin, panel_operator_readonly) | ✅ |
| P3-02 | Scoping de operador por sucursal en queries | ✅ |
| P3-03 | RUT de cliente, email/teléfono de contacto en Tenant | ⏳ |
| P3-04 | UX simplificada cuando tenant tiene una sola sucursal | ⏳ |
| P3-05 | Reimpresión solo en pantalla final del flujo multi-vidrio | ⏳ |
| P3-06 | Re-ingreso de misma patente para reimpresión | ⏳ |
| P3-07 | Dashboards y billing en admin panel (gráficos, proyección facturación) | ⏳ |
| P5-01 | CI/CD automatizado GitHub → GCP (deploy automático en merge a main) | ⏳ |

---

## Fase 3 — Bluetooth 🔲

- Plugin Capacitor para Bluetooth nativo (`@kduma-autoid/capacitor-bluetooth-printer` ya en package.json)
- Protocolo ESC/POS (primario) + ZPL (secundario)
- `PrintService`: descubrimiento de impresora, envío de datos, manejo de errores
- UI: selector de impresora, estado de conexión, reimpresión

**Dependencia**: requiere Ionic/Capacitor completamente integrado.

---

## Fase 4 — Cámara OCR 🔲

- Capacitor Camera plugin para captura de imagen
- OCR: Tesseract.js (local) o API backend
- `ScannerService`: captura → procesamiento → sugerencia de patente/VIN
- UI: preview, resultado editable, confirmación

**Dependencia**: Fase 3 completada (Capacitor integrado).

---

## Fase 5 — Mobile Ionic 🔲

- Migración completa a Ionic/Capacitor si no se realizó en Fase 3
- Compilar APK/AAB para Android
- Acceso nativo: filesystem, notificaciones, background sync
- Testing en tablets Android reales

**Entregable**: APK distribuible para operadores en terreno.

---

## Diagrama de Dependencias

```
Fase 0 → Fase 1 → Fase 2 → Fase 3 → Fase 4 → Fase 5
                                ↑
                    Ionic/Capacitor requerido desde aquí

P3.x → se puede paralelizar con cualquier fase
```
