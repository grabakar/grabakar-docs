# Impresión térmica ESC/POS (APK Android)

Referencia para mantener o extender la impresión de patente por Bluetooth. **Producto**: el ticket físico debe llevar **solo el texto de la patente** (sin UUID, fechas, marca del tenant ni leyendas de calibración en el papel).

**Código fuente (única fuente de verdad de bytes y constantes):**

| Archivo | Rol |
|---------|-----|
| `grabakar-frontend/src/utils/escpos.ts` | Construcción del payload ESC/POS. Constantes: `PATENTE_BITMAP_WIDTH_PX` (280 px = 35 mm @ 203 dpi), `PATENTE_BITMAP_HEIGHT_PX` (200 px), `buildPatenteReceiptBitmap` (ruta principal raster), `buildPatenteReceipt` (fallback texto), `buildPatenteCalibrationReceipt`, `PATENTE_CALIBRATION_GS_SCALES` |
| `grabakar-frontend/src/services/PrintService.ts` | `imprimirPatente` (→ bitmap primero, fallback texto), `imprimirCalibracionPatente` → `BluetoothPrinter.connectAndPrint` |
| `grabakar-frontend/src/pages/ImpresionPage.tsx` | UI, contador `cantidad_impresiones`, usuario `debug_printer` (simula impresión sin BT), botón calibración |
| `grabakar-frontend/src/utils/escpos.test.ts` | Vitest: payload bitmap (GS v 0 header, dimensiones, cut), calibración sin leyendas, constantes de tamaño |

**Documento de producto / alcance Bluetooth:** [features/BLUETOOTH_IMPRESION.md](../features/BLUETOOTH_IMPRESION.md).

---

## 1. Protocolo y transporte

- **Transporte:** Bluetooth Classic, perfil serie **SPP** (RFCOMM).
- **Plugin:** `@kduma-autoid/capacitor-bluetooth-printer` — método usado: `connectAndPrint({ address, data })` donde `data` es un **string** con códigos de byte 0x00–0xFF (`String.fromCharCode`).
- **Plataforma:** solo **Android** en runtime; web/PWA no imprimen por BT.

---

## 2. Contenido permitido en el ticket

- **Impresión normal (`buildPatenteReceipt`):** una sola línea visible: patente en mayúsculas (tras `escposSanitizeLine`), centrada.
- **Calibración (`buildPatenteCalibrationReceipt`):** la misma patente repetida **una vez por cada escala** `GS ! n`. **No** se imprimen títulos, “rollo 55 mm”, etiquetas A/B/C/D, separadores ASCII ni instrucciones. El operador infiere el orden **de arriba hacia abajo** (y el texto de ayuda solo en pantalla en `ImpresionPage`).

---

## 3. Secuencia ESC/POS usada (resumen)

| Bytes | Comando | Uso |
|-------|---------|-----|
| `1B 40` | ESC @ | Inicializar impresora |
| `1B 4D 00` | ESC M 0 | Fuente A |
| `1B 61 01` | ESC a 1 | Alineación centrada |
| `1D 21 n` | GS ! n | Magnificación de carácter (ver §4) |
| `1D 21 00` | GS ! 0 | Reset magnificación (1×1) |
| `1D 56 00` | GS V 0 | Corte de papel (si el hardware lo soporta) |

Tras la línea de patente se envían saltos de línea y luego el corte. El carácter de patente termina en `\n` (0x0A).

---

## 4. Comando `GS ! n` (magnificación)

Convención **Epson-style** usada en el código (compatible con la mayoría de térmicas ESC/POS):

- `n = ((ancho_mult - 1) << 4) | (alto_mult - 1)`
- `ancho_mult` y `alto_mult` ∈ **1 … 8** (enteros).

Ejemplos:

| `n` (hex) | Ancho | Alto | Notas |
|-----------|-------|------|--------|
| `0x33` | 4× | 4× | Simétrico |
| `0x44` | 5× | 5× | Simétrico |
| `0x77` | 8× | 8× | Máximo típico en ambos ejes |
| `0x37` | 4× | 8× | “Alto” sin ensanchar tanto el ancho |
| `0x57` | 6× | 8× | Compromiso ancho/alto |

**Constante de recibo normal:** `PATENTE_RECEIPT_GS_SCALE` en `escpos.ts` (valor por defecto actual: **`0x77`**).

Si en un modelo concreto el texto **se corta** en el ancho útil del rollo (p. ej. patente larga + 8× ancho), bajar el valor (p. ej. `0x66`) o usar escala asimétrica (`0x57`, `0x47`, etc.).

---

## 5. Calibración: orden de líneas

Export `PATENTE_CALIBRATION_GS_SCALES` (orden fijo):

1. Simétricos ascendentes: **`0x33`, `0x44`, `0x55`, `0x66`, `0x77`**
2. Luego dos líneas **altas** (alto 8×): **`0x37`**, **`0x57`**

Cada ítem del arreglo genera: reset `GS ! 0`, aplicar `GS ! code`, línea de patente, reset, salto(s).

Para cambiar el comportamiento: editar solo ese arreglo y/o `PATENTE_RECEIPT_GS_SCALE`, y actualizar tests en `escpos.test.ts`.

---

## 6. Sanitización (`escposSanitizeLine`)

- Elimina caracteres de control que romperían el flujo térmico; recorta a `maxLen` (16 para patente en recibo).
- La patente se muestra en **mayúsculas** en el payload.

---

## 7. Usuario `debug_printer` (QA / desarrollo)

En `ImpresionPage`, si el usuario autenticado tiene `username === 'debug_printer'`, puede completar el flujo de impresión **sin** impresora BT configurada: se simula éxito y se incrementa `cantidad_impresiones`. La calibración **sí** requiere impresora configurada (no usa este bypass en la implementación actual). Ver semilla/backend según entorno.

---

## 8. Limitaciones y siguientes pasos posibles

- **Tamaño máximo con texto ESC/POS:** en la práctica se limita a combinaciones **1×–8×** por eje vía `GS !`. No hay “9×” estándar en este modo.
- **Legibilidad en mm:** depende de DPI del cabezal y ancho printable del rollo (p. ej. 55 mm nominal); no hay conversión fija código → mm en la app.
- **Si se requiere texto más grande que el máximo vectorial:** hay que enviar **gráfico raster** (bitmap de la patente generado en el cliente y comandos `GS ( v 0` / GS `v` 0 según manual del fabricante). Eso implica otro módulo (canvas/offscreen → raster → ESC/POS) y pruebas por modelo.

---

## 9. Pruebas automáticas

```bash
cd grabakar-frontend && npm test -- --run src/utils/escpos.test.ts
```

Los tests verifican presencia de `PATENTE_RECEIPT_GS_SCALE`, que la calibración incluya todos los bytes de `PATENTE_CALIBRATION_GS_SCALES`, y que **no** aparezan cadenas tipo “calibración”, “Preset”, “Rollo”, guiones repetidos en el payload de calibración.

---

## 10. Historial de decisiones (breve)

- Recibo y calibración: **solo patente** en papel; explicaciones solo en UI.
- Escala por defecto del recibo llevada al **máximo `GS !` simétrico** (`0x77`), con documentación de fallback si el hardware corta líneas.
- Calibración: varias escalas + dos modos “altos” para comparar sin texto extra en el ticket.
