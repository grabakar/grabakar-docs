# Plan de pruebas manuales -- Panel de administracion (grabakar-admin)

**Version:** 1.0
**Fecha:** 2026-03-30
**Alcance:** Todas las funcionalidades del panel web de administracion, incluyendo autenticacion, RBAC por persona, CRUD de entidades, dashboard, reportes, configuracion y comportamiento responsive.

**Entornos de prueba:**
- Desktop: Chrome (ultima version), Firefox, Safari
- Movil: Chrome Android, Safari iOS (pantallas >= 360px)
- Backend: staging en GCP (`/api/v1/`)

**Usuarios de prueba requeridos:**

| Alias | `panel_persona` | `is_staff` | Notas |
|-------|-----------------|------------|-------|
| U-PLAT | `platform_admin` | true | Acceso total |
| U-TENANT | `tenant_admin` | false | Asociado a un tenant especifico |
| U-OPER | `panel_operator_readonly` | false | Asociado a un tenant y sucursal |
| U-NONE | `none` | false | Usuario APK sin acceso al panel |
| U-INVALID | -- | -- | Credenciales inexistentes |

---

## 1. Autenticacion

### AUTH-01 -- Login exitoso con platform_admin
- **Prioridad:** Alta
- **Precondiciones:** Usuario U-PLAT existe en el backend con `panel_persona=platform_admin`.
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Ingresar username y contrasena de U-PLAT.
  3. Presionar "Iniciar sesion".
- **Resultado esperado:** Se redirige a `/#/` (Dashboard). El sidebar muestra todos los enlaces: Dashboard, Tenants, Sucursales, Usuarios, Grabados, Reportes, Configuracion. En localStorage existen `admin_access_token`, `admin_refresh_token` y `admin_user`.

### AUTH-02 -- Login exitoso con tenant_admin
- **Prioridad:** Alta
- **Precondiciones:** Usuario U-TENANT existe con `panel_persona=tenant_admin`.
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Ingresar credenciales de U-TENANT.
  3. Presionar "Iniciar sesion".
- **Resultado esperado:** Se redirige al Dashboard. Sidebar muestra: Dashboard, Tenants, Sucursales, Usuarios, Grabados. NO muestra Reportes ni Configuracion.

### AUTH-03 -- Login exitoso con panel_operator_readonly
- **Prioridad:** Alta
- **Precondiciones:** Usuario U-OPER existe con `panel_persona=panel_operator_readonly`.
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Ingresar credenciales de U-OPER.
  3. Presionar "Iniciar sesion".
- **Resultado esperado:** Se redirige al Dashboard. Sidebar muestra: Dashboard, Sucursales, Grabados. NO muestra Tenants, Usuarios, Reportes ni Configuracion.

### AUTH-04 -- Login rechazado para usuario sin panel_persona
- **Prioridad:** Alta
- **Precondiciones:** Usuario U-NONE existe con `panel_persona=none`.
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Ingresar credenciales de U-NONE.
  3. Presionar "Iniciar sesion".
- **Resultado esperado:** Se muestra el mensaje "Tu cuenta no tiene acceso al panel de administracion." No se redirige. No se almacenan tokens en localStorage.

### AUTH-05 -- Login con credenciales invalidas
- **Prioridad:** Alta
- **Precondiciones:** Ninguna.
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Ingresar usuario "inexistente" y contrasena "falsa".
  3. Presionar "Iniciar sesion".
- **Resultado esperado:** Se muestra mensaje de error del backend (ej. "Credenciales invalidas" o similar). No se redirige.

### AUTH-06 -- Login con campos vacios
- **Prioridad:** Media
- **Precondiciones:** Ninguna.
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Dejar ambos campos vacios.
  3. Presionar "Iniciar sesion".
- **Resultado esperado:** El formulario HTML impide el envio (atributo `required` en ambos inputs). No se realiza peticion al backend.

### AUTH-07 -- Estado de carga durante login
- **Prioridad:** Media
- **Precondiciones:** Ninguna.
- **Pasos:**
  1. Ingresar credenciales validas.
  2. Presionar "Iniciar sesion" y observar el boton.
- **Resultado esperado:** El boton muestra "Entrando..." y queda deshabilitado (`disabled`) mientras se procesa la peticion.

### AUTH-08 -- Logout
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con cualquier usuario.
- **Pasos:**
  1. Verificar que se esta autenticado (Dashboard visible).
  2. Ejecutar logout (segun mecanismo del header/sidebar).
- **Resultado esperado:** Se eliminan `admin_access_token`, `admin_refresh_token` y `admin_user` de localStorage. Se redirige a `/#/login`.

### AUTH-09 -- Redireccion a login sin sesion
- **Prioridad:** Alta
- **Precondiciones:** No hay tokens en localStorage (sesion limpia).
- **Pasos:**
  1. Navegar directamente a `/#/` (Dashboard).
- **Resultado esperado:** Se redirige automaticamente a `/#/login`.

### AUTH-10 -- Redireccion a login desde ruta protegida
- **Prioridad:** Media
- **Precondiciones:** No hay sesion activa.
- **Pasos:**
  1. Navegar directamente a `/#/grabados`.
- **Resultado esperado:** Se redirige a `/#/login`.

### AUTH-11 -- Persistencia de sesion al recargar pagina
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Estar en el Dashboard.
  2. Recargar la pagina (F5).
- **Resultado esperado:** Se muestra brevemente "Cargando..." y luego el Dashboard. La sesion persiste (no se redirige a login).

### AUTH-12 -- Ruta inexistente redirige al Dashboard
- **Prioridad:** Baja
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Navegar a `/#/ruta-que-no-existe`.
- **Resultado esperado:** Se redirige a `/#/` (Dashboard).

---

## 2. RBAC -- Visibilidad del sidebar por persona

### RBAC-01 -- Sidebar de platform_admin
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Observar los enlaces del sidebar.
- **Resultado esperado:** Se muestran exactamente: Dashboard, Tenants, Sucursales, Usuarios, Grabados, Reportes, Configuracion (7 enlaces).

### RBAC-02 -- Sidebar de tenant_admin
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Observar los enlaces del sidebar.
- **Resultado esperado:** Se muestran: Dashboard, Tenants, Sucursales, Usuarios, Grabados (5 enlaces). NO aparecen Reportes ni Configuracion.

### RBAC-03 -- Sidebar de panel_operator_readonly
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-OPER.
- **Pasos:**
  1. Observar los enlaces del sidebar.
- **Resultado esperado:** Se muestran: Dashboard, Sucursales, Grabados (3 enlaces). NO aparecen Tenants, Usuarios, Reportes ni Configuracion.

### RBAC-04 -- Acceso directo por URL a pagina restringida (operador a tenants)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-OPER.
- **Pasos:**
  1. Navegar directamente a `/#/tenants`.
- **Resultado esperado:** La pagina se renderiza pero el backend responde 403 (Forbidden). Se debe mostrar un error o lista vacia. No se debe mostrar datos de otros tenants.

### RBAC-05 -- Acceso directo por URL a Reportes (tenant_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar directamente a `/#/reportes`.
- **Resultado esperado:** El sidebar no tiene enlace a Reportes. Si se navega por URL, el backend rechaza con 403.

### RBAC-06 -- Acceso directo por URL a Config (tenant_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar directamente a `/#/config`.
- **Resultado esperado:** El sidebar no tiene enlace a Configuracion. Si se navega por URL, se permite lectura pero no escritura (GET ok, POST/PATCH 403).

### RBAC-07 -- Enlace activo en sidebar resaltado
- **Prioridad:** Baja
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Navegar a cada seccion del sidebar.
  2. Verificar que el enlace actual tiene fondo destacado.
- **Resultado esperado:** El enlace de la pagina actual tiene la clase `bg-slate-600 text-white`. Los demas tienen estilo normal.

---

## 3. Dashboard

### DASH-01 -- KPIs visibles para platform_admin
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen grabados en el sistema.
- **Pasos:**
  1. Navegar al Dashboard (`/#/`).
  2. Observar las tarjetas de KPI.
- **Resultado esperado:** Se muestran 8 tarjetas: Grabados hoy, Grabados mes, Total grabados, Tenants, Usuarios activos, Sync pendientes, Sync errores, Tasa exito sync %. Todos los valores son numericos (no null ni undefined).

### DASH-02 -- Grafico de linea (serie 30 dias)
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Existen grabados en los ultimos 30 dias.
- **Pasos:**
  1. Navegar al Dashboard.
  2. Localizar el grafico "Grabados ultimos 30 dias".
- **Resultado esperado:** Se muestra un grafico de linea con ejes X (fechas) e Y (cantidad). El tooltip muestra datos al pasar el cursor/dedo.

### DASH-03 -- Grafico de torta (tipo de movimiento)
- **Prioridad:** Media
- **Precondiciones:** Sesion activa. Existen grabados con distintos tipos de movimiento.
- **Pasos:**
  1. Navegar al Dashboard.
  2. Localizar el grafico "Por tipo de movimiento".
- **Resultado esperado:** Se muestra un grafico circular con etiquetas de tipo y cantidad. Colores distintos para cada segmento.

### DASH-04 -- Grafico de barras (por tenant)
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Existen multiples tenants con grabados.
- **Pasos:**
  1. Navegar al Dashboard.
  2. Localizar el grafico "Grabados por tenant (top 20)".
- **Resultado esperado:** Se muestra un grafico de barras horizontal con nombres de tenant en el eje Y y cantidades en el eje X.

### DASH-05 -- Dashboard con tenant_admin muestra datos filtrados
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar al Dashboard.
  2. Verificar los valores de KPIs.
- **Resultado esperado:** Los KPIs reflejan unicamente los datos del tenant del usuario. No se muestran datos de otros tenants.

### DASH-06 -- Dashboard con panel_operator_readonly
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-OPER.
- **Pasos:**
  1. Navegar al Dashboard.
- **Resultado esperado:** Los KPIs reflejan los datos filtrados por sucursal del operador.

### DASH-07 -- Estado de carga del dashboard
- **Prioridad:** Baja
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Navegar al Dashboard observando el estado inicial.
- **Resultado esperado:** Se muestra "Cargando dashboard..." mientras se obtienen los datos. Desaparece cuando los datos estan listos.

---

## 4. Tenants

### TEN-01 -- Listar tenants (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen al menos 2 tenants.
- **Pasos:**
  1. Navegar a `/#/tenants`.
- **Resultado esperado:** Se muestra una tabla con columnas: Nombre, Tipo, Sucursales, Usuarios, Grabados mes, color primario (muestra de color), enlace "Ver". Todos los tenants del sistema aparecen listados.

### TEN-02 -- Buscar tenant por nombre
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen tenants con nombres distintos.
- **Pasos:**
  1. Navegar a Tenants.
  2. Escribir parte del nombre de un tenant en el campo "Buscar por nombre...".
- **Resultado esperado:** La tabla se filtra mostrando solo los tenants cuyo nombre coincide con la busqueda. La busqueda es en tiempo real (al escribir).

### TEN-03 -- Crear tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Navegar a Tenants.
  2. Presionar "Nuevo tenant".
  3. Completar: Nombre = "Empresa QA Test", Tipo cliente = "concesionario", Color primario = "#FF0000", Color secundario = "#00FF00".
  4. Presionar "Crear".
- **Resultado esperado:** El formulario se cierra. El nuevo tenant aparece en la tabla. Los colores se muestran correctamente en la muestra de color.

### TEN-04 -- Crear tenant sin nombre (validacion)
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Presionar "Nuevo tenant".
  2. Dejar el campo Nombre vacio.
  3. Presionar "Crear".
- **Resultado esperado:** El formulario HTML impide el envio (campo `required`).

### TEN-05 -- Cancelar creacion de tenant
- **Prioridad:** Baja
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Presionar "Nuevo tenant".
  2. Completar algun campo.
  3. Presionar "Cancelar".
- **Resultado esperado:** El formulario se oculta. No se crea ningun tenant.

### TEN-06 -- Ver detalle de tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existe al menos un tenant.
- **Pasos:**
  1. Navegar a Tenants.
  2. Presionar "Ver" en un tenant.
- **Resultado esperado:** Se navega a `/#/tenants/{id}`. Se muestra: breadcrumb (Tenants / NombreTenant), nombre, tipo cliente, conteos de sucursales/usuarios/grabados mes, colores. Se muestra la seccion "Precios" con tarjetas de "Por grabado" y "Por vidrio". Se muestra la lista de sucursales asociadas.

### TEN-07 -- Breadcrumb vuelve a lista de tenants
- **Prioridad:** Baja
- **Precondiciones:** Estar en la pagina de detalle de un tenant.
- **Pasos:**
  1. Presionar "Tenants" en el breadcrumb.
- **Resultado esperado:** Se navega de vuelta a `/#/tenants`.

### TEN-08 -- Tenant sin sucursales muestra estado vacio
- **Prioridad:** Media
- **Precondiciones:** Sesion activa. Existe un tenant sin sucursales.
- **Pasos:**
  1. Navegar al detalle de ese tenant.
- **Resultado esperado:** La seccion Sucursales muestra el mensaje "Sin sucursales".

### TEN-09 -- Paginacion de tenants
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Existen mas de 20 tenants.
- **Pasos:**
  1. Navegar a Tenants.
  2. Observar que se muestran 20 registros.
  3. Presionar "Siguiente".
- **Resultado esperado:** Se cargan los siguientes 20 tenants. El boton "Anterior" se habilita. El total se muestra correctamente.

### TEN-10 -- Tenant_admin ve solo su tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar a `/#/tenants`.
- **Resultado esperado:** La tabla muestra unicamente el tenant al que pertenece U-TENANT. No se listan otros tenants. El boton "Nuevo tenant" NO aparece (solo lectura).

### TEN-11 -- Error al crear tenant duplicado
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Existe un tenant llamado "Empresa Existente".
- **Pasos:**
  1. Intentar crear otro tenant con el mismo nombre.
- **Resultado esperado:** Se muestra un mensaje de error debajo del formulario (texto en rojo).

---

## 5. Precios de tenant

### PREC-01 -- Ver precios sin configurar
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Tenant sin precios configurados en `configuracion_json`.
- **Pasos:**
  1. Navegar al detalle del tenant.
  2. Observar la seccion "Precios".
- **Resultado esperado:** Se muestran dos tarjetas: "Por grabado" con valor "--" y "Por vidrio" con valor "--". Se muestra el boton "Editar".

### PREC-02 -- Editar precios: abrir formulario
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Estar en detalle de tenant.
- **Pasos:**
  1. Presionar "Editar" en la seccion Precios.
- **Resultado esperado:** Las tarjetas de precio se reemplazan por un formulario con dos inputs numericos: "Precio por grabado (CLP)" y "Precio por vidrio (CLP)". Se muestran botones "Guardar" y "Cancelar". Los inputs tienen `inputMode="numeric"` para teclado numerico en movil.

### PREC-03 -- Guardar precios
- **Prioridad:** Alta
- **Precondiciones:** Formulario de precios abierto.
- **Pasos:**
  1. Ingresar precio grabado = 5000.
  2. Ingresar precio vidrio = 800.
  3. Presionar "Guardar".
- **Resultado esperado:** El formulario se cierra. Las tarjetas muestran "$5.000" y "$800" formateados en CLP chileno. El boton muestra "Guardando..." durante la peticion.

### PREC-04 -- Cancelar edicion de precios
- **Prioridad:** Media
- **Precondiciones:** Formulario de precios abierto con valores modificados.
- **Pasos:**
  1. Modificar los valores de precio.
  2. Presionar "Cancelar".
- **Resultado esperado:** Se vuelve a la vista de tarjetas con los valores originales (no los modificados).

### PREC-05 -- Guardar precios vacios (null)
- **Prioridad:** Media
- **Precondiciones:** Tenant tenia precios configurados.
- **Pasos:**
  1. Presionar "Editar".
  2. Borrar ambos campos (dejarlos vacios).
  3. Presionar "Guardar".
- **Resultado esperado:** Se guardan como null. Las tarjetas vuelven a mostrar "--".

### PREC-06 -- Error al guardar precios (fallo de red)
- **Prioridad:** Media
- **Precondiciones:** Formulario de precios abierto. Simular fallo de red (DevTools > Network > Offline).
- **Pasos:**
  1. Ingresar valores y presionar "Guardar".
- **Resultado esperado:** Se muestra mensaje de error en rojo debajo del formulario. El formulario permanece abierto con los valores ingresados.

### PREC-07 -- Precios formateados correctamente
- **Prioridad:** Media
- **Precondiciones:** Tenant con precio_grabado = 15000, precio_vidrio = 1200.
- **Pasos:**
  1. Navegar al detalle del tenant.
  2. Observar las tarjetas de precios.
- **Resultado esperado:** Se muestran "$15.000" y "$1.200" (formato CLP con separador de miles).

### PREC-08 -- Precios en movil (teclado numerico)
- **Prioridad:** Alta
- **Precondiciones:** Dispositivo movil o emulador. Formulario de precios abierto.
- **Pasos:**
  1. Tocar el input de precio grabado.
- **Resultado esperado:** Se abre el teclado numerico (no el teclado completo) gracias a `inputMode="numeric"`.

---

## 6. Sucursales

### SUC-01 -- Listar sucursales (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen sucursales en distintos tenants.
- **Pasos:**
  1. Navegar a `/#/sucursales`.
- **Resultado esperado:** Se muestra tabla con columnas: Nombre, Tenant, Activa, Grabados mes. Se listan todas las sucursales del sistema.

### SUC-02 -- Filtrar sucursales por tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen sucursales en distintos tenants.
- **Pasos:**
  1. Navegar a Sucursales.
  2. Seleccionar un tenant especifico en el dropdown "Todos los tenants".
- **Resultado esperado:** La tabla muestra solo las sucursales del tenant seleccionado.

### SUC-03 -- Crear sucursal
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existe al menos un tenant.
- **Pasos:**
  1. Navegar a Sucursales.
  2. Presionar "Nueva sucursal".
  3. Seleccionar un tenant del dropdown.
  4. Ingresar nombre = "Sucursal QA Test".
  5. Verificar que "Activa" esta marcado.
  6. Presionar "Crear".
- **Resultado esperado:** El formulario se cierra. La nueva sucursal aparece en la tabla con el tenant correcto y estado "Si".

### SUC-04 -- Crear sucursal sin tenant (validacion)
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Formulario de sucursal abierto.
- **Pasos:**
  1. Dejar el dropdown de tenant en "Seleccionar tenant" (valor 0).
  2. Ingresar un nombre.
  3. Presionar "Crear".
- **Resultado esperado:** No se crea la sucursal (la funcion `handleCreate` retorna si `tenant_id` es falsy).

### SUC-05 -- Crear sucursal inactiva
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Abrir formulario de nueva sucursal.
  2. Seleccionar tenant, ingresar nombre.
  3. Desmarcar checkbox "Activa".
  4. Presionar "Crear".
- **Resultado esperado:** La sucursal se crea con estado "No" en la columna Activa.

### SUC-06 -- Sucursales visibles para tenant_admin (filtrado automatico)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar a Sucursales.
- **Resultado esperado:** Solo se muestran las sucursales del tenant de U-TENANT. El boton "Nueva sucursal" esta presente (tenant_admin puede crear).

### SUC-07 -- Sucursales visibles para panel_operator_readonly
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-OPER.
- **Pasos:**
  1. Navegar a Sucursales.
- **Resultado esperado:** Solo se muestra la sucursal asociada al operador. El boton "Nueva sucursal" esta presente en la UI pero el backend rechazara la creacion con 403.

### SUC-08 -- Paginacion de sucursales
- **Prioridad:** Media
- **Precondiciones:** Existen mas de 20 sucursales.
- **Pasos:**
  1. Verificar que se muestran 20 registros por pagina.
  2. Navegar con botones Anterior/Siguiente.
- **Resultado esperado:** La paginacion funciona correctamente. El total se muestra. Los botones se deshabilitan en los extremos.

---

## 7. Usuarios

### USR-01 -- Listar usuarios (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Navegar a `/#/usuarios`.
- **Resultado esperado:** Se muestra tabla con columnas: Usuario, Nombre, Rol, Tenant, Panel, Activo, Ultimo login. Se listan todos los usuarios del sistema.

### USR-02 -- Filtrar usuarios por tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Seleccionar un tenant en el dropdown.
- **Resultado esperado:** Solo se muestran usuarios del tenant seleccionado.

### USR-03 -- Filtrar usuarios por rol
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Seleccionar "Supervisor" en el dropdown de roles.
- **Resultado esperado:** Solo se muestran usuarios con rol supervisor.

### USR-04 -- Filtrar por tenant y rol combinados
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Seleccionar un tenant especifico.
  2. Seleccionar rol "Operador".
- **Resultado esperado:** Solo se muestran operadores del tenant seleccionado.

### USR-05 -- Crear usuario (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existe al menos un tenant con sucursales.
- **Pasos:**
  1. Presionar "Nuevo usuario".
  2. Completar: Username = "qa_test_user", Contrasena = "Test1234!", Nombre completo = "Usuario QA", Email = "qa@test.cl".
  3. Seleccionar tenant y sucursal.
  4. Seleccionar rol = "Operador".
  5. Dejar "Activo" marcado.
  6. Presionar "Crear".
- **Resultado esperado:** El formulario se cierra. El nuevo usuario aparece en la tabla.

### USR-06 -- Platform_admin ve checkbox "Administrador de plataforma (staff)"
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Formulario de nuevo usuario abierto.
- **Pasos:**
  1. Observar los campos del formulario.
- **Resultado esperado:** Existe un checkbox "Administrador de plataforma (staff)". Al marcarlo, `is_staff` se envia como true y `panel_persona` se establece como `platform_admin`.

### USR-07 -- Tenant_admin ve selector de acceso al panel
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT. Formulario de nuevo usuario abierto.
- **Pasos:**
  1. Observar los campos del formulario.
- **Resultado esperado:** Existe un dropdown "Acceso al panel web" con opciones: "Sin panel", "Admin de cliente", "Solo lectura (operador)". NO existe el checkbox de staff.

### USR-08 -- Crear usuario con contrasena corta (validacion)
- **Prioridad:** Media
- **Precondiciones:** Formulario de nuevo usuario abierto.
- **Pasos:**
  1. Ingresar contrasena de 5 caracteres.
  2. Intentar crear.
- **Resultado esperado:** El formulario HTML impide el envio (atributo `minLength={8}`).

### USR-09 -- Sucursales se filtran al cambiar tenant en formulario
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Formulario abierto. Existen al menos 2 tenants con sucursales distintas.
- **Pasos:**
  1. Seleccionar Tenant A. Observar las sucursales disponibles.
  2. Cambiar a Tenant B.
- **Resultado esperado:** El dropdown de sucursales se actualiza mostrando solo las sucursales del Tenant B. El valor de sucursal se resetea a "Sin sucursal".

### USR-10 -- Boton "Nuevo usuario" no visible para panel_operator_readonly
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-OPER.
- **Pasos:**
  1. Navegar a `/#/usuarios` (por URL directa).
- **Resultado esperado:** El sidebar no tiene enlace a Usuarios. Si se accede por URL, el backend responde 403.

### USR-11 -- Paginacion de usuarios
- **Prioridad:** Media
- **Precondiciones:** Existen mas de 20 usuarios.
- **Pasos:**
  1. Verificar paginacion con botones Anterior/Siguiente.
- **Resultado esperado:** Paginacion funcional. Total mostrado. Botones deshabilitados en extremos.

### USR-12 -- Tenant_admin solo ve usuarios de su tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar a Usuarios.
- **Resultado esperado:** Solo se muestran los usuarios del tenant de U-TENANT. El dropdown de tenant no aparece o esta fijo.

---

## 8. Grabados

### GRAB-01 -- Listar grabados (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen grabados sincronizados.
- **Pasos:**
  1. Navegar a `/#/grabados`.
- **Resultado esperado:** Se muestra tabla con columnas: Patente, RUT, VIN, Tipo mov., Operador, Tenant, Fecha, Sync, enlace "Ver". Los RUT se muestran formateados (XX.XXX.XXX-X). El estado sync tiene badge de color (verde=sincronizado, rojo=error, amarillo=pendiente).

### GRAB-02 -- Filtrar grabados por tenant
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Seleccionar un tenant en el dropdown.
- **Resultado esperado:** Solo se muestran grabados del tenant seleccionado.

### GRAB-03 -- Filtrar grabados por estado sync
- **Prioridad:** Alta
- **Precondiciones:** Existen grabados con distintos estados.
- **Pasos:**
  1. Seleccionar "Error" en el dropdown de estados.
- **Resultado esperado:** Solo se muestran grabados con estado_sync = "error" (badge rojo).

### GRAB-04 -- Filtrar grabados por patente
- **Prioridad:** Alta
- **Precondiciones:** Existen grabados.
- **Pasos:**
  1. Escribir una patente conocida en el campo "Patente".
- **Resultado esperado:** Se filtra la tabla mostrando solo grabados que coinciden con la patente ingresada.

### GRAB-05 -- Filtrar grabados por rango de fechas
- **Prioridad:** Alta
- **Precondiciones:** Existen grabados de distintas fechas.
- **Pasos:**
  1. Seleccionar fecha desde = primer dia del mes.
  2. Seleccionar fecha hasta = dia actual.
- **Resultado esperado:** Solo se muestran grabados dentro del rango de fechas seleccionado.

### GRAB-06 -- Combinar multiples filtros
- **Prioridad:** Media
- **Precondiciones:** Existen datos suficientes.
- **Pasos:**
  1. Seleccionar un tenant.
  2. Seleccionar estado "sincronizado".
  3. Ingresar una fecha desde.
- **Resultado esperado:** Los filtros se aplican acumulativamente. Solo se muestran grabados que cumplen todos los criterios.

### GRAB-07 -- Ver detalle de grabado
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa. Existe al menos un grabado.
- **Pasos:**
  1. En la tabla de grabados, presionar "Ver" en un registro.
- **Resultado esperado:** Se navega a `/#/grabados/{uuid}`. Se muestra breadcrumb (Grabados / PATENTE), y los campos: Patente, RUT Cliente (formateado), VIN, Orden trabajo, Responsable, Tenant, Sucursal, Tipo movimiento, Tipo vehiculo, Fecha creacion, Fecha sync, Estado sync (badge), Duplicado (Si/No), Impresiones.

### GRAB-08 -- Detalle de grabado con campos opcionales vacios
- **Prioridad:** Media
- **Precondiciones:** Existe un grabado sin VIN, sin orden de trabajo, sin fecha de sync.
- **Pasos:**
  1. Navegar al detalle de ese grabado.
- **Resultado esperado:** Los campos vacios muestran "--" en lugar de null/undefined/vacio.

### GRAB-09 -- Grabados visibles para tenant_admin (filtro automatico)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Navegar a Grabados.
- **Resultado esperado:** Solo se muestran grabados del tenant de U-TENANT. No se ven grabados de otros tenants.

### GRAB-10 -- Grabados visibles para panel_operator_readonly (filtro por sucursal)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-OPER.
- **Pasos:**
  1. Navegar a Grabados.
- **Resultado esperado:** Solo se muestran grabados de la sucursal asociada a U-OPER.

### GRAB-11 -- Paginacion de grabados
- **Prioridad:** Media
- **Precondiciones:** Existen mas de 20 grabados.
- **Pasos:**
  1. Navegar paginas con botones Anterior/Siguiente.
- **Resultado esperado:** 20 registros por pagina. Total mostrado. Botones deshabilitados en extremos.

### GRAB-12 -- VIN truncado en lista
- **Prioridad:** Baja
- **Precondiciones:** Existe un grabado con VIN largo.
- **Pasos:**
  1. Observar la columna VIN en la tabla de grabados.
- **Resultado esperado:** Se muestran solo los primeros 8 caracteres seguidos de "..." (truncado en la lista, completo en el detalle).

---

## 9. Reportes

### REP-01 -- Acceso a reportes (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Navegar a `/#/reportes`.
- **Resultado esperado:** Se muestra el formulario "Reporte XLSX plataforma" con campos de fecha inicio, fecha fin y boton "Descargar XLSX".

### REP-02 -- Descargar reporte XLSX
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen grabados en el rango de fechas.
- **Pasos:**
  1. Seleccionar fecha inicio = primer dia del mes.
  2. Seleccionar fecha fin = dia actual.
  3. Presionar "Descargar XLSX".
- **Resultado esperado:** Se descarga un archivo `reporte_plataforma_YYYY-MM-DD_YYYY-MM-DD.xlsx`. El boton muestra "Descargando..." durante la descarga. El archivo contiene datos de grabados de todos los tenants.

### REP-03 -- Descargar reporte sin fechas (validacion)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Dejar ambos campos de fecha vacios.
  2. Presionar "Descargar XLSX".
- **Resultado esperado:** Se muestra el mensaje "Indica fecha inicio y fecha fin." en rojo. No se realiza peticion al backend.

### REP-04 -- Descargar reporte con solo una fecha
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Seleccionar solo fecha inicio (dejar fecha fin vacia).
  2. Presionar "Descargar XLSX".
- **Resultado esperado:** Se muestra el mensaje de error pidiendo ambas fechas.

### REP-05 -- Error de red al descargar
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Simular fallo de red.
- **Pasos:**
  1. Seleccionar fechas validas.
  2. Activar modo offline en DevTools.
  3. Presionar "Descargar XLSX".
- **Resultado esperado:** Se muestra "Error al descargar el reporte." El boton vuelve a estado normal.

### REP-06 -- Reportes no accesible para tenant_admin
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Verificar que "Reportes" no aparece en el sidebar.
  2. Navegar por URL directa a `/#/reportes`.
- **Resultado esperado:** El sidebar no tiene el enlace. Si se accede por URL, el backend responde 403 al intentar descargar.

---

## 10. Configuracion / Leyes-Casos

### CFG-01 -- Listar leyes (platform_admin)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existen leyes configuradas.
- **Pasos:**
  1. Navegar a `/#/config`.
- **Resultado esperado:** Se muestra titulo "Configuracion", subtitulo "Leyes / Casos" con descripcion. Se muestra tabla con columnas: Nombre, Descripcion, Activa, boton "Editar". Se muestra boton "Nueva ley / caso".

### CFG-02 -- Crear ley/caso
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Presionar "Nueva ley / caso".
  2. Ingresar Nombre = "Ley QA 12345", Descripcion = "Ley de prueba QA".
  3. Dejar "Activa" marcado.
  4. Presionar "Crear".
- **Resultado esperado:** El formulario se cierra. La nueva ley aparece en la tabla con estado "Si".

### CFG-03 -- Editar ley/caso
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-PLAT. Existe al menos una ley.
- **Pasos:**
  1. Presionar "Editar" en una ley existente.
  2. Cambiar la descripcion.
  3. Presionar "Guardar".
- **Resultado esperado:** El formulario muestra titulo "Editar ley/caso" con los datos precargados. Al guardar, se cierra el formulario y la tabla refleja los cambios.

### CFG-04 -- Desactivar ley/caso
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Existe una ley activa.
- **Pasos:**
  1. Presionar "Editar" en la ley.
  2. Desmarcar checkbox "Activa".
  3. Presionar "Guardar".
- **Resultado esperado:** La tabla muestra "No" en la columna Activa para esa ley.

### CFG-05 -- Cancelar edicion de ley
- **Prioridad:** Baja
- **Precondiciones:** Formulario de edicion abierto.
- **Pasos:**
  1. Modificar campos.
  2. Presionar "Cancelar".
- **Resultado esperado:** El formulario se cierra. Los datos en la tabla no cambian.

### CFG-06 -- Crear ley sin nombre (validacion)
- **Prioridad:** Media
- **Precondiciones:** Formulario de creacion abierto.
- **Pasos:**
  1. Dejar nombre vacio.
  2. Presionar "Crear".
- **Resultado esperado:** El formulario HTML impide el envio (campo `required`).

### CFG-07 -- Estado vacio (sin leyes)
- **Prioridad:** Baja
- **Precondiciones:** Sesion activa con U-PLAT. No existen leyes en el sistema.
- **Pasos:**
  1. Navegar a Configuracion.
- **Resultado esperado:** Se muestra el mensaje "Sin leyes/casos configurados" debajo de la tabla vacia.

### CFG-08 -- Error al guardar ley
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT. Simular fallo de red.
- **Pasos:**
  1. Abrir formulario de creacion o edicion.
  2. Activar modo offline.
  3. Presionar "Crear" o "Guardar".
- **Resultado esperado:** Se muestra "Error al guardar" en rojo.

### CFG-09 -- Configuracion no accesible para tenant_admin
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa con U-TENANT.
- **Pasos:**
  1. Verificar que "Configuracion" no aparece en el sidebar.
  2. Navegar por URL directa a `/#/config`.
- **Resultado esperado:** El sidebar no tiene el enlace. Si se accede por URL, se pueden ver las leyes (GET permitido) pero no crearlas ni editarlas (POST/PATCH devuelve 403).

---

## 11. Responsive / Movil

### MOB-01 -- Login en pantalla movil
- **Prioridad:** Alta
- **Precondiciones:** Pantalla de 360px de ancho (usar DevTools o dispositivo real).
- **Pasos:**
  1. Navegar a `/#/login`.
  2. Verificar que el formulario es visible y usable.
- **Resultado esperado:** El formulario se centra en pantalla. Los inputs ocupan el ancho completo. El boton es lo suficientemente grande para presionar con el dedo.

### MOB-02 -- Dashboard responsive
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa. Vista movil (360px).
- **Pasos:**
  1. Navegar al Dashboard.
  2. Verificar KPIs y graficos.
- **Resultado esperado:** Las tarjetas de KPI se reorganizan en 2 columnas (en vez de 8). Los graficos se adaptan al ancho. No hay desbordamiento horizontal.

### MOB-03 -- Tablas con scroll horizontal en movil
- **Prioridad:** Alta
- **Precondiciones:** Vista movil. Navegar a cualquier tabla (Tenants, Sucursales, Usuarios, Grabados).
- **Pasos:**
  1. Verificar que la tabla se puede desplazar horizontalmente.
- **Resultado esperado:** Las tablas permiten scroll horizontal si son mas anchas que la pantalla. No se rompe el layout.

### MOB-04 -- Sidebar en movil
- **Prioridad:** Alta
- **Precondiciones:** Vista movil.
- **Pasos:**
  1. Observar el sidebar.
- **Resultado esperado:** El sidebar es visible (ancho fijo de 14rem / 224px). Verificar que no oculta el contenido principal.

### MOB-05 -- Formulario de precios en movil
- **Prioridad:** Alta
- **Precondiciones:** Vista movil. Estar en detalle de tenant.
- **Pasos:**
  1. Presionar "Editar" en precios.
  2. Verificar inputs y botones.
- **Resultado esperado:** Los inputs tienen padding generoso (py-3) para ser tocables. Los botones Guardar/Cancelar ocupan 50% cada uno (flex-1). El teclado numerico se abre al tocar los inputs.

### MOB-06 -- Filtros de grabados en movil
- **Prioridad:** Media
- **Precondiciones:** Vista movil. Navegar a Grabados.
- **Pasos:**
  1. Verificar los filtros (dropdowns, campo de patente, selectores de fecha).
- **Resultado esperado:** Los filtros se reorganizan en multiples filas (flex-wrap). Son tocables y usables en pantalla pequena.

---

## 12. Casos borde y manejo de errores

### EDGE-01 -- Tabla sin resultados (filtro sin coincidencias)
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Navegar a Grabados.
  2. Filtrar por patente "ZZZZZZZ" (inexistente).
- **Resultado esperado:** La tabla se muestra vacia (sin filas en tbody). No se muestra error. El encabezado de la tabla permanece visible.

### EDGE-02 -- Detalle de grabado con UUID inexistente
- **Prioridad:** Media
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Navegar a `/#/grabados/uuid-que-no-existe-12345`.
- **Resultado esperado:** Se muestra "Cargando..." y luego un error o se queda en carga. No se rompe la aplicacion (no hay pantalla blanca).

### EDGE-03 -- Detalle de tenant con ID inexistente
- **Prioridad:** Media
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Navegar a `/#/tenants/99999`.
- **Resultado esperado:** Se muestra estado de carga y luego maneja el error del backend (404). No se rompe la aplicacion.

### EDGE-04 -- Doble clic en boton de creacion
- **Prioridad:** Media
- **Precondiciones:** Formulario de creacion abierto (tenant, sucursal, usuario o ley).
- **Pasos:**
  1. Completar el formulario.
  2. Presionar el boton "Crear" rapidamente dos veces.
- **Resultado esperado:** Solo se envia una peticion. El boton se deshabilita (`disabled`) durante la peticion (`isPending`). No se crean registros duplicados.

### EDGE-05 -- Token expirado (sesion vencida)
- **Prioridad:** Alta
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Abrir DevTools > Application > Local Storage.
  2. Modificar `admin_access_token` a un valor invalido.
  3. Navegar a otra pagina.
- **Resultado esperado:** El backend responde 401. La aplicacion deberia manejar el error (idealmente redirigir a login o mostrar error de sesion expirada).

### EDGE-06 -- Backend no disponible
- **Prioridad:** Media
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Activar modo offline en DevTools.
  2. Navegar entre secciones.
- **Resultado esperado:** Se muestran estados de carga y eventualmente errores. La aplicacion no se congela ni se rompe. Los datos cacheados por TanStack Query pueden mostrarse temporalmente.

### EDGE-07 -- Navegacion con boton atras del navegador
- **Prioridad:** Media
- **Precondiciones:** Sesion activa.
- **Pasos:**
  1. Navegar de Dashboard a Tenants.
  2. Entrar al detalle de un tenant.
  3. Presionar el boton "Atras" del navegador.
- **Resultado esperado:** Se vuelve a la lista de Tenants. Presionar atras nuevamente vuelve al Dashboard.

### EDGE-08 -- Caracteres especiales en busqueda de tenant
- **Prioridad:** Baja
- **Precondiciones:** Sesion activa con U-PLAT.
- **Pasos:**
  1. Navegar a Tenants.
  2. Escribir `<script>alert('xss')</script>` en el campo de busqueda.
- **Resultado esperado:** El texto se trata como texto plano. No se ejecuta ningun script. La busqueda simplemente no encuentra resultados.

### EDGE-09 -- Valores negativos en precios
- **Prioridad:** Media
- **Precondiciones:** Formulario de precios abierto.
- **Pasos:**
  1. Ingresar -1000 en el campo de precio grabado.
  2. Presionar "Guardar".
- **Resultado esperado:** El backend deberia rechazar valores negativos con un mensaje de error apropiado, o la UI deberia impedirlo.

### EDGE-10 -- Recarga de pagina en detalle
- **Prioridad:** Media
- **Precondiciones:** Sesion activa. Estar en `/#/tenants/5` o `/#/grabados/{uuid}`.
- **Pasos:**
  1. Recargar la pagina (F5).
- **Resultado esperado:** La pagina de detalle se recarga correctamente, obteniendo los datos del backend. No se redirige a otra pagina.

---

## Resumen de cobertura

| Area | Casos | Alta | Media | Baja |
|------|-------|------|-------|------|
| Autenticacion | 12 | 6 | 4 | 2 |
| RBAC Sidebar | 7 | 4 | 0 | 3 |
| Dashboard | 7 | 3 | 3 | 1 |
| Tenants | 11 | 5 | 4 | 2 |
| Precios | 8 | 3 | 4 | 1 |
| Sucursales | 8 | 4 | 3 | 1 |
| Usuarios | 12 | 6 | 4 | 2 |
| Grabados | 12 | 6 | 4 | 2 |
| Reportes | 6 | 3 | 3 | 0 |
| Configuracion | 9 | 3 | 4 | 2 |
| Responsive | 6 | 4 | 2 | 0 |
| Casos borde | 10 | 1 | 7 | 2 |
| **Total** | **108** | **48** | **42** | **18** |

---

## Notas para el tester

1. **Prioridad de ejecucion:** Ejecutar primero todos los casos de prioridad Alta. Luego Media. Los de prioridad Baja pueden posponerse si hay restriccion de tiempo.
2. **Isolation multi-tenant:** En cada prueba con filtrado de datos, verificar que NUNCA se muestran datos de otro tenant. Este es un requisito de seguridad critico.
3. **Texto en espanol:** Verificar que TODOS los textos visibles estan en espanol (es-CL). Ningun mensaje en ingles al usuario.
4. **Marca:** Verificar que no aparezca el nombre "GrabaKar" hardcodeado en contextos de usuario final. (Nota: actualmente el sidebar y login muestran "GrabaKar Admin" -- esto es un hallazgo potencial segun la regla de no hardcodear marca.)
5. **Dispositivos moviles:** Las pruebas MOB-* son obligatorias ya que el panel se usa frecuentemente desde celular.
