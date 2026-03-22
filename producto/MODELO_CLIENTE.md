# Modelo de Cliente — Estructura Comercial

## Definición de Cliente

Un **cliente** en GrabaKar es una empresa que contrata el servicio de grabado de patentes. Cada cliente corresponde a un **Tenant** en el sistema.

Un cliente puede ser un concesionario automotriz, una empresa de grabado independiente, una cadena de talleres, o cualquier organización que necesite registrar y trazar grabados de patentes vehiculares bajo la Ley 20.580.

---

## Estructura Jerárquica

```
Cliente (Tenant)
├── Sucursal 1 (ej: "Casa Matriz Santiago")
│   ├── Operador A
│   └── Operador B
├── Sucursal 2 (ej: "Sucursal Viña del Mar")
│   └── Operador C
└── Sucursal N...
```

### Tenant (Cliente)

El tenant es la unidad comercial y de facturación. Todo dato en el sistema pertenece a exactamente un tenant (aislamiento multi-tenant).

| Atributo | Campo Django | Descripción |
|----------|-------------|-------------|
| Nombre | `nombre` | Razón social o nombre comercial |
| Tipo de cliente | `tipo_cliente` | Categoría (concesionario, taller, etc.) |
| Branding | `logo_url`, `color_primario`, `color_secundario` | White-label por tenant |
| Configuración | `configuracion_json` | Vidrios por tipo de vehículo, opciones personalizables |
| Precio por grabado | `precio_por_grabado` | Tarifa unitaria (facturación) |

### Sucursal

Cada tenant tiene una o más sucursales. La mayoría de los clientes operará con **una sola sucursal**, pero clientes más grandes (cadenas, concesionarios con múltiples sedes) pueden tener varias.

| Atributo | Descripción |
|----------|-------------|
| `nombre` | Identificador de la ubicación (ej: "Sucursal Providencia") |
| `tenant` | A qué cliente pertenece |
| `activa` | Si la sucursal está operativa |

Los grabados se asocian a una sucursal, permitiendo reportes y facturación desglosados por ubicación.

### Usuario (Operador / Supervisor / Admin)

| Rol | Sucursal | Dónde opera |
|-----|----------|-----------------------------|
| **Operador** | Exactamente una (obligatoria) | APK (terreno): grabado, impresión, sync |
| **Supervisor** | Multi-sucursal del tenant | Panel web: supervisión, reportes |
| **Admin (del cliente)** | Alcance tenant | Panel web: gestión de usuarios y metas |

**Operador con acceso al panel**: solo lectura, limitado a su sucursal → `panel_persona=panel_operator_readonly`. Ver `tecnico/RBAC_PANEL_ADMIN.md`.

**Administrador de plataforma GrabaKar**: usuario con `panel_persona=platform_admin` e `is_staff=True`. Permisos globales para crear clientes, sucursales y usuarios en cualquier tenant.

---

## UX — Cliente con una sola sucursal

Muchos clientes tendrán una sola sucursal. La simplificación de navegación (ocultar, renombrar la sección "Sucursales") está planificada en **P3-04** del backlog.

---

## Modelo de Facturación

La facturación es **por grabado realizado**, con tarifa plana por cliente.

| Concepto | Cálculo |
|----------|---------|
| Precio unitario | `Tenant.precio_por_grabado` (configurable) |
| Grabados del periodo | Conteo de grabados sincronizados en el mes |
| Total | `precio_por_grabado × grabados_periodo` |

Periodo: mensual, basado en `fecha_sincronizacion`.

Metas de operador: `Usuario.meta_mensual` (grabados esperados/mes). `0` = sin meta. Permite monitorear productividad y proyectar facturación.

---

## Datos de Contacto y Datos Legales (P3-03)

**Estado**: definición de producto — campos no implementados aún en base de datos.

### Campos propuestos en `Tenant`

| Campo | Tipo Django | Descripción |
|-------|------------|-------------|
| `rut` (o `identificador_fiscal`) | `CharField` único, nullable | RUT de la empresa (formato chileno: `76.123.456-7`). Normalizar al guardar: sin puntos, K mayúscula. |
| `email_contacto` | `EmailField` nullable | Correo de la empresa para facturación, incidencias y avisos. No es el email personal de cada `Usuario`. |
| `telefono_contacto` | `CharField` nullable | Teléfono de empresa (opcional). Formato libre o validado por país. |

### Reglas de negocio

1. El RUT debe ser **único** en el sistema si se usa como identificador fuerte.
2. Cambiar RUT o email de contacto post-alta requiere rol **platform_admin** o doble confirmación.
3. Recuperación de contraseña: actúa sobre el `email` del `Usuario` individual, no sobre el `email_contacto` del tenant.

### Preguntas abiertas (cerrar antes de implementar P3-03)

1. ¿Se usa el RUT como `username` del usuario principal del cliente, o solo se guarda como campo en `Tenant`?
2. ¿Los operadores APK pueden recuperar contraseña por email, o solo vía supervisor/admin?
3. ¿Validación de checksum de RUT es obligatoria en frontend y backend?
4. ¿El RUT aparece en facturas/billing generadas desde el panel?

---

## Relación con el Sistema Técnico

| Concepto de negocio | Entidad técnica | Modelo Django |
|---------------------|----------------|---------------|
| Cliente | Tenant | `api.models.Tenant` |
| Sucursal | Sucursal | `api.models.Sucursal` |
| Operador / Supervisor / Admin | Usuario | `api.models.Usuario` |
| Grabado registrado | Grabado | `api.models.Grabado` |
| Vidrio grabado | Impresión de vidrio | `api.models.ImpresionVidrio` |
