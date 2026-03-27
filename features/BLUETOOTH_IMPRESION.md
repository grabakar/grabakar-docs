# BLUETOOTH_IMPRESION — Impresión vía Bluetooth

**Fase**: 3
**Estado**: Implementado en APK Android (pantalla de impresión).

**Referencia técnica detallada (bytes ESC/POS, constantes, tests, límites, roadmap bitmap):** [tecnico/IMPRESION_ESC_POS.md](../tecnico/IMPRESION_ESC_POS.md).

## Alcance

Impresión térmica de patente desde la app, usando una impresora **emparejada** vía Bluetooth Classic (perfil serie **SPP**).

## Protocolo y stack

| Capa | Detalle |
|------|---------|
| Transporte | Bluetooth Classic, socket RFCOMM, UUID estándar SPP `00001101-0000-1000-8000-00805F9B34FB` |
| Carga útil | **ESC/POS** (texto; corte al final si la impresora lo soporta) |
| Plugin Capacitor | `@kduma-autoid/capacitor-bluetooth-printer` (peer declarado para Cap 6; proyecto usa `legacy-peer-deps` con Cap 8) |
| Código app | `grabakar-frontend/src/services/PrintService.ts`, `src/utils/escpos.ts`, `src/pages/ImpresionPage.tsx` |
| Configuración | Dexie `configuracion`: `bt_printer_address`, `bt_printer_name`; UI **Menú → Impresora Bluetooth** (`/impresora`) |

**PWA / navegador:** no hay impresión Bluetooth.

**Android 12+:** la app solicita `BLUETOOTH_CONNECT` al iniciar (`MainActivity`).

## Comportamiento en pantalla de impresión

1. Si hay impresora guardada en ajustes: se llama `connectAndPrint` (conectar → enviar ESC/POS → desconectar). Si falla, se muestra alerta y **no** se incrementa el contador.
2. Si no hay impresora guardada: la app muestra estado de impresora no disponible y permite ir a configuración.
3. Usuario debug `debug_printer`: puede simular impresión sin hardware (incrementa contador sin envío BT), solo para QA/desarrollo.
4. No existe mínimo obligatorio de impresiones por patente (se puede cerrar/finalizar con 1 impresión o más).

## Calibración de tamaño (rollo 55mm)

La pantalla de impresión expone **Imprimir calibración (varios tamaños)**. El ticket **solo contiene la patente** repetida en distintas escalas ESC/POS `GS ! n` (sin títulos ni leyendas en el papel). El orden es de arriba hacia abajo (de menor a mayor escala simétrica, más dos líneas “altas” con ancho menor y alto máximo). Constantes en código: `PATENTE_CALIBRATION_GS_SCALES` en `src/utils/escpos.ts`.

**Impresión normal:** usa el máximo admitido por `GS !` (**8×8**, `0x77`). Si el texto se corta en el ancho útil del rollo, conviene bajar a `0x66` o usar escala asimétrica (p. ej. `0x57`).

Objetivo operativo: maximizar legibilidad dentro del ancho del rollo 55mm; el tamaño exacto en milímetros depende del modelo (resolución de cabezal).

## Hardware

- La impresora debe estar **emparejada** en Ajustes de Android antes de seleccionarla en la app.
- Impresoras que solo exponen **BLE** (sin Classic SPP) pueden no funcionar con este plugin.

## Secundario (no implementado)

- **ZPL** (Zebra) u otros SDKs de marca quedan fuera de este alcance.

## Referencias en código

- `PrintService.imprimirPatente` y `PrintService.imprimirCalibracionPatente`.
- Generación de payload ESC/POS en `src/utils/escpos.ts`.
