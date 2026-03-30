# Impresión térmica ESC/POS (APK Android)

Referencia para mantener o extender la impresión de patente. **Producto**: el ticket físico debe llevar **solo el texto de la patente** (sin UUID, fechas, marca del tenant ni leyendas de calibración en el papel). El texto se imprime **espejado horizontalmente** para ser pegado en vidrio y leído desde el exterior.

**Código fuente (única fuente de verdad de bytes y constantes):**

| Archivo | Rol |
|---------|-----|
| `grabakar-frontend/src/utils/escpos.ts` | Construcción del payload ESC/POS bitmap (`GS v 0`). Constantes: `PATENTE_BITMAP_WIDTH_PX`, `PATENTE_BITMAP_HEIGHT_PX`, `IPOS_*` (integrada). Funciones: `buildPatenteReceiptBitmap` (ruta principal raster), `buildPatenteReceipt` (fallback texto BT) |
| `grabakar-frontend/src/services/PrintService.ts` | `imprimirPatente` (detecta tipo de impresora: integrada → plugin IPosPrinter, BT → BluetoothPrinter) |
| `grabakar-frontend/src/plugins/ipos-printer.ts` | Bridge Capacitor → plugin Java `IPosPrinterPlugin` para impresora integrada Q2 |
| `grabakar-frontend/android/.../IPosPrinterPlugin.java` | Plugin nativo: binding AIDL a iPosPrinterService, renderizado bitmap, impresión |
| `grabakar-frontend/src/pages/ImpresionPage.tsx` | UI, contador `cantidad_impresiones`, usuario `debug_printer` (simula impresión sin hardware) |
| `grabakar-frontend/src/utils/escpos.test.ts` | Vitest: payload bitmap, constantes de tamaño |

**Documento de producto / alcance Bluetooth:** [features/BLUETOOTH_IMPRESION.md](../features/BLUETOOTH_IMPRESION.md).

---

## 1. Dos rutas de impresión

La app soporta dos tipos de impresora:

| Tipo | Transporte | Dispositivo | Configuración |
|------|------------|-------------|---------------|
| **Integrada** (`integrated`) | AIDL bind service | MediaTek Q2 con iPosPrinterService | Auto-detectada, sin pairing |
| **Bluetooth** (`bluetooth`) | SPP / RFCOMM | Impresoras externas (Rongta, Xprinter, etc.) | Requiere pairing BT + selección en app |

El tipo se almacena en IndexedDB (`configuracion.printer_type`). `PrintService.ts` despacha al plugin correspondiente.

---

## 2. Impresora integrada Q2 — iPosPrinterService

### 2.1 Hallazgos del dispositivo (2026-03-30)

El dispositivo de producción Q2 (Android 8.1, API 27, armeabi-v7a) tiene impresora térmica integrada controlada por `com.iposprinter.iposprinterservice`. Parámetros confirmados por inspección ADB y reverse-engineering del bytecode DEX (VDEX de `/system/priv-app/IPosPrinterService/`):

| Parámetro | Valor | Fuente |
|-----------|-------|--------|
| **Ancho cabezal** | 384 dots (48 mm @ 203 DPI) | constante en DEX (`const/16 v1, #384`) |
| **DPI** | 203 | Estándar térmico, confirmado por dimensiones |
| **Print depth** | 3 (default) | `cat /sys/devices/printer/printdepth` |
| **Fuente por defecto** | ST_ASC24_12 (12w × 24h dots) | DEX `escPosPrinterInit` bytecode |
| **Service action** | `com.iposprinter.iposprinterservice.IPosPrintService` | `dumpsys package` |
| **AIDL descriptor** | `com.iposprinter.iposprinterservice.IPosPrinterService` | DEX string table |
| **Callback descriptor** | `com.iposprinter.iposprinterservice.IPosPrinterCallback` | DEX string table |

### 2.2 Interfaz AIDL — transaction codes confirmados

| TX code | Método | Firma |
|---------|--------|-------|
| 1 | `printerInit` | `(IPosPrinterCallback)` |
| 5 | `setPrinterPrintFontSize` | `(int fontSize, IPosPrinterCallback)` |
| 6 | `setPrinterPrintAlignment` | `(int alignment, IPosPrinterCallback)` |
| 9 | `printText` | `(String text, IPosPrinterCallback)` |
| **16** | **`sendUserCMDData`** | `(byte[] data, IPosPrinterCallback)` ← **enviar ESC/POS raw** |
| **17** | **`printerPerformPrint`** | `(int feedlines, IPosPrinterCallback)` ← **flush + avance** |

### 2.3 Hallazgos críticos de las pruebas de impresión

Se realizaron **25 iteraciones** de prueba en el dispositivo Q2. Resumen de descubrimientos:

1. **ESC/POS `GS !` (text scaling) NO funciona.** El parser `EscPosCMDParse` del Q2 ignora completamente `GS ! n`. Todos los valores (0x33, 0x55, 0x77) producen texto del mismo tamaño — muy pequeño.

2. **`setPrinterPrintFontSize` AIDL tampoco afecta el tamaño** cuando se envían datos vía `sendUserCMDData`. El estado AIDL y el parser ESC/POS son independientes.

3. **La única ruta que produce texto grande es bitmap raster (`GS v 0`)** enviado vía `sendUserCMDData` (tx 16). El tamaño del texto se controla completamente en el bitmap renderizado.

4. **El ancho del raster DEBE ser exactamente 384 px (48 bytes/fila).** Anchos menores (280, 360) producen datos corruptos/gibberish porque el cabezal espera filas de 48 bytes.

5. **`ESC a 1` (centrado) NO debe usarse con `GS v 0`.** Causa que el raster se desplace y envuelva ("wrap around"). El raster se imprime left-aligned siempre.

6. **Texto espejado horizontalmente** (`canvas.scale(-1, 1, W/2, H/2)`) con offset de **+55 px** en coordenadas canvas para centrar visualmente en el papel. Sin este offset, el texto queda desplazado al margen derecho.

7. **Secuencia de impresión funcional:**
   ```
   tx 1  → printerInit(callback)           // resetear
   tx 16 → sendUserCMDData(payload, cb)     // enviar raster GS v 0
   tx 16 → sendUserCMDData(feed_bytes, cb)  // 4× LF para avance post-impresión
   tx 17 → printerPerformPrint(4, cb)       // flush buffer + avance
   ```

8. **tx 18 (`printRawData`) causa flood de líneas en blanco.** Acepta los datos pero el callback falla y genera cientos de líneas vacías. **NO USAR.**

9. **El callback AIDL** (`IPosPrinterCallback`) requiere implementar `onRunResult(boolean)` y `onReturnString(String)`. Se implementa como un `Binder` stub con descriptor `com.iposprinter.iposprinterservice.IPosPrinterCallback`.

10. **Nodos sysfs** (`/sys/devices/printer/`): `transmitdata`, `printdepth`, `printenable`, `powerenable`, `prn_factorytest`, `prndev_status`, `runlines`, `cleanprintbuf`. El sysfs `transmitdata` **no procesa ESC/POS** — es una interfaz de bajo nivel para el driver del kernel, no para la app.

### 2.4 Parámetros finales del bitmap (confirmados)

| Parámetro | Valor | Razón |
|-----------|-------|-------|
| **Ancho bitmap** | 384 px | Debe coincidir con el cabezal térmico |
| **Alto bitmap** | 100 px | Suficiente para legibilidad, ahorra papel |
| **Ancho objetivo del texto** | 72% de 384 = ~276 px | ≈35 mm sobre papel de 48 mm |
| **Offset X (canvas)** | W/2 + 55 | Compensa el desplazamiento del espejo |
| **Espejo** | `canvas.scale(-1, 1, W/2, H/2)` | Para pegar en vidrio |
| **Fuente** | Monospace bold, anti-alias OFF | Nitidez en impresión térmica 1-bit |
| **Font size** | Auto-fit (empieza en H×0.8, reduce hasta caber) | Adapta a patentes de 6-8 chars |
| **Feed post-impresión** | 4 bytes LF (0x0a) | Espacio visible para cortar |

### 2.5 Mapa de fuentes AIDL (referencia)

| Valor | Fuente | Ancho (dots) | Alto (dots) |
|-------|--------|-------------|-------------|
| 0 | ST_ASC16_8 | 8 | 16 |
| 1 | ST_ASC24_12 | 12 | 24 |
| 2 | ST_ASC32_16 | 16 | 32 |
| 3 | ST_ASC48_24 | 24 | 48 |

---

## 3. Impresora Bluetooth externa (ruta existente)

### 3.1 Protocolo y transporte

- **Transporte:** Bluetooth Classic, perfil serie **SPP** (RFCOMM).
- **Plugin:** `@kduma-autoid/capacitor-bluetooth-printer` — método: `connectAndPrint({ address, data })`.
- **Plataforma:** solo **Android** en runtime.

### 3.2 Secuencia ESC/POS (texto)

| Bytes | Comando | Uso |
|-------|---------|-----|
| `1B 40` | ESC @ | Inicializar impresora |
| `1B 4D 00` | ESC M 0 | Fuente A |
| `1B 61 01` | ESC a 1 | Alineación centrada |
| `1D 21 n` | GS ! n | Magnificación de carácter |
| `1D 21 00` | GS ! 0 | Reset magnificación (1×1) |
| `1D 56 00` | GS V 0 | Corte de papel |

**Constante de recibo:** `PATENTE_RECEIPT_GS_SCALE` = `0x33` (4×4).

### 3.3 Ruta bitmap (Bluetooth)

`buildPatenteReceiptBitmap` genera un raster de 280×200 px (35 mm × 25 mm a 203 DPI) via OffscreenCanvas y lo envía con `GS v 0` empaquetado en el payload BT SPP.

---

## 4. Contenido permitido en el ticket

- **Solo la patente** en mayúsculas, sin UUID, fechas, marca del tenant ni leyendas.
- Texto **espejado horizontalmente** (para lectura desde exterior del vidrio).

---

## 5. Sanitización (`escposSanitizeLine`)

- Elimina caracteres de control; recorta a `maxLen` (16 para patente).
- Patente en **mayúsculas**.

---

## 6. Usuario `debug_printer` (QA / desarrollo)

En `ImpresionPage`, `username === 'debug_printer'` permite completar el flujo **sin** impresora configurada: simula éxito e incrementa `cantidad_impresiones`.

---

## 7. Pruebas automáticas

```bash
cd grabakar-frontend && npm test -- --run src/utils/escpos.test.ts
```

---

## 8. Historial de decisiones

- **2026-03-30:** Descubierto que el Q2 ignora `GS !` text scaling. Solo ruta bitmap (`GS v 0` a 384px ancho) produce texto legible.
- **2026-03-30:** Confirmada secuencia AIDL funcional: tx 1 → tx 16 → tx 16 (feed) → tx 17. tx 18 descartado por causar flood de líneas en blanco.
- **2026-03-30:** Offset de centrado +55px calibrado en pruebas iterativas con el dispositivo Q2.
- **2026-03-30:** Texto espejado horizontalmente confirmado como requisito para aplicación en vidrio.
- **2026-03-30:** `PATENTE_RECEIPT_GS_SCALE` cambiado de `0x77` a `0x33` para ruta BT (texto). No aplica a ruta integrada (bitmap).
- Recibo y calibración: **solo patente** en papel; explicaciones solo en UI.
