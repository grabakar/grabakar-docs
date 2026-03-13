# BLUETOOTH_IMPRESION — Impresión vía Bluetooth (Stub)

**Fase**: 3
**Dependencia**: Ionic/Capacitor integrado (Fase 5 puede adelantarse parcialmente).
**Estado**: No iniciado.

## Alcance

Impresión de patente en sticker/calcomanía desde la app, conectando a impresora portátil vía Bluetooth.

## Protocolo

- **Primario**: ESC/POS (impresoras térmicas portátiles comunes).
- **Secundario**: ZPL (Zebra), si se requiere por hardware del cliente.

## Interfaz

`PrintService` definida en [GRABADO_PATENTE.md](GRABADO_PATENTE.md) y [FLUJO_MULTI_VIDRIO.md](FLUJO_MULTI_VIDRIO.md). Métodos esperados: `discover()`, `connect(deviceId)`, `print(data)`, `disconnect()`.

## Notas

- Requiere Capacitor Bluetooth plugin.
- Fase 1 registra eventos de impresión sin hardware real (mock/stub).
