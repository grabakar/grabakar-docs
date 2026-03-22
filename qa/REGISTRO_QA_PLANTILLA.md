# Registro de QA — plantilla

> Copiar este archivo a `REGISTRO_QA_<YYYY>_<tema>.md` y completar secciones.
> Instrucciones del agente: [prompts/agents/QA_AGENT_VERIFICATION_RBAC_OPENAPI.md](../prompts/agents/QA_AGENT_VERIFICATION_RBAC_OPENAPI.md).

---

## Metadatos

| Campo | Valor |
|-------|--------|
| **Tema / alcance** | |
| **Ambiente** | local / staging / producción |
| **Commit / tag / build** | |
| **Fecha** | |
| **Ejecutor** | (humano o agente) |
| **Duración estimada** | |

---

## 1. Alcance y exclusiones

- **Incluido:**
- **Excluido (fuera de alcance de esta ronda):**

---

## 2. Datos y usuarios de prueba

Documentar usuarios creados o fixtures usados. No commitear secretos reales; usar placeholders o variables de entorno.

| Usuario | `panel_persona` | `rol` | `is_staff` | Tenant | Sucursal | Uso previsto |
|---------|-----------------|-------|------------|--------|----------|--------------|
| | | | | | | |

---

## 3. Plan de pruebas

### 3.1 OpenAPI / Spectacular

| ID | Caso | Prioridad | Pasos resumidos | Resultado | Notas |
|----|------|-----------|-----------------|-----------|-------|
| O-01 | | P0 | | | |

### 3.2 API admin — RBAC

| ID | Caso | Persona | Endpoint / acción | Esperado HTTP | Resultado | Notas |
|----|------|---------|-------------------|---------------|-----------|-------|
| A-01 | | | | | | |

### 3.3 Panel web (`grabakar-admin`)

| ID | Caso | Persona | Resultado | Notas |
|----|------|---------|-----------|-------|
| U-01 | | | | |

### 3.4 Regresión (APK / auth común)

| ID | Caso | Resultado | Notas |
|----|------|-----------|-------|
| R-01 | | | |

---

## 4. Casos límite y negativos

| ID | Descripción | Entrada / truco | Esperado | Resultado |
|----|-------------|-----------------|----------|-----------|
| B-01 | | | | |

---

## 5. Ejecución automatizada (si aplica)

| Comando | Repo | Resultado (exit code / resumen) |
|---------|------|----------------------------------|
| `pytest` | grabakar-backend | |
| `npm run build` | grabakar-admin | |
| `python manage.py spectacular --validate --fail-on-warn` | grabakar-backend | |

---

## 6. Brechas y seguimiento

| Severidad | Descripción | Archivo / área afectada | Issue / acción |
|-----------|-------------|---------------------------|----------------|
| | | | |

---

## 7. Conclusión

- **Estado:** Aprobado / Aprobado con observaciones / No aprobado
- **Condiciones de salida:**
- **Próximos pasos recomendados:**
