# Agente Frontend — grabakar-frontend

Lee `AGENT_RULES.md` primero. Estas son reglas adicionales específicas para el frontend.

> **Implementation Plan:** Read [FRONTEND_IMPLEMENTATION.md](implementation_plans/FRONTEND_IMPLEMENTATION.md) for exact step-by-step implementation instructions, code templates, component structures, and testing checklists for every frontend task.

## Stack

- React 18+ con TypeScript estricto
- Vite como bundler
- Dexie.js para IndexedDB (toda persistencia local)
- React Router para navegación
- Service Worker para PWA y soporte offline
- Testing: Vitest + React Testing Library

## Estructura de Carpetas

```
src/
  pages/          # Vistas/rutas principales
  components/     # Componentes reutilizables
  services/       # Lógica de comunicación (API, sync, auth)
  utils/          # Funciones puras auxiliares
  hooks/          # Custom hooks
  types/          # Interfaces y tipos TypeScript
```

## Estado

- **Global (React Context):** auth (usuario, token, tenant), sync (estado de cola, conexión), theme (white-label).
- **Local:** formularios, UI transiente. Usar `useState`/`useReducer` directo.
- No usar Redux ni Zustand. Context es suficiente para esta app.

## Estilos

- CSS variables para theming white-label. El tenant define colores, logo, nombre.
- Diseño **tablet-first** (dispositivo principal del técnico en terreno).
- Responsive: tablet > mobile > desktop.
- Sin frameworks CSS pesados. CSS modules o utility classes propias.

## Persistencia Local

- Dexie.js para TODA persistencia offline: grabados pendientes, cola de sync, datos de sesión.
- Nunca asumir conectividad. Todo flujo crítico debe funcionar offline.
- La sincronización con el backend es eventual y gestionada por `SyncManager`.

## Interfaces Clave

- **SyncManager** — gestiona cola offline, reintentos, resolución de conflictos.
- **AuthService** — login, refresh token, logout, datos de sesión.
- **PrintService** (stub por ahora) — futura integración con impresora de certificados.
- **ScannerService** (stub por ahora) — futura integración con cámara/scanner de patentes.

## Testing

- Vitest + React Testing Library.
- Testear lógica de negocio a fondo: validaciones, transformaciones, sync logic.
- Mockear TODAS las llamadas API usando los contratos de `grabakar-docs/tecnico/API_CONTRACTS.md`.
- No testear implementación interna de componentes. Testear comportamiento visible.

## Reglas Específicas

- Todo texto visible al usuario en español (Chile). Sin hardcodear nombres de marca.
- Importar tipos desde `src/types/`. No definir tipos inline en componentes.
- Cada página es un componente en `src/pages/` que ensambla componentes de `src/components/`.
- Los `services/` nunca importan componentes React. Son lógica pura.
- Manejar errores de red con feedback al usuario y fallback offline.

## Documentación de Referencia

- Specs de features: `grabakar-docs/features/`, `grabakar-docs/planificacion/BACKLOG.md`
- Contratos API (para mockear): `grabakar-docs/tecnico/API_CONTRACTS.md`
- Modelo de datos (para tipos): `grabakar-docs/tecnico/MODELO_DATOS.md`
