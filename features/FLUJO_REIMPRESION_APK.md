# Flujo de reimpresión en APK y patente ya ingresada

**Backlog:** P3-05, P3-06
**Superficie:** `grabakar-frontend` (APK/PWA).
**Invariante:** [APK_SESION_OFFLINE_INVARIANTE.md](APK_SESION_OFFLINE_INVARIANTE.md) — no degradar sesión offline por estos flujos.

---

## Contexto

Existen dos necesidades distintas que antes estaban mezcladas en specs:

1. **P3-05 — Reimpresión en el flujo multi-vidrio:** simplificar UX; la **reimpresión múltiple** (varios intentos de impresión del mismo sticker lógico) no debe requerir UI compleja en **cada** paso de vidrio; la **reimpresión como acción explícita de revisión** se concentra en la **pantalla final** del flujo (según decisión de producto).
2. **P3-06 — Misma patente en otro momento:** si un vehículo/placa **ya fue grabada** anteriormente, el operador debe poder **volver a ingresar la misma patente** para un nuevo ciclo de impresión/reimpresión según reglas del negocio.

El documento legacy [FLUJO_MULTI_VIDRIO.md](FLUJO_MULTI_VIDRIO.md) describe el flujo actual con «Imprimir» por vidrio y reimpresión por múltiples taps; este documento **alinea** el comportamiento objetivo con P3-05/P3-06.

---

## Definiciones

| Término | Significado |
|---------|-------------|
| **Reimpresión intra-flujo** | Nuevo intento de impresión sobre el **mismo** grabado en curso (mismos vidrios en secuencia). |
| **Reimpresión post-flujo** | Ajustes o copias después de haber **finalizado** el grabado (desde pantalla final o desde home). |
| **Nuevo grabado con misma patente** | Nuevo registro (nuevo UUID) o política de duplicados; ver sección «Reglas de negocio». |

---

## P3-05 — Comportamiento objetivo (intra-flujo)

**Decisión de producto:** no es necesario exponer «reimprimir» en **cada** pantalla de vidrio de forma prominente; el flujo debe priorizar **«Imprimir y avanzar»** (o equivalente) para reducir fricción.

**Reimpresiones:** como cada impresión física es equivalente, los **reintentos** pueden concentrarse donde el operador **cierra** el trabajo:

- **Pantalla final** del flujo multi-vidrio: aquí se ofrece acción clara de **reimpresión** (p. ej. reimprimir resumen, reimprimir último vidrio, o lista corta de vidrios con contador), según diseño UX final.

**Registro de datos:** cada acción de impresión sigue incrementando o registrando eventos según el modelo `ImpresionVidrio` / sync (debe ser consistente con auditoría).

---

## P3-06 — Misma patente, nuevo ingreso

**Necesidad:** Si la patente **ya existe** en historial (local o servidor), el operador debe poder **ingresarla de nuevo** para:

- Reimprimir documentación física.
- Iniciar un **nuevo** grabado válido (p. ej. otro vehículo con misma patente en contexto distinto — sujeto a normativa interna del cliente).

### Reglas de negocio (a cerrar en implementación)

| Pregunta | Opciones típicas |
|----------|------------------|
| ¿Un segundo grabado con la misma patente es **siempre** un nuevo UUID? | Sí (recomendado para trazabilidad) vs. reabrir el anterior (solo en casos excepcionales). |
| ¿El APK bloquea duplicados **solo** en IndexedDB local? | Comportamiento actual de formulario; alinear con sync y servidor (evitar conflictos). |
| ¿Qué muestra la UI si detecta patente duplicada reciente? | Advertencia + permitir continuar vs. flujo administrativo. |

### Auditoría

- Toda reimpresión o nuevo grabado con patente ya vista debe dejar **trazabilidad**: usuario, timestamp, device_id, tenant, sucursal.

---

## Impacto en documentos existentes

- Actualizar [FLUJO_MULTI_VIDRIO.md](FLUJO_MULTI_VIDRIO.md) cuando el código refleje «final screen reprint» y «print+next» unificado.
- [dashboard-print.md](dashboard-print.md) — billing/charts siguen en panel; no en APK.

---

## Referencias

- [GRABADO_PATENTE.md](GRABADO_PATENTE.md)
- [SINCRONIZACION_OFFLINE.md](SINCRONIZACION_OFFLINE.md)
- [BACKLOG.md](../planificacion/BACKLOG.md) — P3-05, P3-06
