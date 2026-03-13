# Reglas Generales para Agentes

Estas reglas aplican a TODOS los agentes que trabajen en cualquier repo de GrabaKar.

> **Implementation Plan:** Read [AGENT_RULES_IMPLEMENTATION.md](implementation_plans/AGENT_RULES_IMPLEMENTATION.md) for exact PDCA procedures, git workflow commands, verification scripts, parallel work protocol, and error handling standards.

## Estilo de Código

- Minimizar líneas de código. Sin bloat.
- No agregar logs salvo que estés depurando activamente. Eliminarlos antes de commit.
- Preferir pocos archivos con más funcionalidad sobre muchos archivos con poco contenido.
- Sin comentarios obvios ni narrativos. Comentarios SOLO para intención no evidente, trade-offs o restricciones.
- Organizar en carpetas: `utils/`, `scripts/`, `prompts/`.

## Metodología de Trabajo: PDCA

Seguir este orden estricto:

1. **CHECK** — Leer el estado actual: spec de la feature, contrato API, código existente relacionado.
2. **ACT** — Identificar qué falta, qué está mal, qué bloquea.
3. **PLAN** — Definir los pasos concretos antes de tocar código.
4. **DO** — Implementar. Sin desviaciones del plan.

## Antes de Empezar

- Leer la spec de la feature en `grabakar-docs/backlog/` o `grabakar-docs/fases/`.
- Leer el contrato API en `grabakar-docs/tecnico/API_CONTRACTS.md`.
- Verificar la tarea asignada en el backlog y sus criterios de aceptación.

## Al Terminar

- Ejecutar linter (`ruff` en backend, `eslint` en frontend).
- Ejecutar tests (`pytest` en backend, `vitest` en frontend).
- Verificar que TODOS los criterios de aceptación de la tarea se cumplen.
- Revisar que no se modificaron archivos fuera del alcance de la tarea.

## Convenciones de Git

**Branches:** `feature/<fase>-<nombre-corto>`
- Ejemplo: `feature/f1-login-auth`, `feature/f2-grabado-crud`

**Commits:** Conventional Commits en español.
- `feat(auth): implementar login con JWT`
- `fix(sync): corregir conflicto de merge en cola offline`
- `test(grabado): agregar tests para validación de patente`
- `refactor(api): simplificar serializer de vidrios`

## Herramientas

- Usar `@mantic` para navegación y búsqueda de código en el proyecto.
- Usar `@exa` para buscar documentación externa, APIs de terceros, o referencias técnicas.

## Reglas Absolutas

- **NUNCA hardcodear "GrabaKar"** ni ningún nombre de marca. Siempre usar configuración de tenant (`tenant.nombre`, `tenant.logo`, etc.).
- **Todo texto visible al usuario en español (Chile).** Sin excepciones.
- **Escribir tests para toda función con lógica de negocio.** No es opcional.
- **No modificar archivos fuera del alcance de tu tarea.** Si necesitas un cambio en otro módulo, documentarlo como dependencia en la tarea.
- **No crear archivos de documentación** salvo que se solicite explícitamente.
- La documentación debe ser funcional (cómo hacer X), no narrativa.

## Ubicación de Documentación

Toda la documentación del proyecto vive en `grabakar-docs/`:
- Specs de features: `fases/`, `backlog/`
- Contratos API: `tecnico/API_CONTRACTS.md`
- Arquitectura: `tecnico/ARQUITECTURA.md`
- Modelos de datos: `tecnico/MODELO_DATOS.md`
