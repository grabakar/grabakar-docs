# Modelo de Datos

## ERD (Diagrama de Relaciones)

```
┌──────────────┐       ┌───────────┐       ┌──────────────────┐
│    Tenant    │1────N│  Usuario   │       │     LeyCaso       │
│──────────────│       │───────────│       │──────────────────│
│ id (PK)      │       │ id (PK)   │       │ id (PK)          │
│ nombre       │       │ tenant FK │       │ nombre           │
│ logo_url     │       │ nombre    │       │ descripcion      │
│ color_1      │       │ rol       │       │ activa           │
│ color_2      │       │ activo    │       └────────┬─────────┘
│ config       │       │ sucursal FK─┐             │
│ tipo_cliente │       └─────┬─────┘ │             │
└────┬─────────┘             │       │             │
     │                       │       │             │
     │1────N ┌───────────────┘       │             │
     │       │                       │             │
     │  ┌────┴───────┐               │             │
     │  │  Sucursal  │               │             │
     │  │────────────│               │             │
     │  │ id (PK)    │               │             │
     │  │ tenant FK  │───────────────┘             │
     │  │ nombre     │                             │
     │  │ activa     │                             │
     │  └────┬───────┘                             │
     │       │                                     │
     │N      │              1                 1    │
     │  ┌────┴──────────────┴──────────────────┴───┐
     └──│              Grabado                      │
        │──────────────────────────────────────────│
        │ uuid (PK)                                │
        │ patente                                  │
        │ vin_chasis                               │
        │ orden_trabajo                            │
        │ responsable_texto                        │
        │ usuario_responsable FK ──────────────────┘ (Usuario)
        │ ley_caso FK ────────────────────────────── (LeyCaso)
        │ sucursal FK ────────────────────────────── (Sucursal)
        │ tipo_movimiento                          │
        │ tipo_vehiculo                            │
        │ formato_impresion                        │
        │ fecha_creacion_local                     │
        │ fecha_sincronizacion                     │
        │ device_id                                │
        │ estado_sync                              │
        │ es_duplicado                             │
        │ tenant FK ─────────────────────────────── (Tenant)
        └──────────────┬───────────────────────────┘
                       │
                  1    │ N
        ┌──────────────┴────────────────┐
        │       ImpresionVidrio          │
        │───────────────────────────────│
        │ uuid (PK)                     │
        │ grabado FK                    │
        │ numero_vidrio                 │
        │ nombre_vidrio                 │
        │ fecha_impresion               │
        │ impreso                       │
        │ cantidad_impresiones          │
        └───────────────────────────────┘
```

## Entidades

### Tenant

Multi-tenant con white-labeling. Cada empresa de grabado es un tenant.

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| id | INT (PK, auto) | No | — | |
| nombre | VARCHAR(100) | No | — | Nombre comercial |
| logo_url | VARCHAR(255) | Sí | NULL | URL de logo para branding |
| color_primario | VARCHAR(7) | Sí | '#1976D2' | Hex color |
| color_secundario | VARCHAR(7) | Sí | '#424242' | Hex color |
| configuracion_json | JSONB | Sí | '{}' | Overrides: vidrios por tipo, formatos, etc. |
| tipo_cliente | VARCHAR(50) | Sí | '' | Clasificación: "CONCESIÓN", etc. Para reportes XLSX. **Pendiente migración.** |

```python
class Tenant(models.Model):
    nombre = models.CharField(max_length=100)
    logo_url = models.URLField(blank=True, null=True)
    color_primario = models.CharField(max_length=7, default='#1976D2')
    color_secundario = models.CharField(max_length=7, default='#424242')
    configuracion_json = models.JSONField(default=dict, blank=True)
    tipo_cliente = models.CharField(max_length=50, blank=True, default='')

    def __str__(self):
        return self.nombre
```

### Usuario

Extiende `AbstractUser` de Django. Cada usuario pertenece a exactamente un tenant.

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| id | INT (PK, auto) | No | — | Hereda de AbstractUser |
| tenant | FK(Tenant) | No | — | ON DELETE CASCADE |
| nombre_completo | VARCHAR(150) | No | — | |
| rol | VARCHAR(15) | No | — | `operador` / `supervisor` / `admin` |
| activo | BOOLEAN | No | True | |
| sucursal | FK(Sucursal) | Sí | NULL | Sucursal donde opera. **Pendiente migración.** |

```python
class Usuario(AbstractUser):
    class Rol(models.TextChoices):
        OPERADOR = 'operador'
        SUPERVISOR = 'supervisor'
        ADMIN = 'admin'

    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE, related_name='usuarios')
    nombre_completo = models.CharField(max_length=150)
    rol = models.CharField(max_length=15, choices=Rol.choices)
    activo = models.BooleanField(default=True)
    sucursal = models.ForeignKey('Sucursal', on_delete=models.SET_NULL, null=True, blank=True)

    class Meta:
        indexes = [models.Index(fields=['tenant', 'activo'])]
```

### Sucursal

Sucursal/branch dentro de un Tenant. Un cliente (Tenant) puede tener múltiples sucursales. **Pendiente migración.**

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| id | INT (PK, auto) | No | — | |
| tenant | FK(Tenant) | No | — | ON DELETE CASCADE |
| nombre | VARCHAR(150) | No | — | Nombre de la sucursal |
| activa | BOOLEAN | No | True | |

```python
class Sucursal(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE, related_name='sucursales')
    nombre = models.CharField(max_length=150)
    activa = models.BooleanField(default=True)

    class Meta:
        unique_together = ['tenant', 'nombre']
        indexes = [models.Index(fields=['tenant', 'activa'])]

    def __str__(self):
        return f'{self.tenant.nombre} - {self.nombre}'
```

### LeyCaso

Normativas legales aplicables a grabados. Se cachean en el dispositivo al login.

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| id | INT (PK, auto) | No | — | |
| nombre | VARCHAR(100) | No | — | Ej: "Ley 20.580" |
| descripcion | TEXT | Sí | NULL | |
| activa | BOOLEAN | No | True | Solo leyes activas se muestran en frontend |

```python
class LeyCaso(models.Model):
    nombre = models.CharField(max_length=100)
    descripcion = models.TextField(blank=True, null=True)
    activa = models.BooleanField(default=True)

    def __str__(self):
        return self.nombre
```

### Grabado

Registro principal. UUID generado en dispositivo. PK es UUID, no auto-increment.

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| uuid | UUID (PK) | No | — | UUID v4 generado en dispositivo |
| patente | VARCHAR(10) | No | — | Normalizada: mayúsculas, sin espacios/guiones |
| vin_chasis | VARCHAR(17) | Sí | NULL | |
| orden_trabajo | VARCHAR(50) | Sí | NULL | |
| responsable_texto | VARCHAR(150) | Sí | '' | Texto libre: operador físico. Distinto del usuario logueado. **Pendiente migración.** |
| usuario_responsable | FK(Usuario) | No | — | ON DELETE PROTECT. Mapea a NOMBRE en reportes. |
| ley_caso | FK(LeyCaso) | No | — | ON DELETE PROTECT |
| sucursal | FK(Sucursal) | Sí | NULL | Denormalizado para reportes. **Pendiente migración.** |
| tipo_movimiento | VARCHAR(15) | No | — | `venta` / `demo` / `capacitacion` |
| tipo_vehiculo | VARCHAR(10) | No | — | `auto` / `moto` |
| formato_impresion | VARCHAR(12) | No | — | `horizontal` / `vertical` |
| fecha_creacion_local | DATETIME | No | — | Timestamp del dispositivo al crear |
| fecha_sincronizacion | DATETIME | Sí | NULL | Timestamp del servidor al recibir. NULL hasta sync. |
| device_id | VARCHAR(64) | No | — | Identificador del dispositivo |
| estado_sync | VARCHAR(15) | No | 'pendiente' | `pendiente` / `sincronizado` / `error` |
| es_duplicado | BOOLEAN | No | False | True si se detectó duplicado y usuario continuó |
| tenant | FK(Tenant) | No | — | ON DELETE CASCADE |

```python
class Grabado(models.Model):
    class TipoMovimiento(models.TextChoices):
        VENTA = 'venta'
        DEMO = 'demo'
        CAPACITACION = 'capacitacion', 'Capacitación'

    class TipoVehiculo(models.TextChoices):
        AUTO = 'auto'
        MOTO = 'moto'

    class FormatoImpresion(models.TextChoices):
        HORIZONTAL = 'horizontal'
        VERTICAL = 'vertical'

    class EstadoSync(models.TextChoices):
        PENDIENTE = 'pendiente'
        SINCRONIZADO = 'sincronizado'
        ERROR = 'error'

    uuid = models.UUIDField(primary_key=True)
    patente = models.CharField(max_length=10, db_index=True)
    vin_chasis = models.CharField(max_length=17, blank=True, null=True)
    orden_trabajo = models.CharField(max_length=50, blank=True, null=True)
    responsable_texto = models.CharField(max_length=150, blank=True, default='')
    usuario_responsable = models.ForeignKey('Usuario', on_delete=models.PROTECT)
    ley_caso = models.ForeignKey('LeyCaso', on_delete=models.PROTECT)
    sucursal = models.ForeignKey('Sucursal', on_delete=models.SET_NULL, null=True, blank=True)
    tipo_movimiento = models.CharField(max_length=15, choices=TipoMovimiento.choices)
    tipo_vehiculo = models.CharField(max_length=10, choices=TipoVehiculo.choices)
    formato_impresion = models.CharField(max_length=12, choices=FormatoImpresion.choices)
    fecha_creacion_local = models.DateTimeField()
    fecha_sincronizacion = models.DateTimeField(null=True, blank=True)
    device_id = models.CharField(max_length=64)
    estado_sync = models.CharField(max_length=15, choices=EstadoSync.choices, default=EstadoSync.PENDIENTE)
    es_duplicado = models.BooleanField(default=False)
    tenant = models.ForeignKey('Tenant', on_delete=models.CASCADE)

    class Meta:
        indexes = [
            models.Index(fields=['patente', 'device_id', 'fecha_creacion_local'], name='idx_duplicado'),
            models.Index(fields=['estado_sync'], name='idx_sync_queue'),
            models.Index(fields=['fecha_creacion_local'], name='idx_purge'),
            models.Index(fields=['tenant'], name='idx_tenant'),
        ]
```

### ImpresionVidrio

Cada vidrio grabado dentro de un registro Grabado.

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| uuid | UUID (PK) | No | — | UUID v4 |
| grabado | FK(Grabado) | No | — | ON DELETE CASCADE |
| numero_vidrio | INT | No | — | Posición en secuencia (1-based) |
| nombre_vidrio | VARCHAR(50) | No | — | "Parabrisas", "Luneta", etc. |
| fecha_impresion | DATETIME | Sí | NULL | Timestamp de última impresión |
| impreso | BOOLEAN | No | False | |
| cantidad_impresiones | INT | No | 0 | Contador acumulativo (auditoría) |

```python
class ImpresionVidrio(models.Model):
    uuid = models.UUIDField(primary_key=True)
    grabado = models.ForeignKey(Grabado, on_delete=models.CASCADE, related_name='impresiones')
    numero_vidrio = models.PositiveSmallIntegerField()
    nombre_vidrio = models.CharField(max_length=50)
    fecha_impresion = models.DateTimeField(null=True, blank=True)
    impreso = models.BooleanField(default=False)
    cantidad_impresiones = models.PositiveIntegerField(default=0)

    class Meta:
        unique_together = ['grabado', 'numero_vidrio']
        ordering = ['numero_vidrio']
        indexes = [models.Index(fields=['grabado'], name='idx_impresion_grabado')]
```

## Índices

| Tabla | Índice | Campos | Propósito |
|---|---|---|---|
| Grabado | `idx_duplicado` | (patente, device_id, fecha_creacion_local) | Detección de duplicados local y en sync |
| Grabado | `idx_sync_queue` | (estado_sync) | Obtener registros pendientes de sincronización |
| Grabado | `idx_purge` | (fecha_creacion_local) | Purga de registros >30 días |
| Grabado | `idx_tenant` | (tenant) | Filtrado multi-tenant en todas las queries |
| ImpresionVidrio | `idx_impresion_grabado` | (grabado) | JOIN con Grabado |
| Usuario | (tenant, activo) | — | Listado de usuarios activos por tenant |
| Sucursal | (tenant, activa) | — | Listado de sucursales activas por tenant |

## Retención y Purga (30 días)

Registros sincronizados mayores a 30 días se eliminan. Registros no sincronizados nunca se purgan.

**Backend (Celery beat, semanal):**

```python
from django.utils import timezone
from datetime import timedelta

def purgar_registros_antiguos():
    limite = timezone.now() - timedelta(days=30)
    Grabado.objects.filter(
        fecha_creacion_local__lt=limite,
        estado_sync='sincronizado'
    ).delete()  # CASCADE borra ImpresionVidrio asociadas
```

**Frontend (al startup + cada 24h):**

```typescript
async function purgarRegistrosAntiguos(): Promise<number> {
  const limite = new Date();
  limite.setDate(limite.getDate() - 30);
  const eliminados = await db.grabados
    .where('fecha_creacion_local').below(limite.toISOString())
    .and(r => r.estado_sync === 'sincronizado')
    .delete();
  await db.impresiones_vidrio
    .where('grabado_uuid').noneOf(
      (await db.grabados.toArray()).map(g => g.uuid)
    ).delete();
  return eliminados;
}
```

## Schema IndexedDB (Frontend — Dexie.js)

```typescript
import Dexie, { Table } from 'dexie';

interface Grabado {
  uuid: string;
  patente: string;
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
  fecha_creacion_local: string;
  estado_sync: 'pendiente' | 'sincronizado' | 'error';
  fecha_sincronizacion: string | null;
  device_id: string;
  tenant_id: number;
  completado: boolean;
}

interface ImpresionVidrio {
  id?: number;
  grabado_uuid: string;
  nombre_vidrio: string;
  orden: number;
  cantidad_impresiones: number;
  omitido: boolean;
  fecha_primera_impresion: string | null;
  fecha_ultima_impresion: string | null;
}

interface Configuracion {
  clave: string;
  valor: any;
}

class GrabakarDB extends Dexie {
  grabados!: Table<Grabado, string>;
  impresiones_vidrio!: Table<ImpresionVidrio, number>;
  configuracion!: Table<Configuracion, string>;

  constructor() {
    super('grabakar');
    this.version(1).stores({
      grabados: 'uuid, patente, fecha_creacion_local, estado_sync, tenant_id',
      impresiones_vidrio: '++id, grabado_uuid, nombre_vidrio',
      configuracion: 'clave'
    });
  }
}

export const db = new GrabakarDB();
```

Los campos listados en `stores()` son solo los indexados. Dexie almacena todos los campos del objeto, pero solo los declarados en el schema son buscables via `.where()`.
