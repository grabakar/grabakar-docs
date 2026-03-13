# CAMARA_OCR — Escaneo OCR de Patente/VIN (Stub)

**Fase**: 4
**Dependencia**: Ionic/Capacitor integrado (Fase 3+).
**Estado**: No iniciado.

## Alcance

Escanear patente y/o VIN con la cámara del dispositivo para autocompletar campos del formulario de grabado.

## Stack Candidato

- **Captura**: Capacitor Camera plugin.
- **OCR local**: Tesseract.js (funciona offline, menor precisión).
- **OCR backend**: API dedicada con modelo entrenado (requiere conectividad).

## Interfaz

`ScannerService` con métodos: `capture()`, `recognize(image)`, `getSuggestion()`. Retorna texto reconocido editable por el operador antes de confirmar.

## Notas

- La decisión local vs backend se tomará en Fase 4 según pruebas de precisión.
- El operador siempre puede editar el resultado del OCR antes de guardar.
