# ROLES_USUARIOS — Roles y Permisos

Este documento describe los **roles en la aplicación móvil / APK** (`operador`, `supervisor`, `admin` en el modelo `Usuario.rol`).

La **personas del panel web** (administrador de plataforma GrabaKar, administrador de cliente, operador con acceso solo lectura al panel) son un eje distinto: ver [PANEL_ADMIN_PERSONAS_Y_PERMISOS.md](PANEL_ADMIN_PERSONAS_Y_PERMISOS.md) y el stub técnico [tecnico/RBAC_PANEL_ADMIN.md](../tecnico/RBAC_PANEL_ADMIN.md).

**Regla de pertenencia:** cada **operador** debe pertenecer a **una sucursal** (cliente con varias sedes sigue teniendo operadores acotados a una sucursal cada uno). Ver [MODELO_CLIENTE.md](MODELO_CLIENTE.md).

---

## Roles (APK y API)

### operador

Técnico en terreno con tablet. Opera en estacionamientos, concesionarios, o en ruta. Frecuentemente sin internet.

- Crear registros de grabado
- Ejecutar flujo multi-vidrio
- Imprimir patentes (Bluetooth, Fase 3+)
- Ver sus propios registros
- Sincronizar manualmente

### supervisor

Jefe de operaciones en oficina. Monitorea la productividad y calidad del equipo.

- Todo lo de operador
- Ver registros de todos los operadores del tenant
- Descargar reportes CSV (diario, mensual)
- Dashboard de estado de sincronización

### admin

Gerente o TI de la empresa cliente. Configura el sistema para su organización.

- Todo lo de supervisor
- Gestionar usuarios (crear, desactivar, cambiar rol)
- Configurar tenant: branding, logo, colores, nombre
- Configurar lista de vidrios por tipo de vehículo
- Configurar expiración de token offline

## Matriz de Permisos

| Acción | operador | supervisor | admin |
|--------|:--------:|:----------:|:-----:|
| Crear grabado | ✓ | ✓ | ✓ |
| Flujo multi-vidrio | ✓ | ✓ | ✓ |
| Imprimir (Bluetooth) | ✓ | ✓ | ✓ |
| Ver registros propios | ✓ | ✓ | ✓ |
| Sync manual | ✓ | ✓ | ✓ |
| Ver registros de otros operadores | — | ✓ | ✓ |
| Descargar reportes CSV | — | ✓ | ✓ |
| Dashboard supervisor | — | ✓ | ✓ |
| Gestionar usuarios | — | — | ✓ |
| Configurar tenant/branding | — | — | ✓ |
| Configurar vidrios por tipo vehículo | — | — | ✓ |
| Configurar expiración token offline | — | — | ✓ |

---

## Panel web vs APK

| Superficie | Usuarios | Notas |
|------------|----------|--------|
| APK / PWA | Principalmente **operadores**; también supervisor/admin si usan la app | Sesión offline obligatoria para viabilidad en terreno: [APK_SESION_OFFLINE_INVARIANTE.md](../features/APK_SESION_OFFLINE_INVARIANTE.md) |
| Panel (`grabakar-admin`) | Dueños, supervisores, **administrador de plataforma** | Analíticas, facturación, gestión de clientes; no es la app de campo |

---

## Backlog — Matriz de permisos del panel

Implementación detallada de RBAC (endpoint × persona) está planificada en [BACKLOG.md](../planificacion/BACKLOG.md) (P3-01, P3-08).
