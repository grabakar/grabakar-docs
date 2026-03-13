# Estrategia de Sincronización Offline-First

## Flujo de Sincronización

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐
│ Crear       │     │ Encolar      │     │ Detectar       │
│ Grabado     │────→│ para sync    │────→│ conectividad   │
│ (IndexedDB) │     │ estado_sync: │     │ online?        │
│             │     │ "pendiente"  │     │                │
└─────────────┘     └──────────────┘     └───────┬────────┘
                                                 │
                                    ┌────────────┴────────────┐
                                    │                         │
                                    ▼                         ▼
                              ┌───────────┐           ┌─────────────┐
                              │ OFFLINE   │           │ ONLINE      │
                              │ Reintentar│           │ Batch upload│
                              │ en 60s    │           │ POST /sync/ │
                              └───────────┘           └──────┬──────┘
                                                             │
                                                    ┌────────┴────────┐
                                                    │                 │
                                                    ▼                 ▼
                                              ┌──────────┐    ┌────────────┐
                                              │ SUCCESS  │    │ ERROR      │
                                              │ Marcar   │    │ Retry con  │
                                              │ synced   │    │ backoff    │
                                              └──────────┘    └────────────┘
```

## Detección de Conectividad

Dos mecanismos combinados:

1. **`navigator.onLine`**: Evento nativo del browser. Falsos positivos frecuentes (detecta red local pero no internet). Se usa como señal rápida.
2. **Health ping**: `GET /api/v1/health/` — endpoint liviano que retorna `200 OK`. Confirma conectividad real al backend.

```typescript
class ConnectivityService {
  private online = navigator.onLine;
  private pingInterval: number | null = null;
  private listeners: Set<(online: boolean) => void> = new Set();

  start() {
    window.addEventListener('online', () => this.check());
    window.addEventListener('offline', () => this.setOnline(false));
    this.schedulePing();
  }

  private schedulePing() {
    const interval = this.online ? 30_000 : 60_000;
    this.pingInterval = window.setInterval(() => this.check(), interval);
  }

  private async check() {
    try {
      const res = await fetch('/api/v1/health/', { method: 'GET', cache: 'no-store' });
      this.setOnline(res.ok);
    } catch {
      this.setOnline(false);
    }
  }

  private setOnline(value: boolean) {
    if (this.online !== value) {
      this.online = value;
      this.listeners.forEach(cb => cb(value));
      if (this.pingInterval) clearInterval(this.pingInterval);
      this.schedulePing();
    }
  }

  isOnline(): boolean { return this.online; }
  onChange(cb: (online: boolean) => void) { this.listeners.add(cb); }
}
```

Frecuencias de ping:
- **Online**: cada 30 segundos (detectar pérdida rápida).
- **Offline**: cada 60 segundos (no desperdiciar batería).

## Protocolo de Batch Upload

**Endpoint**: `POST /api/v1/sync/upload/`

**Restricciones**:
- Máximo 100 registros por batch.
- Si la cola tiene >100 registros, se envían en batches secuenciales.
- Cada batch es una transacción independiente.

**Payload**:

```json
{
  "device_id": "abc-123-def",
  "batch": [
    {
      "uuid": "550e8400-...",
      "patente": "BBCL99",
      "vin_chasis": "1HGCM82633A004352",
      "orden_trabajo": "OC-2026-001",
      "usuario_responsable_id": 42,
      "ley_caso_id": 1,
      "tipo_movimiento": "venta",
      "tipo_vehiculo": "auto",
      "formato_impresion": "horizontal",
      "fecha_creacion_local": "2026-03-04T10:30:00-03:00",
      "es_duplicado": false,
      "impresiones": [
        {
          "nombre_vidrio": "Parabrisas",
          "orden": 1,
          "cantidad_impresiones": 1,
          "fecha_primera_impresion": "2026-03-04T10:32:00-03:00",
          "fecha_ultima_impresion": "2026-03-04T10:32:00-03:00"
        }
      ]
    }
  ]
}
```

**Response 200**:

```json
{
  "recibidos": 50,
  "exitosos": 48,
  "fallidos": 2,
  "errores": [
    { "uuid": "...", "error": "ley_caso_id no pertenece al tenant" },
    { "uuid": "...", "error": "usuario_responsable_id inactivo" }
  ]
}
```

## Estrategia de Retry

Exponential backoff cuando un batch falla (error de red, 5xx del servidor):

| Intento | Espera | Acumulado |
|---|---|---|
| 1 | 5s | 5s |
| 2 | 15s | 20s |
| 3 | 45s | 1m 5s |
| 4 | 2min | 3m 5s |
| 5 | 5min | 8m 5s |
| 6+ | 30min (max) | — |

El retry se resetea a 5s cuando:
- El usuario fuerza un sync manual.
- Cambia de offline → online (evento `navigator.onLine`).

Los errores 4xx (validación, auth) no activan retry — indican un problema con los datos, no con la red.

## Resolución de Conflictos

**Regla**: el dispositivo gana.

Cuando el servidor recibe un registro cuyo `uuid` ya existe:
- Si `fecha_creacion_local` del incoming > `fecha_creacion_local` almacenado → **update**.
- Si `fecha_creacion_local` del incoming <= almacenado → **skip** (el servidor ya tiene la versión más reciente).
- `fecha_sincronizacion` se re-setea al timestamp del servidor en ambos casos.

No hay merge de campos individuales. El registro se trata como unidad atómica.

## Manejo de Timestamps

| Timestamp | Origen | Confiable | Uso |
|---|---|---|---|
| `fecha_creacion_local` | Reloj del dispositivo | No (puede estar mal) | Orden cronológico de negocio. Reportes. Purga. |
| `fecha_sincronizacion` | Reloj del servidor | Sí (autoritativo) | Auditoría. Verificación de sync. |

**El reloj del dispositivo puede estar equivocado.** Esto es aceptado. La app no intenta corregirlo. Las consecuencias:
- Un reporte diario puede incluir registros con `fecha_creacion_local` de un día diferente al real.
- La detección de duplicados (30 días) puede ser imprecisa si el reloj está muy desfasado.
- El backend registra `fecha_sincronizacion` como punto de referencia verificable.

## Registros Tardíos (Late-Arriving Records)

Cuando un dispositivo sincroniza registros de días anteriores (estuvo offline):

1. Backend inserta los registros normalmente.
2. Backend detecta que `fecha_creacion_local` pertenece a un día para el cual ya se generó un reporte.
3. Celery task regenera los reportes afectados.
4. Si hay un supervisor configurado para recibir reportes, se notifica que el reporte fue actualizado.

```python
def procesar_batch(device_id, batch, usuario):
    fechas_afectadas = set()
    for record in batch:
        grabado = crear_o_actualizar_grabado(record, usuario)
        fechas_afectadas.add(grabado.fecha_creacion_local.date())
    regenerar_reportes_si_necesario.delay(
        tenant_id=usuario.tenant_id,
        fechas=list(fechas_afectadas)
    )
```

## Purga de Datos Locales

**Cuándo**: al startup de la app + cada 24 horas (timer).

**Regla**: eliminar de IndexedDB donde `estado_sync == 'sincronizado'` AND `fecha_creacion_local` < ahora - 30 días.

**Registros no sincronizados nunca se purgan**, sin importar su antigüedad.

```typescript
async function purgar() {
  const limite = new Date();
  limite.setDate(limite.getDate() - 30);
  const ids = await db.grabados
    .where('estado_sync').equals('sincronizado')
    .and(r => r.fecha_creacion_local < limite.toISOString())
    .primaryKeys();
  await db.transaction('rw', [db.grabados, db.impresiones_vidrio], async () => {
    for (const uuid of ids) {
      await db.impresiones_vidrio.where('grabado_uuid').equals(uuid).delete();
    }
    await db.grabados.bulkDelete(ids);
  });
}
```

## Manejo de Errores en Batch

Errores individuales no detienen el batch completo.

- El backend procesa cada registro del batch independientemente.
- Registros exitosos se marcan `sincronizado` en la response.
- Registros fallidos se retornan con su `uuid` y mensaje de error.
- En el frontend, los registros fallidos permanecen en la cola con `estado_sync: 'error'` y el detalle del error almacenado.
- El usuario puede ver los registros con error en la pantalla de cola de sincronización.
- Errores típicos: `ley_caso_id` inválida, `usuario_responsable` inactivo, UUID ya existente con fecha más reciente.

## Máquina de Estados del SyncManager

```
                    ┌─────────┐
          ┌────────→│  IDLE   │←────────┐
          │         └────┬────┘         │
          │              │              │
          │         hay registros       │
          │         pendientes +        │
          │         online              │
          │              │              │
          │              ▼              │
          │         ┌─────────┐         │
          │         │ SYNCING │         │
          │         └────┬────┘         │
          │              │              │
          │    ┌─────────┴─────────┐    │
          │    │                   │    │
          │    ▼                   ▼    │
     ┌─────────┐            ┌──────────┐
     │ SUCCESS │            │  ERROR   │
     │ reset   │            │ backoff  │
     │ backoff │            │ wait     │
     └─────────┘            └──────────┘
```

**Transiciones**:
- `IDLE → SYNCING`: hay registros con `estado_sync: 'pendiente'` y `ConnectivityService.isOnline() == true`.
- `SYNCING → SUCCESS`: batch procesado (puede incluir errores individuales, pero la request HTTP fue exitosa).
- `SYNCING → ERROR`: request HTTP falló (network error, 5xx, timeout).
- `SUCCESS → IDLE`: inmediato. Si quedan más registros pendientes, transiciona de vuelta a SYNCING.
- `ERROR → IDLE`: tras esperar el intervalo de backoff.

**Triggers de sync**:
- Cambio de offline → online.
- Nuevo registro guardado en IndexedDB.
- Timer periódico (cada 30s cuando online).
- Botón manual "Sincronizar ahora" en UI.
