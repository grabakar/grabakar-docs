# GRABADO_PATENTE — Creación de Registro de Grabado

## Descripción

Formulario principal de la app. Captura los datos del grabado de patente sobre los vidrios de un vehículo. Genera un UUID v4 en el dispositivo, guarda en IndexedDB y encola para sincronización. Tras guardar, el registro se vuelve de solo lectura y el flujo continúa al multi-vidrio.

## Reglas de Negocio

- Cada registro de grabado genera un `uuid` v4 en el dispositivo al momento de guardar (no al abrir el formulario).
- El campo `patente` se normaliza a **mayúsculas sin espacios ni guiones** antes de guardar. Ejemplo: `"bb-df 12"` → `"BBDF12"`.
- **Detección de duplicados**: Al confirmar la patente, buscar en IndexedDB registros de los últimos 30 días con la misma patente. Si existe, mostrar alerta:
  > _"La patente BBDF12 ya fue grabada el 15/02/2026. ¿Deseas continuar de todos modos?"_
  Si el usuario acepta, el registro se guarda con `es_duplicado: true`.
- Tras guardar: `patente` y `vin_chasis` pasan a **solo lectura**. El botón cambia de "Guardar" a "Imprimir".
- **Prevención de reimpresión fraudulenta**: Una vez guardado, no se puede modificar la patente y reimprimir. Cada impresión incrementa `cantidad_impresiones` en el registro, campo auditado.
- `usuario_responsable` se pre-llena con el usuario autenticado y no es editable.

## Flujo de Usuario

1. Usuario tap "Nuevo Grabado" en Home.
2. App muestra formulario con campos pre-llenados (usuario, fecha).
3. Usuario ingresa patente → ingresa confirmación de patente.
4. Si patente no coincide: error inline _"Las patentes no coinciden."_
5. Si patente coincide: buscar duplicados en IndexedDB (últimos 30 días).
6. Si duplicado encontrado: mostrar diálogo de confirmación con fecha del registro anterior.
7. Usuario completa campos restantes (VIN, orden de trabajo, ley, tipo movimiento, tipo vehículo, formato impresión).
8. Tap "Guardar":
   - Generar UUID v4.
   - Guardar en IndexedDB con `estado_sync: "pendiente"`.
   - Campos `patente` y `vin_chasis` pasan a solo lectura.
   - Botón cambia a "Continuar a Grabado de Vidrios".
9. Tap "Continuar a Grabado de Vidrios" → Navegar al flujo multi-vidrio.

## Frontend

### Modelo IndexedDB (Dexie.js)

```typescript
interface Grabado {
  uuid: string;                  // UUID v4 generado en dispositivo
  patente: string;               // Normalizada: mayúsculas, sin espacios/guiones
  patente_confirmacion: string;
  vin_chasis: string | null;
  orden_trabajo: string | null;
  usuario_responsable_id: number;
  usuario_responsable_nombre: string;
  ley_caso_id: number;
  ley_caso_nombre: string;
  tipo_movimiento: 'venta' | 'demo' | 'capacitacion';
  tipo_vehiculo: 'auto' | 'moto';
  formato_impresion: 'horizontal' | 'vertical';
  es_duplicado: boolean;
  fecha_creacion_local: string;  // ISO 8601
  estado_sync: 'pendiente' | 'sincronizado' | 'error';
  fecha_sincronizacion: string | null;
  tenant_id: number;
}
```

### Schema Dexie

```typescript
db.version(1).stores({
  grabados: 'uuid, patente, fecha_creacion_local, estado_sync, tenant_id',
  impresiones_vidrio: '++id, grabado_uuid, nombre_vidrio',
  configuracion: 'clave'
});
```

### Componente Formulario

- Usar `react-hook-form` con validación en `onBlur` y `onSubmit`.
- Campo `patente`: input con `maxLength={8}`, `autoCapitalize`, transform a mayúsculas en `onChange`.
- Campo `patente_confirmacion`: validar match contra `patente` en `onBlur`.
- Selectores (`ley_caso`, `tipo_movimiento`, `tipo_vehiculo`, `formato_impresion`): cargar opciones desde IndexedDB (config cacheada en login).
- `usuario_responsable`: campo de solo lectura, pre-llenado desde `useAuth()`.
- Botón "Guardar" deshabilitado hasta que validación pase.

### Detección de Duplicados

```typescript
async function detectarDuplicado(patente: string): Promise<Grabado | null> {
  const hace30Dias = new Date();
  hace30Dias.setDate(hace30Dias.getDate() - 30);
  return db.grabados
    .where('patente').equals(patente)
    .and(r => new Date(r.fecha_creacion_local) >= hace30Dias)
    .first();
}
```

### Post-Guardado

- Deshabilitar campos `patente`, `vin_chasis` con `readOnly`.
- Cambiar botón a "Continuar a Grabado de Vidrios" (color primario del tenant).
- Mostrar toast de confirmación: _"Grabado guardado correctamente."_

## Backend

### Endpoint de Sincronización

El registro se sincroniza vía `POST /api/v1/sync/upload/` (ver `SINCRONIZACION_OFFLINE.md`). El backend valida e inserta.

### Modelo Django

```python
class Grabado(models.Model):
    uuid = models.UUIDField(primary_key=True)
    patente = models.CharField(max_length=8, db_index=True)
    vin_chasis = models.CharField(max_length=17, blank=True, null=True)
    orden_trabajo = models.CharField(max_length=50, blank=True, null=True)
    usuario_responsable = models.ForeignKey('Usuario', on_delete=models.PROTECT)
    ley_caso = models.ForeignKey('Ley', on_delete=models.PROTECT)
    tipo_movimiento = models.CharField(max_length=15, choices=[('venta','Venta'), ('demo','Demo'), ('capacitacion','Capacitación')])
    tipo_vehiculo = models.CharField(max_length=10, choices=[('auto','Auto'), ('moto','Moto')])
    formato_impresion = models.CharField(max_length=12, choices=[('horizontal','Horizontal'), ('vertical','Vertical')])
    es_duplicado = models.BooleanField(default=False)
    fecha_creacion_local = models.DateTimeField()
    fecha_sincronizacion = models.DateTimeField(auto_now_add=True)
    tenant = models.ForeignKey('Tenant', on_delete=models.CASCADE)

    class Meta:
        indexes = [models.Index(fields=['tenant', 'patente', 'fecha_creacion_local'])]
```

### Validación Servidor

- Verificar que `uuid` no exista con `fecha_creacion_local` más reciente (conflicto → dispositivo gana si es más nuevo).
- Verificar que `ley_caso` pertenezca al tenant del usuario.
- Verificar que `usuario_responsable` pertenezca al tenant.

## Validaciones

| Campo | Regla | Mensaje |
|-------|-------|---------|
| patente | Requerido, min 6 chars, max 8, alfanumérico | _"La patente debe tener entre 6 y 8 caracteres alfanuméricos."_ |
| patente_confirmacion | Debe coincidir con patente | _"Las patentes no coinciden."_ |
| vin_chasis | Opcional, si presente: exactamente 17 chars alfanuméricos | _"El VIN/Chasis debe tener 17 caracteres alfanuméricos."_ |
| orden_trabajo | Opcional, max 50 chars | _"La orden de trabajo no puede exceder 50 caracteres."_ |
| ley_caso | Requerido, debe existir en config cacheada | _"Selecciona una ley vigente."_ |
| tipo_movimiento | Requerido, valor de enum | _"Selecciona el tipo de movimiento."_ |
| tipo_vehiculo | Requerido, valor de enum | _"Selecciona el tipo de vehículo."_ |
| formato_impresion | Requerido, valor de enum | _"Selecciona el formato de impresión."_ |

## Criterio de Aceptación

1. Guardar un grabado genera UUID v4, persiste en IndexedDB con `estado_sync: "pendiente"`.
2. Patente se normaliza a mayúsculas sin espacios/guiones al guardar.
3. Si patente existe en IndexedDB (últimos 30 días), se muestra alerta de duplicado con fecha.
4. Usuario puede continuar con duplicado; registro se marca `es_duplicado: true`.
5. Tras guardar, campos `patente` y `vin_chasis` quedan de solo lectura.
6. Botón cambia de "Guardar" a "Continuar a Grabado de Vidrios" tras guardar exitoso.
7. `usuario_responsable` se pre-llena automáticamente y no es editable.
8. Formulario funciona completamente offline (sin conexión a internet).
9. Navegación a flujo multi-vidrio pasa el `uuid` del grabado creado.

## Casos Edge

- **Patente con caracteres especiales**: El regex `/^[A-Z0-9]{6,8}$/` se aplica post-normalización. Si no pasa, mostrar error antes de intentar guardar.
- **UUID collision**: Probabilidad despreciable (2^122 espacio). No se implementa detección, pero el backend rechaza INSERT duplicado con 409.
- **IndexedDB llena**: Capturar `QuotaExceededError` al guardar. Mostrar: _"Almacenamiento lleno. Sincroniza los registros pendientes o libera espacio."_
- **Usuario cambiado en otro dispositivo (desactivado)**: El grabado se guarda localmente. Al sincronizar, el backend rechaza con 403. El registro queda en `estado_sync: "error"` con detalle.
- **App se cierra durante guardado**: `Dexie.transaction()` garantiza atomicidad. Si la transacción no completó, no hay registro parcial.
- **Confirmación de patente con paste**: Deshabilitar paste en campo `patente_confirmacion` para forzar doble digitación.
