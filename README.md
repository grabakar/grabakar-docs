# GrabaKar — Documentación

App B2B offline-first para grabado de patentes vehiculares en Chile (Ley 20.580). Multi-tenant, white-label, con sincronización automática.

**Estado actual**: Fases 0–1 completas. Fase 2 backend completa, frontend pendiente. Ver [ROADMAP](planificacion/ROADMAP.md).

**Regla crítica de impresión**: no existe un mínimo obligatorio de impresiones por patente. Se permite cerrar/finalizar con 1 impresión (o más), y también imprimir tantas veces como el operador necesite.

---

## Índice

### `tecnico/` — Referencia técnica

| Documento | Qué cubre |
|-----------|-----------|
| [ARQUITECTURA.md](tecnico/ARQUITECTURA.md) | Decisiones de arquitectura, stack, diagrama de componentes |
| [MODELO_DATOS.md](tecnico/MODELO_DATOS.md) | ERD, entidades, índices, esquema IndexedDB |
| [API_CONTRACTS.md](tecnico/API_CONTRACTS.md) | Contratos REST API (APK): auth, grabados, sync, reportes |
| [ADMIN_PANEL_API.md](tecnico/ADMIN_PANEL_API.md) | Contratos API panel admin + guía operacional |
| [RBAC_PANEL_ADMIN.md](tecnico/RBAC_PANEL_ADMIN.md) | Matriz RBAC de las 3 personas del panel web |
| [SYNC_STRATEGY.md](tecnico/SYNC_STRATEGY.md) | Estrategia offline-first: sync, conflictos, purga |
| [SEGURIDAD.md](tecnico/SEGURIDAD.md) | JWT, tokens, rate limiting, OWASP, multi-tenant |
| [TESTING.md](tecnico/TESTING.md) | Estrategia de testing, cobertura, herramientas |
| [DEPLOYMENT.md](tecnico/DEPLOYMENT.md) | Docker local, variables de entorno, Dockerfile |
| [GCP.md](tecnico/GCP.md) | Infraestructura GCP, deploy manual, bootstrap, operación VM |
| [DEVOPS_CICD_STRATEGY.md](tecnico/DEVOPS_CICD_STRATEGY.md) | GitHub Actions CI/CD: pipelines, workflows, branch strategy |
| [TECHNICAL_ASSESSMENT.md](tecnico/TECHNICAL_ASSESSMENT.md) | Diagnóstico de calidad, issues resueltos y pendientes |
| [deployments/](tecnico/deployments/) | Runbooks de despliegue específicos |
| [deployments/APK_VARIANTS_PIPELINE.md](tecnico/deployments/APK_VARIANTS_PIPELINE.md) | Pipeline APK dual (Local + GCP), scripts, artifacts y troubleshooting Java 21 |
| [IMPRESION_ESC_POS.md](tecnico/IMPRESION_ESC_POS.md) | Payload ESC/POS, `GS !`, constantes en `escpos.ts`, calibración sin leyendas, tests, límites y bitmap futuro |

---

### `producto/` — Especificación de producto

| Documento | Qué cubre |
|-----------|-----------|
| [VISION.md](producto/VISION.md) | Problema, solución, usuarios objetivo, diferenciadores |
| [ROLES_USUARIOS.md](producto/ROLES_USUARIOS.md) | Roles APK (operador/supervisor/admin), matriz de permisos |
| [MODELO_CLIENTE.md](producto/MODELO_CLIENTE.md) | Tenant → Sucursal → Usuario, facturación, datos legales (P3-03) |
| [GLOSARIO.md](producto/GLOSARIO.md) | Términos de dominio |

---

### `features/` — Especificaciones de feature

| Documento | Estado | Qué cubre |
|-----------|--------|-----------|
| [LOGIN_SESION.md](features/LOGIN_SESION.md) | ✅ | Auth, tokens, sesión offline, auto-refresh |
| [GRABADO_PATENTE.md](features/GRABADO_PATENTE.md) | ✅ | Formulario principal, validaciones, duplicados, anti-fraude |
| [FLUJO_MULTI_VIDRIO.md](features/FLUJO_MULTI_VIDRIO.md) | ✅ | Poka-Yoke por vidrio, impresiones, finalización |
| [SINCRONIZACION_OFFLINE.md](features/SINCRONIZACION_OFFLINE.md) | ✅ backend / ⏳ frontend | Batch upload, conflictos, backoff, purga |
| [WHITE_LABELING.md](features/WHITE_LABELING.md) | ✅ | CSS variables, branding por tenant |
| [REPORTES.md](features/REPORTES.md) | ✅ backend / ⏳ frontend | CSV/JSON diario-mensual, XLSX plataforma |
| [ADMIN_PANEL.md](features/ADMIN_PANEL.md) | ✅ | Especificación del panel web admin |
| [dashboard-print.md](features/dashboard-print.md) | ✅ | Convención: dashboards y billing solo en panel, no en APK |
| [FLUJO_REIMPRESION_APK.md](features/FLUJO_REIMPRESION_APK.md) | ⏳ P3-05/06 | Reimpresión en pantalla final, re-ingreso misma patente |
| [UX_SUCURSAL_UNICA_PANEL.md](features/UX_SUCURSAL_UNICA_PANEL.md) | ⏳ P3-04 | UX simplificada para tenants con una sola sucursal |
| [BLUETOOTH_IMPRESION.md](features/BLUETOOTH_IMPRESION.md) | ✅ APK | Bluetooth SPP, ESC/POS; detalle en [IMPRESION_ESC_POS.md](tecnico/IMPRESION_ESC_POS.md) |
| [CAMARA_OCR.md](features/CAMARA_OCR.md) | 🔲 Fase 4 | Escaneo patente/VIN, Tesseract.js |

---

### `planificacion/` — Roadmap y backlog

| Documento | Qué cubre |
|-----------|-----------|
| [ROADMAP.md](planificacion/ROADMAP.md) | Fases 0–5 con estado actual, mejoras P3.x |
| [BACKLOG.md](planificacion/BACKLOG.md) | Tareas priorizadas con criterios de aceptación y dependencias |

---

### `qa/` — Calidad y testing

| Tipo | Documentos |
|------|-----------|
| **Metodología** | [QA_MASTER_PLAN.md](qa/QA_MASTER_PLAN.md), [QA_AGENT_WORKFLOW.md](qa/QA_AGENT_WORKFLOW.md), [QA_IN_DEVICE_RUNBOOK.md](qa/QA_IN_DEVICE_RUNBOOK.md) |
| **Setup y herramientas** | [QA_AGENT_SETUP.md](qa/QA_AGENT_SETUP.md), [QA_GITHUB_ISSUES.md](qa/QA_GITHUB_ISSUES.md) |
| **Checklist release** | [QA_CHECKLIST_RELEASE.md](qa/QA_CHECKLIST_RELEASE.md) |
| **Casos de prueba** | `TC_*.md` — Login, Grabado, Multi-Vidrio, Sync, Reportes, Seguridad, White-Label, RBAC, Admin Panel, Rendimiento, Usabilidad, Edge Cases, Backend |
| **Registros de ejecución** | `REGISTRO_QA_*.md` — Evidencia de baterías ejecutadas |
| **User stories** | [US_ADMIN_PANEL.md](qa/US_ADMIN_PANEL.md) |
| **Plantilla** | [REGISTRO_QA_PLANTILLA.md](qa/REGISTRO_QA_PLANTILLA.md) |

---

### `prompts/` — Agentes y reglas de desarrollo

| Documento | Qué cubre |
|-----------|-----------|
| [AGENT_RULES.md](prompts/AGENT_RULES.md) | Reglas universales para todos los agentes (PDCA, commits, scope) |
| [BACKEND_AGENT.md](prompts/BACKEND_AGENT.md) | Contexto y reglas para desarrollo Django/DRF |
| [FRONTEND_AGENT.md](prompts/FRONTEND_AGENT.md) | Contexto y reglas para desarrollo React/Dexie offline-first |
| [ONBOARDING_AGENT.md](prompts/ONBOARDING_AGENT.md) | Setup del entorno de desarrollo para nuevos miembros |

> Review de PRs y baterías de QA están en los skills de Claude Code (`/review`, `/qa`).

---

### `referencias/`

| Documento | Qué cubre |
|-----------|-----------|
| [LEY_20580.md](referencias/LEY_20580.md) | Referencia legal — Ley de grabado de patentes (Chile) |

---

## Quick Start para desarrollo

1. Leer `prompts/AGENT_RULES.md`
2. Ver estado actual en `planificacion/ROADMAP.md`
3. Tomar tarea de `planificacion/BACKLOG.md`
4. Leer el feature spec correspondiente en `features/`
5. Consultar `tecnico/API_CONTRACTS.md` y `tecnico/MODELO_DATOS.md`
6. Crear rama `feature/<fase>-<nombre-corto>` y trabajar

## Convenciones

- **Idioma**: Narrativa en español. Identificadores técnicos (campos, endpoints) en inglés o español según convención del código.
- **Commits**: Conventional Commits en español (`feat:`, `fix:`, `chore:`)
- **Ramas**: `feature/<fase>-<nombre-corto>` (ej: `feature/p3-rut-cliente`)
- **Estado en docs**: ✅ implementado · ⏳ pendiente · 🔲 no iniciado
