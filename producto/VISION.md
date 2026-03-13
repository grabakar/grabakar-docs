# VISION — GrabaKar

## Problema

El grabado de patentes en vidrios es un proceso manual con alto riesgo de error, realizado en terreno (estacionamientos subterráneos, concesionarios). Los operadores necesitan operar sin internet por horas o días. No existe trazabilidad confiable del proceso, y los errores de transcripción de patentes generan reprocesos costosos.

## Solución

App offline-first que registra grabados, valida patentes visualmente mediante mecanismo Poka-Yoke (mostrar patente en grande antes de cada grabado), imprime vía Bluetooth, y sincroniza automáticamente cuando hay red. Toda la operación funciona sin conexión; la sincronización es transparente.

## Usuarios

| Rol | Contexto | Necesidad principal |
|-----|----------|---------------------|
| Operador de grabado | Terreno, tablet, sin internet frecuente | Registrar grabados rápido y sin errores |
| Supervisor | Oficina, desktop/tablet, con internet | Ver estado de operaciones, descargar reportes |
| Administrador | Oficina, desktop | Gestionar usuarios, configurar tenant/branding |

## Diferenciadores

- **Offline-first real**: No es "degraded mode". La app funciona completa sin internet. La sincronización es un proceso de fondo, no un requisito.
- **Multi-tenant / White-label**: Cada empresa cliente tiene su branding, usuarios, y datos aislados.
- **Trazabilidad completa**: Cada grabado, cada vidrio, cada impresión queda registrada con timestamps y usuario responsable.
- **Poka-Yoke**: Prevención activa de errores en el flujo de grabado (doble digitación, visualización grande de patente).

## Normativa

Ley 20.580 (Chile): Obliga el grabado de patente en todos los vidrios del vehículo como medida antirrobo. El sistema debe cumplir con los requisitos de registro y trazabilidad derivados de esta ley.

## Roadmap Resumido

| Fase | Alcance | Estimación |
|------|---------|------------|
| 0 — Infraestructura | Setup repos, Docker Compose, CI/CD, docs | 1 semana |
| 1 — MVP Core | Auth, grabado CRUD, multi-vidrio, local storage, sync API | 3-4 semanas |
| 2 — Sync + Reportes | Auto-sync, reportes CSV, dashboard supervisor | 2-3 semanas |
| 3 — Bluetooth | Impresión ESC/POS vía Bluetooth | 2 semanas |
| 4 — Cámara OCR | Escaneo de patente y VIN con cámara | 2 semanas |
| 5 — Mobile Ionic | Exportar a Ionic/Capacitor, compilar Android | 2-3 semanas |

Cada fase es incremental. Detalle completo en [ROADMAP.md](../planificacion/ROADMAP.md).
