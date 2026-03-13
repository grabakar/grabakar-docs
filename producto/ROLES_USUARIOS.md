# ROLES_USUARIOS — Roles y Permisos

## Roles

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
