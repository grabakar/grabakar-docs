# SINCRONIZACION_OFFLINE — Estrategia de Sincronización Offline-First

## Descripción

Toda la operación de la app es offline-first. Los datos se persisten primero en IndexedDB (Dexie.js) y se sincronizan con el backend cuando hay conexión. La estrategia incluye detección de conectividad, reintentos con backoff exponencial, resolución de conflictos (dispositivo gana), y purga automática de registros antiguos.

## Reglas de Negocio

- **Principio fundamental**: La app funciona sin internet. IndexedDB es la fuente de verdad local. El backend es el destino de sincronización.
- Los registros se crean con `estado_sync: "pendiente"` y transicionan a `"sincronizado"` o `"error"`.
- La sincronización es unidireccional: dispositivo → servidor. No hay descarga de registros al dispositivo.
- **Resolución de conflictos**: Si el servidor ya tiene un registro con el mismo UUID, compara `fecha_creacion_local`. Si el registro entrante es más reciente, se actualiza. Si es más antiguo o igual, se descarta (respuesta 200 con `"accion": "ignorado"`).
- Registros sincronizados y con más de 30 días se purgan de IndexedDB. Registros **no sincronizados nunca se purgan**, sin importar antigüedad.

## Flujo de Sincronización

```
┌──────────────────┐
│ Guardar registro  │
│ en IndexedDB      │
│ estado: pendiente │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐    No     ┌────────────────────┐
│ ¿Hay conexión?   │─────────→ │ Esperar detección  │
│                  │           │ de conectividad    │
└────────┬─────────┘           └────────┬───────────┘
         │ Sí                           │ Online detectado
         ▼                              │
┌──────────────────┐ ◄─────────────────┘
│ SyncManager:     │
│ Obtener batch    │
│ (max 100 records)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ POST /api/v1/    │
│ sync/upload/     │
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
 Success    Error
    │         │
    ▼         ▼
 Marcar    ¿Reintentable?
 sincronizado  │
    │     ┌───┴───┐
    │     Sí      No
    │     │       │
    │     ▼       ▼
    │  Backoff   Marcar
    │  y retry   error
    │     │       │
    ▼     ▼       ▼
┌──────────────────┐
│ ¿Más pendientes? │──→ Sí: Siguiente batch
│                  │──→ No: Idle
└──────────────────┘
```

## Detección de Conectividad

Dos mecanismos combinados:

1. **`navigator.onLine`**: Escuchar eventos `online`/`offline` en `window`. Reacción inmediata pero no 100% confiable (puede reportar `true` en redes cautivas).
2. **Ping periódico**: `GET /api/v1/health/` cada 30 segundos cuando `navigator.onLine === true`. Si falla 3 veces consecutivas, considerar offline.

```typescript
interface ConectividadState {
  isOnline: boolean;         // Estado combinado (navigator + ping)
  ultimoPingExitoso: Date | null;
  fallosConsecutivos: number;
}
```

La sincronización solo se dispara cuando `isOnline === true` (ambos indicadores coinciden).

## Frontend

### SyncManager

Clase singleton que gestiona todo el ciclo de sincronización.

```typescript
class SyncManager {
  private static instance: SyncManager;
  private sincronizando = false;
  private intervaloBackoff = 5000; // ms inicial
  private readonly MAX_BACKOFF = 1800000; // 30 min
  private readonly BATCH_SIZE = 100;
  private listeners: Set<(estado: SyncStatus) => void> = new Set();

  static getInstance(): SyncManager { /* singleton */ }

  async iniciarSync(): Promise<void> {
    if (this.sincronizando) return;
    this.sincronizando = true;
    this.notificar();
    try {
      let pendientes = await this.obtenerPendientes();
      while (pendientes.length > 0) {
        const batch = pendientes.slice(0, this.BATCH_SIZE);
        await this.enviarBatch(batch);
        pendientes = await this.obtenerPendientes();
      }
      this.intervaloBackoff = 5000;
    } catch (e) {
      this.programarReintento();
    } finally {
      this.sincronizando = false;
      this.notificar();
    }
  }

  private async obtenerPendientes(): Promise<Grabado[]> {
    return db.grabados
      .where('estado_sync').equals('pendiente')
      .toArray();
  }

  private async enviarBatch(batch: Grabado[]): Promise<void> {
    const payload = await Promise.all(
      batch.map(async g => ({
        ...g,
        impresiones: await db.impresiones_vidrio
          .where('grabado_uuid').equals(g.uuid)
          .toArray()
      }))
    );
    const response = await fetch('/api/v1/sync/upload/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getAccessToken()}`
      },
      body: JSON.stringify({ registros: payload })
    });
    if (!response.ok) throw new SyncError(response.status);
    const resultado = await response.json();
    await this.procesarResultado(resultado);
  }

  private async procesarResultado(resultado: SyncResultado): Promise<void> {
    for (const item of resultado.resultados) {
      if (item.status === 'ok' || item.status === 'ignorado') {
        await db.grabados.update(item.uuid, {
          estado_sync: 'sincronizado',
          fecha_sincronizacion: new Date().toISOString()
        });
      } else if (item.status === 'error') {
        await db.grabados.update(item.uuid, {
          estado_sync: 'error',
          error_sync: item.mensaje
        });
      }
    }
  }

  private programarReintento(): void {
    setTimeout(() => this.iniciarSync(), this.intervaloBackoff);
    this.intervaloBackoff = Math.min(this.intervaloBackoff * 3, this.MAX_BACKOFF);
  }
  // Backoff: 5s → 15s → 45s → 2m15s → 6m45s → 20m15s → 30m (max)

  subscribe(fn: (estado: SyncStatus) => void): () => void { /* ... */ }
  private notificar(): void { /* ... */ }
}
```

### Hook React

```typescript
interface SyncStatus {
  pendientes: number;
  sincronizando: boolean;
  ultimaSync: Date | null;
  error: string | null;
}

function useSyncStatus(): SyncStatus {
  const [status, setStatus] = useState<SyncStatus>({
    pendientes: 0,
    sincronizando: false,
    ultimaSync: null,
    error: null,
  });

  useEffect(() => {
    const syncManager = SyncManager.getInstance();
    const unsubscribe = syncManager.subscribe(setStatus);
    return unsubscribe;
  }, []);

  return status;
}
```

### Indicador Visual en UI

- Badge en header con número de registros pendientes: `(3)` en color de advertencia.
- Icono de sync giratorio cuando `sincronizando === true`.
- Si `error !== null`, icono rojo con tooltip del error.
- Tap en el indicador muestra detalle: _"3 registros pendientes | Última sincronización: hace 15 min"_.

### Triggers de Sincronización

1. **Evento `online`**: Iniciar sync inmediatamente.
2. **Post-guardado**: Intentar sync tras guardar un nuevo grabado (si hay conexión).
3. **App resume (visibility change)**: Al volver a la app, verificar conexión e intentar sync.
4. **Periódico**: Cada 5 minutos si hay conexión y registros pendientes.

### Purga de Registros Antiguos

```typescript
async function purgarRegistrosAntiguos(): Promise<number> {
  const hace30Dias = new Date();
  hace30Dias.setDate(hace30Dias.getDate() - 30);
  const aPurgar = await db.grabados
    .where('estado_sync').equals('sincronizado')
    .and(r => new Date(r.fecha_creacion_local) < hace30Dias)
    .toArray();
  const uuids = aPurgar.map(g => g.uuid);
  await db.transaction('rw', [db.grabados, db.impresiones_vidrio], async () => {
    await db.impresiones_vidrio.where('grabado_uuid').anyOf(uuids).delete();
    await db.grabados.where('uuid').anyOf(uuids).delete();
  });
  return uuids.length;
}
```

- Se ejecuta al inicio de la app y cada 24 horas (via `setInterval`).
- Solo purga registros con `estado_sync === 'sincronizado'` y `fecha_creacion_local` > 30 días.

## Backend

### Endpoint de Sync

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/api/v1/sync/upload/` | Recibir batch de grabados |
| GET | `/api/v1/health/` | Health check para detección de conectividad |

### Request `POST /api/v1/sync/upload/`

```json
{
  "registros": [
    {
      "uuid": "a1b2c3d4-e5f6-...",
      "patente": "BBDF12",
      "vin_chasis": null,
      "orden_trabajo": "OT-001",
      "usuario_responsable_id": 42,
      "ley_caso_id": 1,
      "tipo_movimiento": "venta",
      "tipo_vehiculo": "auto",
      "formato_impresion": "horizontal",
      "es_duplicado": false,
      "fecha_creacion_local": "2026-03-04T10:30:00-03:00",
      "tenant_id": 1,
      "impresiones": [
        { "nombre_vidrio": "Parabrisas", "orden": 1, "cantidad_impresiones": 1, "omitido": false, "fecha_primera_impresion": "2026-03-04T10:31:00-03:00", "fecha_ultima_impresion": "2026-03-04T10:31:00-03:00" },
        { "nombre_vidrio": "Luneta", "orden": 2, "cantidad_impresiones": 0, "omitido": true, "fecha_primera_impresion": null, "fecha_ultima_impresion": null }
      ]
    }
  ]
}
```

### Response `POST /api/v1/sync/upload/`

```json
{
  "resultados": [
    { "uuid": "a1b2c3d4-e5f6-...", "status": "ok" },
    { "uuid": "x9y8z7w6-...", "status": "error", "mensaje": "ley_caso_id no pertenece al tenant" },
    { "uuid": "m1n2o3p4-...", "status": "ignorado", "mensaje": "registro más reciente ya existe en servidor" }
  ]
}
```

### Lógica de Servidor

```python
@api_view(['POST'])
@permission_classes([IsAuthenticated])
def sync_upload(request):
    resultados = []
    for registro in request.data['registros']:
        try:
            existente = Grabado.objects.filter(uuid=registro['uuid']).first()
            if existente:
                if registro['fecha_creacion_local'] > existente.fecha_creacion_local:
                    # Dispositivo gana: actualizar
                    actualizar_grabado(existente, registro)
                    resultados.append({'uuid': str(registro['uuid']), 'status': 'ok'})
                else:
                    resultados.append({'uuid': str(registro['uuid']), 'status': 'ignorado', 'mensaje': 'registro más reciente ya existe en servidor'})
            else:
                crear_grabado(registro, request.user.tenant)
                resultados.append({'uuid': str(registro['uuid']), 'status': 'ok'})
        except ValidationError as e:
            resultados.append({'uuid': str(registro['uuid']), 'status': 'error', 'mensaje': str(e)})
    return Response({'resultados': resultados})
```

### Health Endpoint

```python
@api_view(['GET'])
def health(request):
    return Response({'status': 'ok', 'timestamp': now().isoformat()})
```

## Validaciones

| Escenario | Regla | Mensaje (en response) |
|-----------|-------|----------------------|
| Token expirado | 401 Unauthorized | Trigger refresh token flow en cliente |
| Tenant mismatch | `tenant_id` del registro ≠ tenant del usuario | _"tenant_id no corresponde al usuario autenticado"_ |
| Ley inválida | `ley_caso_id` no existe o no pertenece al tenant | _"ley_caso_id no pertenece al tenant"_ |
| UUID duplicado más nuevo | Registro en server es más reciente | `status: "ignorado"` |
| Batch vacío | `registros` es array vacío | 400: _"El batch no puede estar vacío."_ |
| Batch excede límite | > 100 registros | 400: _"El batch no puede exceder 100 registros."_ |

## Criterio de Aceptación

1. Un registro guardado sin conexión aparece en Home con indicador de "pendiente".
2. Al recuperar conexión, los registros pendientes se sincronizan automáticamente en batches de max 100.
3. Tras sync exitoso, `estado_sync` cambia a `"sincronizado"` y `fecha_sincronizacion` se registra.
4. Si un registro falla en sync, se marca como `"error"` y los demás del batch continúan.
5. El backoff exponencial incrementa: 5s → 15s → 45s → ~2min → ~5min → ... → max 30min.
6. `useSyncStatus()` refleja en tiempo real: pendientes, sincronizando, última sync, error.
7. Registros sincronizados con más de 30 días se purgan automáticamente de IndexedDB.
8. Registros no sincronizados **nunca** se purgan, sin importar antigüedad.
9. La purga se ejecuta al inicio de la app y cada 24 horas.
10. El health endpoint responde en < 500ms.

## Casos Edge

- **Conexión inestable (online/offline rápido)**: Debounce de 3 segundos en evento `online` antes de iniciar sync, para evitar disparar sync en conexiones fugaces.
- **Sync interrumpida (app se cierra mid-request)**: Los registros quedan como `"pendiente"`. Al reabrir, se reintenta. El servidor es idempotente por UUID.
- **Token expira durante sync de batch largo**: Interceptar 401, refrescar token, reintentar el batch actual.
- **Conflicto de zona horaria**: `fecha_creacion_local` incluye offset UTC (ISO 8601 con timezone). El servidor compara en UTC.
- **Dispositivo sin espacio para IndexedDB**: Capturar errores de cuota. Si no se puede guardar, mostrar: _"Almacenamiento lleno. Sincroniza los registros o libera espacio."_ La purga debería liberar espacio si hay registros antiguos sincronizados.
- **Reloj del dispositivo adelantado**: Un `fecha_creacion_local` en el futuro ganaría siempre en resolución de conflictos. El servidor puede validar que la fecha no exceda `now() + 24h` y rechazar.
- **Múltiples dispositivos con mismo usuario**: Cada dispositivo genera UUIDs únicos. No hay colisión. El servidor acepta todos.
- **Backend caído por horas**: El backoff llega a 30 min max. Los registros se acumulan en IndexedDB sin límite (la app sigue operativa).
