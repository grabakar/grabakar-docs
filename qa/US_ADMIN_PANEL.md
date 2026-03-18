# Casos de Uso (User Stories): Panel de Administración (SuperAdmin)

**Módulo**: Plataforma de Administración Core
**Roles involucrados**: `SuperAdmin` (Plataforma)

---

## 1. Gestión de Inquilinos (Tenants)

**US-ADMIN-001: Crear nuevo Tenant**
**Como** SuperAdmin de la plataforma,
**Quiero** registrar una nueva organización (Tenant) definiendo su nombre, colores de marca y configuración,
**Para que** puedan comenzar a utilizar la aplicación GrabaKar con un entorno aislado y personalizado.

**US-ADMIN-002: Editar Branding de un Tenant**
**Como** SuperAdmin de la plataforma,
**Quiero** modificar los colores primarios/secundarios y el logo de un Tenant existente,
**Para que** la aplicación móvil refleje los cambios de imagen corporativa de nuestros clientes.

**US-ADMIN-003: Visualizar listado de Tenants y métricas**
**Como** SuperAdmin de la plataforma,
**Quiero** ver una lista de todos los Tenants activos junto con sus conteos de sucursales, usuarios y grabados del mes en curso,
**Para que** pueda monitorear rápidamente el uso del sistema y el volumen de actividad por cliente.

---

## 2. Gestión de Sucursales

**US-ADMIN-010: Crear una Sucursal para un Tenant**
**Como** SuperAdmin de la plataforma,
**Quiero** dar de alta nuevas sucursales asociándolas obligatoriamente a un Tenant específico,
**Para que** los operadores puedan organizar territorialmente los grabados realizados.

**US-ADMIN-011: Activar/Desactivar Sucursal**
**Como** SuperAdmin de la plataforma,
**Quiero** poder activar o desactivar una sucursal existente,
**Para que** se bloquee la operación o sincronización de registros si la ubicación física deja de operar con nosotros.

---

## 3. Gestión de Usuarios y Roles

**US-ADMIN-020: Crear Operador o Supervisor**
**Como** SuperAdmin de la plataforma,
**Quiero** crear usuarios asignándoles un rol (`operador` o `supervisor`), un `Tenant` y, opcionalmente, una `Sucursal`,
**Para que** el personal del cliente pueda acceder a la plataforma con los permisos y ámbitos de visión correctos.

**US-ADMIN-021: Desactivar Usuario (Revocar Acceso)**
**Como** SuperAdmin de la plataforma,
**Quiero** marcar a un usuario como inactivo (`activo=false`),
**Para que** se le rechace el acceso inmediato tanto en el frontend web como en la app móvil (invalidando sus tokens).

**US-ADMIN-022: Resetear Contraseña de Usuario**
**Como** SuperAdmin de la plataforma,
**Quiero** poder asignar temporalmente una nueva contraseña a cualquier usuario del sistema,
**Para que** pueda asistirlos recuperar su acceso en caso de pérdida de credenciales.

---

## 4. Analíticas y Monitoreo Global

**US-ADMIN-030: Visualizar Dashboard Ejecutivo**
**Como** SuperAdmin de la plataforma,
**Quiero** ver un panel de métricas agrupadas que muestre grabados del día/mes, estado global de la sincronización y tasas de error,
**Para que** pueda asegurar la salud operativa de la plataforma en tiempo real.

**US-ADMIN-031: Analizar Volumen Operativo por Tenant y Tipo**
**Como** SuperAdmin de la plataforma,
**Quiero** ver gráficos comparativos del uso del sistema desglosados por Tenant y por tipo de movimiento (Venta, Demo),
**Para que** podamos entender la adopción del producto y orientar decisiones de negocio.

---

## 5. Auditoría de Grabados (Visión Global)

**US-ADMIN-040: Buscar y Auditar cualquier Grabado de la plataforma**
**Como** SuperAdmin de la plataforma,
**Quiero** visualizar todos los registros de grabados de la base de datos de manera transversal (sin importar el Tenant), aplicando filtros por fecha, patente, estado de sincronización y sucursal,
**Para que** pueda investigar errores reportados o auditar la consistencia de datos generados.

**US-ADMIN-041: Inspeccionar Historial de Impresión de Vidrios**
**Como** SuperAdmin de la plataforma,
**Quiero** ver el detalle anidado de qué vidrios específicos fueron impresos para un Grabado puntual,
**Para que** pueda verificar si los operadores completaron el flujo de grabado correctamente.
