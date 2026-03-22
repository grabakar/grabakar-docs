# GLOSARIO — Términos de Dominio

| Término | Definición |
|---------|------------|
| **Patente** | Placa de matrícula del vehículo. En Chile, formato alfanumérico de 6 caracteres (ej: BBDF12). |
| **Grabado** | Proceso de grabar la patente en los vidrios del vehículo mediante ácido o micro-percusión. En el sistema, el registro digital de este proceso. |
| **Vidrio** | Cada ventana/cristal del vehículo donde se graba la patente. Un auto tiene 6 vidrios estándar; una moto, 1. |
| **VIN** | Vehicle Identification Number. Número de chasis de 17 caracteres alfanuméricos que identifica unívocamente un vehículo. |
| **Orden de Trabajo (OT/OC)** | Documento que autoriza el grabado. Puede ser orden de compra del concesionario o solicitud del cliente. |
| **Tenant / Cliente** | Empresa que contrata el servicio de grabado. Cada tenant tiene datos, usuarios, sucursales y branding aislados (arquitectura multi-tenant). Es la unidad de facturación. Ver [MODELO_CLIENTE.md](MODELO_CLIENTE.md). |
| **Sucursal** | Ubicación física de un tenant (ej: sede, local, concesionario). Un tenant puede tener una o más sucursales. Los grabados se asocian a una sucursal. |
| **Operador** | Usuario de campo que ingresa patentes e imprime en la APK. Pertenece a **una sola sucursal**; no edita datos en el panel salvo que se defina una vista de solo lectura. Ver [PANEL_ADMIN_PERSONAS_Y_PERMISOS.md](PANEL_ADMIN_PERSONAS_Y_PERMISOS.md). |
| **Administrador de plataforma** | Personal de GrabaKar con permisos globales (crear clientes, sucursales, usuarios; ver todo). No confundir con el admin del cliente. |
| **Administrador de cliente** | Usuario del tenant con permisos de gestión sobre su empresa (sucursales, operadores, metas) dentro de su cliente, sin ver otros tenants. |
| **Precio por grabado** | Tarifa unitaria cobrada al tenant por cada grabado realizado. Base del modelo de facturación. |
| **Meta mensual** | Cantidad de grabados esperados de un operador en el mes. Configurada por admin del tenant vía panel. Permite seguimiento de productividad y proyecciones. |
| **Poka-Yoke** | Mecanismo de prevención de errores. En GrabaKar: mostrar la patente en grande antes de cada grabado para validación visual, más doble digitación obligatoria. |
| **Cola de sincronización** | Registros guardados localmente con `estado_sync: "pendiente"`, esperando envío al servidor cuando haya conectividad. |
| **Token offline** | Credencial local (JWT almacenado en IndexedDB) que permite al operador autenticarse y operar sin conexión a internet. Tiene expiración configurable por tenant. |
| **Impresión** | Acto de imprimir la patente (sticker o calcomanía) que se aplica al vidrio. Se registra cada evento de impresión con contador. |
| **Tipo de movimiento** | Contexto del grabado: `venta` (vehículo nuevo), `demo` (demostración), `capacitacion` (entrenamiento). |
| **Ley 20.580** | Legislación chilena que obliga el grabado de patentes en vidrios vehiculares como medida antirrobo. |
