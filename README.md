# GrabaKar — Sistema de Grabado de Patentes

App B2B offline-first para registrar el grabado de patentes vehiculares en vidrios, orientada a operadores en terreno en Chile. Multi-tenant, white-label, con sincronización automática.

## Estructura de Documentación

### producto/

- [VISION.md](producto/VISION.md) — Visión del producto, problema, solución, diferenciadores
- [GLOSARIO.md](producto/GLOSARIO.md) — Glosario de términos de dominio
- [ROLES_USUARIOS.md](producto/ROLES_USUARIOS.md) — Roles, permisos, matriz de acceso

### features/

- [LOGIN_SESION.md](features/LOGIN_SESION.md) — Autenticación y sesión offline
- [GRABADO_PATENTE.md](features/GRABADO_PATENTE.md) — Registro de grabado (formulario principal)
- [FLUJO_MULTI_VIDRIO.md](features/FLUJO_MULTI_VIDRIO.md) — Flujo iterativo por vidrio (Poka-Yoke)
- [SINCRONIZACION_OFFLINE.md](features/SINCRONIZACION_OFFLINE.md) — Estrategia de sync offline-first
- [WHITE_LABELING.md](features/WHITE_LABELING.md) — Multi-tenant y personalización de marca
- [BLUETOOTH_IMPRESION.md](features/BLUETOOTH_IMPRESION.md) — Impresión Bluetooth (stub — Fase 3)
- [CAMARA_OCR.md](features/CAMARA_OCR.md) — Escaneo OCR de patente/VIN (stub — Fase 4)
- [REPORTES.md](features/REPORTES.md) — Reportes XLSX plataforma + JSON/CSV per-tenant (Fase 2)

### tecnico/

- [API_CONTRACTS.md](tecnico/API_CONTRACTS.md) — Contratos de API REST
- [ARQUITECTURA.md](tecnico/ARQUITECTURA.md) — Arquitectura del sistema
- [MODELO_DATOS.md](tecnico/MODELO_DATOS.md) — Modelo de datos (ERD, entidades, índices)
- [SYNC_STRATEGY.md](tecnico/SYNC_STRATEGY.md) — Estrategia de sincronización offline
- [TESTING.md](tecnico/TESTING.md) — Testing y cobertura
- [DEPLOYMENT.md](tecnico/DEPLOYMENT.md) — Deployment y Docker
- [SEGURIDAD.md](tecnico/SEGURIDAD.md) — Seguridad y autenticación

### planificacion/

- [ROADMAP.md](planificacion/ROADMAP.md) — Fases del proyecto y timeline
- [BACKLOG.md](planificacion/BACKLOG.md) — Backlog priorizado por fase

### prompts/

- [AGENT_RULES.md](prompts/AGENT_RULES.md) — Reglas base para agentes AI
- [FRONTEND_AGENT.md](prompts/FRONTEND_AGENT.md) — Prompt de agente frontend
- [BACKEND_AGENT.md](prompts/BACKEND_AGENT.md) — Prompt de agente backend
- [REVIEW_AGENT.md](prompts/REVIEW_AGENT.md) — Prompt de agente de revisión
- `implementation_plans/` — Planes de implementación detallados
- `agents/` — Prompts de agentes paralelos backend

### referencias/

- [CURSOR_WORKFLOW.md](referencias/CURSOR_WORKFLOW.md) — Metodología de trabajo y convenciones
- [LEY_20580.md](referencias/LEY_20580.md) — Referencia Ley 20.580 (stub)

### Estado del backend

- [BACKEND_STATUS_REPORT.md](BACKEND_STATUS_REPORT.md) — Estado actual, endpoints, tests, gaps

## Quick Start

Para comenzar a desarrollar, leer en orden:

1. `prompts/AGENT_RULES.md` — Reglas y contexto del proyecto
2. `planificacion/BACKLOG.md` — Tareas priorizadas
3. El feature spec de tu tarea asignada
