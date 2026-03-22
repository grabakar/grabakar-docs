# QA — GrabaKar

## Agente de QA (prompts)

* **[QA_AGENT_VERIFICATION_RBAC_OPENAPI.md](../prompts/agents/QA_AGENT_VERIFICATION_RBAC_OPENAPI.md)** — Prompt para verificar implementación **OpenAPI (drf-spectacular)** y **RBAC del panel** (`panel_persona`, alcance tenant/sucursal). Indica cómo **registrar planes y resultados** en esta carpeta.
* Flujo emulador / APK: [QA_AGENT_WORKFLOW.md](QA_AGENT_WORKFLOW.md), [QA_AGENT_SETUP.md](QA_AGENT_SETUP.md).

## Registro de ejecuciones QA

* **[REGISTRO_QA_PLANTILLA.md](REGISTRO_QA_PLANTILLA.md)** — Plantilla para `REGISTRO_QA_<YYYY>_<tema>.md` (plan, casos límite, resultados, brechas).
* **`REGISTRO_QA_<YYYY>_RBAC_OPENAPI.md`** — Registro recomendado para la verificación post-implementación de **OpenAPI (drf-spectacular)** y **RBAC del panel** (ver `TC_RBAC_PANEL_ADMIN_QA.md` y el prompt del agente).
* **[REGISTRO_QA_2026_BATTERY_AUTOMATED.md](REGISTRO_QA_2026_BATTERY_AUTOMATED.md)** — Batería automatizada reciente (GitHub + pytest + builds + Vitest); enlaza evidencia en `qa_reports/`.

## Test Cases (Admin API)

* `US_ADMIN_PANEL.md`: Historias de usuario funcionales para el SuperAdmin.
* `TC_ADMIN_PANEL.md`: Casos de validación del panel admin (histórico: menciona solo `is_staff` / IsPlatformAdmin; el modelo actual usa **`Usuario.panel_persona`** — ver `tecnico/RBAC_PANEL_ADMIN.md` y registros `REGISTRO_QA_*.md`).
* `TC_RBAC_PANEL_ADMIN_QA.md`: Checklist orientativo para QA manual/automatizado sobre RBAC (opcional; puede crear/actualizar el agente de QA).

## Otros

* Plan maestro y matrices: [QA_MASTER_PLAN.md](QA_MASTER_PLAN.md), [QA_CHECKLIST_RELEASE.md](QA_CHECKLIST_RELEASE.md).
