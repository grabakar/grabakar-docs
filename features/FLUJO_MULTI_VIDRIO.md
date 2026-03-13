# FLUJO_MULTI_VIDRIO — Flujo Iterativo de Grabado por Vidrio (Poka-Yoke)

## Descripción

Tras crear un registro de grabado, la app itera por cada vidrio del vehículo mostrando la patente en formato grande para validación visual. El operador imprime la patente en cada vidrio y avanza. El diseño Poka-Yoke (a prueba de errores) garantiza que el operador vea y confirme la patente correcta en cada paso.

## Reglas de Negocio

- La lista de vidrios proviene de la config del tenant según `tipo_vehiculo`:
  - `auto`: 6 vidrios — Parabrisas, Luneta, Delantera Izquierda, Delantera Derecha, Trasera Izquierda, Trasera Derecha.
  - `moto`: 1 vidrio — Parabrisas.
- Cada vidrio genera un registro `ImpresionVidrio` en IndexedDB asociado al `Grabado` padre.
- El botón "Imprimir" incrementa `cantidad_impresiones` en el registro `ImpresionVidrio`. Fase 1: no hay integración Bluetooth, solo se registra el evento.
- El operador puede reimprimir un vidrio (múltiples taps a "Imprimir"). Cada tap incrementa el contador.
- El operador puede saltar un vidrio sin imprimir. El registro se crea con `cantidad_impresiones: 0` y `omitido: true`.
- En el último vidrio, el botón "Siguiente Vidrio" cambia a "Finalizar".
- Al finalizar: marcar el `Grabado` como `completado: true` en IndexedDB, navegar a Home.

## Flujo de Usuario

1. Llega desde `GRABADO_PATENTE` con el `uuid` del grabado.
2. App carga la lista de vidrios según `tipo_vehiculo` desde config cacheada.
3. **Pantalla de vidrio N**:
   - Header: _"Vidrio 3 de 6"_ (indicador de progreso).
   - Centro: Patente en texto grande (min 48px, bold, monospace). Ej: **BBDF12**.
   - Nombre del vidrio: _"Delantera Izquierda"_.
   - Botón "Imprimir" (acción primaria, grande).
   - Botón "Omitir Vidrio" (acción secundaria, menor prominencia).
   - Botón "Siguiente Vidrio" / "Finalizar" (habilitado siempre).
4. Tap "Imprimir":
   - Incrementar `cantidad_impresiones` del `ImpresionVidrio` en IndexedDB.
   - Feedback visual: toast _"Impresión registrada ({n})"_ donde `{n}` es el total de impresiones.
   - Botón no navega automáticamente; el operador decide cuándo avanzar.
5. Tap "Omitir Vidrio":
   - Mostrar confirmación: _"¿Omitir el grabado de {nombre_vidrio}? Se registrará como no impreso."_
   - Si confirma: marcar `omitido: true`, avanzar al siguiente vidrio.
6. Tap "Siguiente Vidrio": Avanzar al vidrio N+1.
7. Tap "Finalizar" (último vidrio):
   - Marcar `Grabado.completado = true` en IndexedDB.
   - Toast: _"Grabado finalizado correctamente."_
   - Navegar a Home.

## Frontend

### Modelo IndexedDB

```typescript
interface ImpresionVidrio {
  id?: number;                   // Auto-increment (Dexie)
  grabado_uuid: string;          // FK al Grabado padre
  nombre_vidrio: string;         // "Parabrisas", "Luneta", etc.
  orden: number;                 // Posición en secuencia (1-based)
  cantidad_impresiones: number;  // Contador de impresiones
  omitido: boolean;
  fecha_primera_impresion: string | null;  // ISO 8601
  fecha_ultima_impresion: string | null;   // ISO 8601
}
```

### Componente FlujoMultiVidrio

```typescript
interface FlujoMultiVidrioProps {
  grabadoUuid: string;
}

// Estado interno
interface EstadoFlujo {
  vidrioActual: number;    // índice 0-based
  vidrios: ImpresionVidrio[];
  patente: string;
  totalVidrios: number;
}
```

### Layout de Pantalla de Vidrio

```
┌─────────────────────────────┐
│  ← Atrás    Vidrio 3 de 6  │  ← Header con progreso
│                             │
│  ● ● ● ○ ○ ○               │  ← Progress dots
│                             │
│        B B D F 1 2          │  ← Patente grande (48px+, monospace, bold)
│                             │
│   Delantera Izquierda       │  ← Nombre del vidrio
│                             │
│  ┌───────────────────────┐  │
│  │     🖨 IMPRIMIR       │  │  ← Botón primario grande
│  └───────────────────────┘  │
│                             │
│     Omitir este vidrio      │  ← Link secundario
│                             │
│  ┌───────────────────────┐  │
│  │   SIGUIENTE VIDRIO →  │  │  ← Botón de navegación
│  └───────────────────────┘  │
└─────────────────────────────┘
```

### Creación de Registros

Al entrar al flujo, crear todos los `ImpresionVidrio` de una vez en una transacción Dexie:

```typescript
async function inicializarVidrios(grabadoUuid: string, tipoVehiculo: string): Promise<ImpresionVidrio[]> {
  const config = await db.configuracion.get('vidrios_por_tipo_vehiculo');
  const nombresVidrios: string[] = config.valor[tipoVehiculo];
  const registros = nombresVidrios.map((nombre, i) => ({
    grabado_uuid: grabadoUuid,
    nombre_vidrio: nombre,
    orden: i + 1,
    cantidad_impresiones: 0,
    omitido: false,
    fecha_primera_impresion: null,
    fecha_ultima_impresion: null,
  }));
  await db.transaction('rw', db.impresiones_vidrio, async () => {
    await db.impresiones_vidrio.bulkAdd(registros);
  });
  return registros;
}
```

### Función de Impresión

```typescript
async function registrarImpresion(impresionId: number): Promise<number> {
  const ahora = new Date().toISOString();
  await db.impresiones_vidrio.update(impresionId, {
    cantidad_impresiones: db.impresiones_vidrio.get(impresionId)
      .then(r => (r?.cantidad_impresiones ?? 0) + 1),
    fecha_primera_impresion: /* set only if null */,
    fecha_ultima_impresion: ahora,
    omitido: false,
  });
  const updated = await db.impresiones_vidrio.get(impresionId);
  return updated!.cantidad_impresiones;
}
```

### Navegación

- Botón "Atrás" en el primer vidrio: diálogo de confirmación _"¿Abandonar el flujo de grabado? Los vidrios ya registrados se conservarán."_
- Botón hardware "Back" (Android): mismo comportamiento que botón "Atrás".
- No se permite navegar a un vidrio anterior una vez avanzado (flujo unidireccional).

## Backend

### Modelo Django

```python
class ImpresionVidrio(models.Model):
    grabado = models.ForeignKey('Grabado', on_delete=models.CASCADE, related_name='impresiones')
    nombre_vidrio = models.CharField(max_length=50)
    orden = models.PositiveSmallIntegerField()
    cantidad_impresiones = models.PositiveIntegerField(default=0)
    omitido = models.BooleanField(default=False)
    fecha_primera_impresion = models.DateTimeField(null=True, blank=True)
    fecha_ultima_impresion = models.DateTimeField(null=True, blank=True)

    class Meta:
        unique_together = ['grabado', 'orden']
        ordering = ['orden']
```

### Sincronización

Los registros `ImpresionVidrio` se sincronizan junto con su `Grabado` padre en el payload de sync (ver `SINCRONIZACION_OFFLINE.md`). Ejemplo de payload:

```json
{
  "uuid": "a1b2c3d4-...",
  "patente": "BBDF12",
  "...campos grabado...",
  "impresiones": [
    { "nombre_vidrio": "Parabrisas", "orden": 1, "cantidad_impresiones": 1, "omitido": false, "fecha_primera_impresion": "2026-03-04T10:30:00Z", "fecha_ultima_impresion": "2026-03-04T10:30:00Z" },
    { "nombre_vidrio": "Luneta", "orden": 2, "cantidad_impresiones": 0, "omitido": true, "fecha_primera_impresion": null, "fecha_ultima_impresion": null }
  ]
}
```

## Validaciones

| Acción | Regla | Mensaje |
|--------|-------|---------|
| Omitir vidrio | Requiere confirmación | _"¿Omitir el grabado de {nombre_vidrio}? Se registrará como no impreso."_ |
| Abandonar flujo | Requiere confirmación | _"¿Abandonar el flujo de grabado? Los vidrios ya registrados se conservarán."_ |
| Finalizar | Al menos 1 vidrio con impresión > 0 | _"Debes imprimir al menos un vidrio para finalizar."_ |

## Criterio de Aceptación

1. La lista de vidrios corresponde al `tipo_vehiculo` del grabado, según config del tenant.
2. La patente se muestra en formato grande (min 48px), monospace, bold, en cada pantalla de vidrio.
3. Cada tap de "Imprimir" incrementa `cantidad_impresiones` en IndexedDB y muestra feedback.
4. El operador puede omitir un vidrio con confirmación; se registra como `omitido: true`.
5. Indicador de progreso muestra posición actual vs total (_"Vidrio 3 de 6"_).
6. En el último vidrio, botón muestra "Finalizar" en lugar de "Siguiente Vidrio".
7. Al finalizar, `Grabado.completado` es `true` y se navega a Home.
8. Todos los registros `ImpresionVidrio` persisten en IndexedDB para sync posterior.
9. El flujo funciona completamente offline.

## Casos Edge

- **App se cierra en medio del flujo**: Al reabrir, si hay un `Grabado` con `completado: false`, ofrecer retomar: _"Tienes un grabado en progreso para la patente {patente}. ¿Deseas continuar?"_ Cargar vidrios existentes y posicionar en el primer vidrio sin impresión.
- **Config de vidrios cambia entre login y grabado**: Usar la config que estaba vigente al momento de crear el grabado. No recalcular vidrios mid-flow.
- **Tipo vehículo con 0 vidrios configurados**: No debería ocurrir (validación de backend). Si ocurre, mostrar error: _"No hay vidrios configurados para este tipo de vehículo. Contacta al administrador."_
- **Múltiples impresiones rápidas (tap doble)**: Debounce de 500ms en botón "Imprimir" para evitar doble incremento accidental.
- **Pantalla rotación**: Layout debe adaptarse. La patente grande debe permanecer legible en landscape y portrait.
