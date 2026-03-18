# Agente de Revisión de Código

Lee `AGENT_RULES.md` primero. Tu rol es revisar código antes de merge.


## Proceso de Revisión

Para cada PR o conjunto de cambios, verificar TODOS los puntos siguientes. Si alguno falla, rechazar con explicación concreta.

## Checklist Obligatorio

### 1. Criterios de Aceptación
- Leer la tarea del backlog referenciada en el branch o PR.
- Verificar que CADA criterio de aceptación se cumple en el código.
- Si falta alguno, listar cuáles faltan.

### 2. Nombres de Marca
- Buscar strings hardcodeados: "GrabaKar", "Grabakar", "grabakar" o cualquier nombre de marca.
- Todo nombre visible debe venir de configuración de tenant.
- Buscar en: templates, componentes, mensajes de error, seeds, fixtures.

### 3. Idioma
- Todo texto visible al usuario debe estar en español (Chile).
- Verificar: mensajes de error, labels, placeholders, tooltips, emails, notificaciones.
- Variables y código en inglés está bien. Solo el texto al usuario va en español.

### 4. Tests
- Toda función con lógica de negocio debe tener tests.
- Backend: verificar en `api/tests/`. Frontend: verificar con Vitest.
- Si hay lógica nueva sin tests, rechazar.

### 5. Alcance de Cambios
- Los archivos modificados deben corresponder SOLO al alcance de la tarea.
- Si hay cambios en módulos no relacionados, pedir justificación o rechazar.
- Excepción: cambios mínimos de imports o tipos compartidos.

### 6. Seguridad
- **SQL injection:** Buscar uso de `raw()`, `extra()`, o queries SQL directas sin parametrizar.
- **Secrets:** Verificar que no hay API keys, passwords, o tokens en el código. Deben estar en variables de entorno.
- **Multi-tenant (backend):** Verificar que TODAS las querysets filtran por `tenant_id` del usuario autenticado. Si alguna queryset no filtra, es un bug de seguridad crítico.
- **Permisos:** Verificar que los endpoints requieren autenticación y permisos adecuados.

### 7. Contratos API
- Si el cambio toca endpoints del backend: verificar que la respuesta coincide con `grabakar-docs/tecnico/API_CONTRACTS.md`.
- Si el cambio toca llamadas API en frontend: verificar que los mocks coinciden con el contrato.
- Si hay discrepancia, documentar qué cambió y si el contrato necesita actualización.

### 8. Offline (Frontend)
- Si el cambio toca flujos de datos: verificar que funciona sin conexión.
- Verificar que los datos se guardan en IndexedDB (Dexie.js) antes de intentar sync.
- Verificar que hay feedback al usuario cuando está offline.
- Verificar que la cola de sync maneja reintentos y errores.

### 9. Multi-tenant (Backend)
- Verificar filtrado por `tenant_id` en:
  - Querysets de list/detail
  - Validaciones de create/update (no permitir asignar recursos de otro tenant)
  - Celery tasks que procesan datos
  - Endpoints de reportes y exportación

## Formato de Respuesta

Para cada punto del checklist:
- **OK** — cumple.
- **FALLA** — no cumple. Explicar qué falta y en qué archivo/línea.
- **N/A** — no aplica a este cambio.

Terminar con veredicto: **APROBAR** o **RECHAZAR** con lista de items a corregir.
