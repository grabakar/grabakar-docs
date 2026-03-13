# CURSOR_WORKFLOW — Metodología de Trabajo

## Ciclo PDCA

Orden de operación: **CHECK → ACT → PLAN → DO**.

1. **CHECK**: Verificar estado actual — leer código, tests, linter, git status.
2. **ACT**: Corregir problemas encontrados antes de avanzar.
3. **PLAN**: Definir qué se va a hacer, qué archivos se tocan, qué criterios de aceptación.
4. **DO**: Implementar. Commits atómicos por paso lógico.

## Estilo de Código

- Minimal: menos líneas posibles sin sacrificar legibilidad.
- Sin bloat: no agregar código "por si acaso".
- Sin comentarios narrativos: no explicar lo que el código ya dice. Comentarios solo para intención no obvia, trade-offs, o constraints.
- Logs solo para debugging activo. Eliminar antes de commit.

## Herramientas

- `@mantic` para navegación de código y búsqueda semántica.
- `@exa` para referencias externas y documentación de librerías.
- `grep`/`rg` para búsquedas exactas de texto.

## Debugging

- Trazar outputs hacia su origen. Análisis vertical, no horizontal.
- Leer el error completo antes de actuar.
- Reproducir antes de arreglar.

## Documentación

- Repo separado (`grabakar-docs/`) de los repos de código.
- Documentos funcionales, no narrativos.
- Cada feature tiene su spec en `features/`.
- Specs incluyen: reglas de negocio, flujo, modelos, validaciones, criterios de aceptación, casos edge.

## Organización de Proyecto

- Repos separados: frontend, backend, docs.
- Carpetas del workspace: `utils/`, `scripts/`, `prompts/`.
- Preferir pocos archivos con más funcionalidad sobre muchos archivos con poca.

## Git

- **Branch naming**: `feature/<fase>-<nombre-corto>` (ej: `feature/f1-grabado-crud`)
- **Commits**: Conventional commits en español.
  - `feat: agregar formulario de grabado`
  - `fix: corregir normalización de patente`
  - `docs: agregar spec de multi-vidrio`
  - `refactor: extraer servicio de sync`
- Commits atómicos: un cambio lógico por commit.
- No pushear código que no compile o no pase lint.
