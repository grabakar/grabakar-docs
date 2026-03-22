# BLUETOOTH_IMPRESION — Impresión vía Bluetooth

**Fase**: 3
**Estado**: Implementado en APK Android (multi-vidrio).

## Alcance

Impresión de comprobante térmico por vidrio desde la app, usando una impresora **emparejada** vía Bluetooth Classic (perfil serie **SPP**).

## Protocolo y stack

| Capa | Detalle |
|------|---------|
| Transporte | Bluetooth Classic, socket RFCOMM, UUID estándar SPP `00001101-0000-1000-8000-00805F9B34FB` |
| Carga útil | **ESC/POS** (texto; corte al final si la impresora lo soporta) |
| Plugin Capacitor | `@kduma-autoid/capacitor-bluetooth-printer` (peer declarado para Cap 6; proyecto usa `legacy-peer-deps` con Cap 8) |
| Código app | `grabakar-frontend/src/services/PrintService.ts`, `src/utils/escpos.ts` |
| Configuración | Dexie `configuracion`: `bt_printer_address`, `bt_printer_name`; UI **Menú → Impresora Bluetooth** (`/impresora`) |

**PWA / navegador:** no hay impresión Bluetooth; `imprimirCertificado` no hace envío nativo (el contador de impresiones sigue funcionando).

**Android 12+:** la app solicita `BLUETOOTH_CONNECT` al iniciar (`MainActivity`).

## Comportamiento en flujo multi-vidrio

1. Si hay impresora guardada en ajustes: se llama `connectAndPrint` (conectar → enviar ESC/POS → desconectar). Si falla, se muestra alerta y **no** se incrementa el contador.
2. Si no hay impresora guardada: no se envía nada por Bluetooth y **sí** se incrementa el contador (modo desarrollo / sin hardware).

## Hardware

- La impresora debe estar **emparejada** en Ajustes de Android antes de seleccionarla en la app.
- Impresoras que solo exponen **BLE** (sin Classic SPP) pueden no funcionar con este plugin.

## Secundario (no implementado)

- **ZPL** (Zebra) u otros SDKs de marca quedan fuera de este alcance.

## Referencias en código

- `PrintService.imprimirCertificado` — entrada usada desde `FlujoMultiVidrioPage`.
- Interfaz histórica en [GRABADO_PATENTE.md](GRABADO_PATENTE.md) / [FLUJO_MULTI_VIDRIO.md](FLUJO_MULTI_VIDRIO.md): la API concreta es la de `PrintService.ts` (params con `grabadoUuid`, `patente`, `nombreVidrio`).
