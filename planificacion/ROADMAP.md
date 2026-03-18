# ROADMAP — Plan de Fases

Estimaciones aproximadas, sujetas a capacidad del equipo.

## Fase 0 — Infraestructura (1 semana)

- Setup repositorios: frontend (React/Vite), backend (Django), docs
- Docker Compose: PostgreSQL, Redis, backend, frontend dev
- CI/CD base: lint, test, build
- Estructura de documentación (este repo)
- Configuración de ambientes: dev, staging

**Entregable**: Repos funcionales con CI verde, docker-compose up levanta todo.

**Dependencias**: Ninguna.

**Estado**: Backend completo (P0-01, P0-04). Frontend pendiente (P0-02, P0-05). Docs existentes (P0-03).

## Fase 1 — MVP Core (3–4 semanas)

- Autenticación: login, JWT, token offline, sesión persistente
- Grabado CRUD: formulario principal, validación, UUID local, Poka-Yoke (doble digitación)
- Flujo multi-vidrio: iteración por vidrio, registro de impresiones, omisión
- Local storage: IndexedDB vía Dexie.js, modelos Grabado + ImpresionVidrio
- Sync API: endpoint de upload, resolución de conflictos, cola de pendientes
- Home: lista de grabados recientes, indicador de pendientes de sync
- White-label base: branding por tenant (logo, colores)

**Entregable**: App funcional offline con sync manual. Operador puede crear grabados, recorrer vidrios, y sincronizar.

**Dependencias**: Fase 0 completada.

**Estado**: Backend completo (P1-01 a P1-05, 37 tests). Frontend pendiente (P1-06 a P1-13).

## Fase 2 — Sync Automático + Reportes (2–3 semanas)

- Auto-sync: detectar conectividad, sincronizar en background
- Reportes per-tenant JSON/CSV: diario y mensual (implementado)
- **Reporte plataforma XLSX**: multi-tab, cross-tenant, month-to-date, agrupado por `fecha_sincronizacion`. Requiere modelo Sucursal + campos nuevos.
- Modelo Sucursal (branch por tenant) + migración
- Celery task diaria para generación automática XLSX
- Dashboard supervisor: vista de operadores, estados de sync, métricas básicas

**Entregable**: Sincronización transparente. Reporte XLSX descargable que replica el formato externo actual.

**Dependencias**: Fase 1 completada. Endpoints de sync estables.

**Estado**: Backend completo (reportes JSON/CSV, XLSX y modelo Sucursal). Frontend pendiente.

## Fase 3 — Bluetooth (2 semanas)

- Integración Ionic/Capacitor para acceso Bluetooth nativo
- Protocolo ESC/POS (primario) + ZPL (secundario)
- PrintService: descubrimiento de impresora, envío de datos, manejo de errores
- UI: selector de impresora, estado de conexión, reimpresión

**Entregable**: Impresión real desde la app en terreno.

**Dependencias**: Fase 2 completada. Requiere Ionic/Capacitor (inicio de migración mobile).

## Fase 4 — Cámara OCR (2 semanas)

- Capacitor Camera plugin para captura de imagen
- OCR: Tesseract.js (local) o API backend para reconocimiento
- ScannerService: captura → procesamiento → sugerencia de patente/VIN
- UI: preview de cámara, resultado OCR editable, confirmación

**Entregable**: Operador puede escanear patente/VIN con la cámara en lugar de tipear.

**Dependencias**: Fase 3 completada (Ionic/Capacitor ya integrado).

## Fase 5 — Mobile Ionic (2–3 semanas)

- Migrar app React a Ionic/Capacitor (si no se hizo en Fase 3)
- Compilar APK/AAB para Android
- Acceso nativo: filesystem, notificaciones, background sync
- Testing en dispositivos reales (tablets Android)

**Entregable**: APK distribuible para tablets de operadores.

**Dependencias**: Fase 4 completada.

## Diagrama de Dependencias

```
Fase 0 → Fase 1 → Fase 2 → Fase 3 → Fase 4 → Fase 5
                                ↑
                    Ionic/Capacitor requerido desde aquí
```
