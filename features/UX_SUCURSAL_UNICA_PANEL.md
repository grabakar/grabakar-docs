# UX — Cliente con una sola sucursal (panel admin)

**Backlog:** P3-04
**Superficie:** `grabakar-admin` (panel web).

---

## Problema

Muchos clientes tendrán **una sola sucursal**. En ese caso:

- El menú o sección **«Sucursales»** puede parecer redundante o confuso.
- Los operadores siguen perteneciendo a **una** sucursal; para el usuario del panel puede ser obvio cuál es.

---

## Principios de diseño

1. **No eliminar el modelo:** la sucursal sigue existiendo en datos; solo se adapta la **presentación**.
2. **Evitar clics innecesarios** para ver/editar la única sucursal.
3. **Escalar sin sorpresa:** al agregar una segunda sucursal, la UI debe mostrar el patrón multi-sucursal sin rediseño traumático.

---

## Comportamiento propuesto (borrador)

| Condición | Comportamiento sugerido |
|-----------|-------------------------|
| `count(sucursales del tenant) == 1` | En navegación lateral: renombrar o agrupar bajo **«Mi sede»** / **«Ubicación»** en lugar de listado genérico; o mostrar dashboard del tenant con **card única** con nombre de sucursal y acceso a detalle. |
| `count == 1` | Formularios de alta de operador: **pre-seleccionar** la sucursal única (campo oculto o deshabilitado con etiqueta clara). |
| `count > 1` | Mantener listado y filtros por sucursal como hoy. |

---

## Contenido de ayuda (microcopy)

- Texto corto bajo el nombre del tenant: _«1 sede: [Nombre sucursal]»_ cuando aplique.
- Si el cliente amplía a 2+ sucursales: mensaje de bienvenida opcional la primera vez: _«Ahora puede administrar varias sedes.»_

---

## Fuera de alcance

- Cambiar nombres de rutas URL.
- Eliminar el modelo `Sucursal`.

---

## Criterios de aceptación (producto)

1. Un usuario **admin de cliente** con una sola sucursal completa tareas comunes (ver operadores, metas) sin sentir que «Sucursales» es un módulo vacío.
2. No se pierde la capacidad de editar nombre/estado de la sucursal única.
3. Al pasar de 1 a 2+ sucursales, la UI revela el listado multi-sucursal de forma clara.

---

## Referencias

- [MODELO_CLIENTE.md](../producto/MODELO_CLIENTE.md)
- [BACKLOG.md](../planificacion/BACKLOG.md) — P3-04
