# GLOSARIO — Términos de Dominio

| Término | Definición |
|---------|------------|
| **Patente** | Placa de matrícula del vehículo. En Chile, formato alfanumérico de 6 caracteres (ej: BBDF12). |
| **Grabado** | Proceso de grabar la patente en los vidrios del vehículo mediante ácido o micro-percusión. En el sistema, el registro digital de este proceso. |
| **Vidrio** | Cada ventana/cristal del vehículo donde se graba la patente. Un auto tiene 6 vidrios estándar; una moto, 1. |
| **VIN** | Vehicle Identification Number. Número de chasis de 17 caracteres alfanuméricos que identifica unívocamente un vehículo. |
| **Orden de Trabajo (OT/OC)** | Documento que autoriza el grabado. Puede ser orden de compra del concesionario o solicitud del cliente. |
| **Tenant** | Empresa o marca que usa el sistema. Cada tenant tiene datos, usuarios y branding aislados (arquitectura multi-tenant). |
| **Poka-Yoke** | Mecanismo de prevención de errores. En GrabaKar: mostrar la patente en grande antes de cada grabado para validación visual, más doble digitación obligatoria. |
| **Cola de sincronización** | Registros guardados localmente con `estado_sync: "pendiente"`, esperando envío al servidor cuando haya conectividad. |
| **Token offline** | Credencial local (JWT almacenado en IndexedDB) que permite al operador autenticarse y operar sin conexión a internet. Tiene expiración configurable por tenant. |
| **Impresión** | Acto de imprimir la patente (sticker o calcomanía) que se aplica al vidrio. Se registra cada evento de impresión con contador. |
| **Tipo de movimiento** | Contexto del grabado: `venta` (vehículo nuevo), `demo` (demostración), `capacitacion` (entrenamiento). |
| **Ley 20.580** | Legislación chilena que obliga el grabado de patentes en vidrios vehiculares como medida antirrobo. |
