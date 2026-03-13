# GrabaKar - Especificación Técnica: App de Grabado de Patentes

**Versión:** 2.0  
**Fecha:** 2026-03-04  
**Estado:** Borrador para revisión

---

## 1. Resumen del Producto

Aplicación B2B para el registro y gestión del grabado de seguridad de patentes vehiculares en vidrios, en cumplimiento con la normativa chilena (Ley 20.580 y reglamentos asociados). El grabado se realiza frecuentemente en terreno (estacionamientos subterráneos, concesionarios), por lo que la operación offline es crítica.

**Nombre comercial:** GrabaKar (configurable via white-labeling; nunca hardcodeado).  
**Idioma:** Español (Chile). Toda la UI, mensajes de error, reportes y documentación de usuario deben estar en español. Terminología: "Patente" (no "Plate"), "Grabado" (no "Engraving"), "Vidrio" (no "Glass").  
**Mercado objetivo:** Empresas de grabado de patentes en Chile (concesionarios, talleres, operadores móviles).

---

## 2. Arquitectura General

### 2.1 Repositorios

| Repositorio | Contenido | Stack |
|---|---|---|
| `grabakar-frontend` | Webapp React, exportable a Ionic | React 18+, TypeScript, Vite, Capacitor/Ionic |
| `grabakar-backend` | API REST, lógica de negocio, sincronización | Django 5+, Django REST Framework, PostgreSQL 16+, Celery, Redis |
| `grabakar-docs` | Documentación completa del proyecto | Markdown |
| `grabakar-infra` (condicional) | IaC, configs de deployment, analytics DB | Terraform/Pulumi, Docker Compose, scripts de CI/CD |

El repositorio de infraestructura se crea solo si se justifica (ej: base de datos separada para analytics, pipelines de CI/CD complejos, o configs de GCP que no pertenecen al backend).

### 2.2 Stack Técnico

**Frontend:**
- React 18+ con TypeScript y Vite como bundler
- Fase 1: Webapp responsive (PWA con service workers para offline)
- Fase 2: Exportar a Ionic/Capacitor para acceso nativo a Bluetooth y cámara
- Estado local: IndexedDB via Dexie.js (o equivalente) para la cola offline
- UI: Componentes en español, responsive para tablet Android (target principal) y móvil

**Backend:**
- Django 5+ con Django REST Framework
- PostgreSQL 16+ como base de datos principal
- Celery + Redis para tareas asíncronas (sincronización, generación de reportes)
- Docker Compose para entorno local y staging
- Gunicorn + Nginx en producción

**Despliegue:**
- Target primario: GCP (Cloud Run o GKE para el backend, Cloud SQL para PostgreSQL, Cloud Storage para archivos)
- Diseño agnóstico a la plataforma: toda la infraestructura se define via Docker Compose localmente. El backend no debe tener dependencias directas a servicios de GCP; usar abstracciones (ej: django-storages para archivos, variables de entorno para configuración)
- CI/CD: GitHub Actions (un workflow por repositorio)

### 2.3 Diagrama de Arquitectura Offline-First

```
┌──────────────────────────────────────┐
│           DISPOSITIVO (Tablet/Móvil)          │
│                                      │
│  React App (PWA / Ionic)             │
│  ├─ UI en español                    │
│  ├─ IndexedDB (cola offline)         │
│  │   └─ Registros de grabado         │
│  │   └─ Cola de sincronización       │
│  ├─ Service Worker (cache, offline)  │
│  ├─ Bluetooth PrintService           │
│  └─ CameraService (OCR futuro)       │
│                                      │
│  Estado de red: online / offline     │
│  ├─ Online  → sync automático        │
│  └─ Offline → almacena en cola local │
└──────────────┬───────────────────────┘
               │ HTTPS (cuando hay red)
               ▼
┌──────────────────────────────────────┐
│           BACKEND (Django)                    │
│                                      │
│  API REST (DRF)                      │
│  ├─ /api/auth/ (login, refresh)      │
│  ├─ /api/grabados/ (CRUD + batch)    │
│  ├─ /api/sync/ (upload cola)         │
│  ├─ /api/reportes/ (diario, mensual) │
│  └─ /api/config/ (branding, leyes)   │
│                                      │
│  Celery Workers                      │
│  ├─ Procesamiento de sync batches    │
│  ├─ Generación de reportes CSV       │
│  └─ Purga de datos antiguos          │
│                                      │
│  PostgreSQL                          │
│  ├─ Datos de grabado                 │
│  ├─ Usuarios y tenants               │
│  └─ Configuración de branding        │
└──────────────────────────────────────┘
```

---

## 3. Reglas de Negocio Críticas

### 3.1 Offline-First Estricto

- El dispositivo debe operar por días sin conexión a internet.
- Todos los datos se guardan en IndexedDB local con cola de sincronización.
- Al detectar conectividad, la sincronización ocurre automáticamente en segundo plano.
- El frontend es la fuente de verdad para datos no sincronizados; conflictos se resuelven a favor del dispositivo.

### 3.2 Gestión de Sesión Resiliente

- El login requiere conectividad (Wi-Fi o datos móviles).
- La sesión expira después de 3 horas.
- **Problema crítico:** Si la sesión expira mientras el dispositivo está offline, el usuario NO debe perder acceso ni datos.
- **Mecanismo de fallback:** Al hacer login exitoso, se genera un token offline (PIN local o extensión de token) que permite operar el dispositivo por un período extendido (configurable, default 72 horas) sin conectividad. Este token se almacena cifrado en el dispositivo. Al reconectarse, se fuerza una re-autenticación.

### 3.3 Retención Local

- Los registros se almacenan localmente por 30 días.
- Una tarea automatizada (cron local o service worker) purga registros mayores a 30 días que ya hayan sido sincronizados exitosamente.
- Registros no sincronizados NUNCA se purgan, independiente de su antigüedad.

### 3.4 Detección de Duplicados

- Si se ingresa una patente que ya existe en el dispositivo (últimos 30 días), el sistema muestra una alerta con la fecha del registro previo.
- El operador puede continuar (puede haber razones legítimas) pero queda registrado como potencial duplicado.

### 3.5 Protección contra Exploit de Reimpresión

- Una vez que un registro se guarda, los campos de patente y VIN se bloquean (solo lectura).
- El botón "Guardar" cambia a "Imprimir" después del primer guardado.
- Cada impresión incrementa un contador (`cantidad_impresiones`) que se reporta al backend.
- Reimpresiones quedan registradas con timestamp para auditoría.

### 3.6 Trazabilidad

Metadatos obligatorios por cada registro:
- `uuid`: Generado en el dispositivo (UUID v4)
- `fecha_creacion_local`: Timestamp del dispositivo al crear
- `fecha_sincronizacion`: Timestamp del servidor al recibir
- `usuario_responsable`: ID del operador
- `device_id`: Identificador del dispositivo
- `cantidad_impresiones`: Contador acumulativo
- `estado_sync`: `pendiente | sincronizado | error`

---

## 4. Modelo de Datos

### 4.1 Entidades Principales

**Grabado** (registro principal)
| Campo | Tipo | Requerido | Notas |
|---|---|---|---|
| uuid | UUID v4 | Sí | Generado en dispositivo |
| patente | VARCHAR(10) | Sí | Mín. 6 caracteres |
| patente_confirmacion | VARCHAR(10) | Sí | Debe coincidir con `patente` |
| vin_chasis | VARCHAR(17) | No | VIN/Chassis |
| orden_trabajo | VARCHAR(50) | No | Orden de compra |
| usuario_responsable | FK Usuario | Sí | Operador que realiza el grabado |
| ley_caso | FK LeyCaso | Sí | Ley/normativa aplicable |
| tipo_movimiento | ENUM | Sí | `venta`, `demo`, `capacitacion` |
| tipo_vehiculo | ENUM | Sí | `auto`, `moto` |
| formato_impresion | ENUM | Sí | `horizontal`, `vertical` |
| fecha_creacion_local | DATETIME | Sí | Timestamp del dispositivo |
| fecha_sincronizacion | DATETIME | No | Null hasta sync exitoso |
| device_id | VARCHAR(64) | Sí | Identificador del dispositivo |
| estado_sync | ENUM | Sí | `pendiente`, `sincronizado`, `error` |
| es_duplicado | BOOLEAN | No | True si se detectó duplicado |
| tenant_id | FK Tenant | Sí | Multi-tenant |

**ImpresionVidrio** (cada vidrio grabado)
| Campo | Tipo | Requerido | Notas |
|---|---|---|---|
| uuid | UUID v4 | Sí | |
| grabado_id | FK Grabado | Sí | |
| numero_vidrio | INT | Sí | Posición en la secuencia |
| nombre_vidrio | VARCHAR(50) | Sí | Ej: "Parabrisas", "Lateral izq." |
| fecha_impresion | DATETIME | Sí | Timestamp de la impresión física |
| impreso | BOOLEAN | Sí | |
| cantidad_impresiones | INT | Sí | Default 0 |

**Usuario**
| Campo | Tipo | Requerido | Notas |
|---|---|---|---|
| id | INT | Sí | |
| username | VARCHAR(50) | Sí | |
| nombre_completo | VARCHAR(100) | Sí | |
| rol | ENUM | Sí | `operador`, `supervisor`, `admin` |
| tenant_id | FK Tenant | Sí | |
| activo | BOOLEAN | Sí | |

**Tenant** (multi-tenant / white-labeling)
| Campo | Tipo | Requerido | Notas |
|---|---|---|---|
| id | INT | Sí | |
| nombre | VARCHAR(100) | Sí | Nombre comercial |
| logo_url | VARCHAR(255) | No | |
| color_primario | VARCHAR(7) | No | Hex color |
| color_secundario | VARCHAR(7) | No | Hex color |
| configuracion_json | JSONB | No | Overrides de configuración |

**LeyCaso** (normativa aplicable)
| Campo | Tipo | Requerido | Notas |
|---|---|---|---|
| id | INT | Sí | |
| nombre | VARCHAR(100) | Sí | Ej: "Ley 20.580" |
| descripcion | TEXT | No | |
| activa | BOOLEAN | Sí | |

### 4.2 Índices Clave

- `Grabado`: índice compuesto en (`patente`, `device_id`, `fecha_creacion_local`) para detección de duplicados
- `Grabado`: índice en `estado_sync` para queries de cola de sincronización
- `Grabado`: índice en `fecha_creacion_local` para la purga de 30 días
- `Grabado`: índice en `tenant_id` para queries multi-tenant
- `ImpresionVidrio`: índice en `grabado_id`

---

## 5. Flujo de UI

### 5.1 Pantallas Principales

```
Login → Lista de Grabados (Home) → Formulario de Grabado → Flujo Multi-Vidrio → Historial/Cola de Sync
                                                                                    → Reportes
```

### 5.2 Login
- Campos: usuario y contraseña
- Requiere conectividad
- Al login exitoso: descarga configuración (leyes, branding) y genera token offline
- Si hay token offline válido y no hay red: permite acceso con advertencia visual

### 5.3 Home (Lista de Grabados)
- Lista de grabados recientes (últimos 30 días en dispositivo)
- Botón "+" prominente para crear nuevo grabado
- Indicador visual de estado de sincronización (ícono por registro)
- Indicador de conectividad en header
- Menú hamburguesa (arriba izquierda):
  - Configuración de impresión (horizontal/vertical)
  - Tipo de vehículo (auto/moto - afecta tamaño y formato)
  - Cola de sincronización
  - Cerrar sesión

### 5.4 Formulario de Grabado (botón "+")
- **Patente** (obligatorio, mín. 6 caracteres, alfanumérico)
- **Confirmación de Patente** (re-ingreso, debe coincidir)
- **VIN/Chassis** (opcional)
- **Orden de Trabajo** (opcional)
- **Responsable** (pre-llenado con usuario actual, editable)
- **Ley / Caso** (selector basado en LeyCaso activas)
- **Tipo de Movimiento** (selector: Venta, Demo, Capacitación)
- Botón **"Guardar"**: valida campos, guarda en local, inicia flujo multi-vidrio
- Validación: si la patente ya existe en el dispositivo, mostrar alerta antes de guardar

### 5.5 Flujo Multi-Vidrio (Poka-Yoke)
- Después de guardar datos generales, la UI itera por cada vidrio a grabar
- En la pantalla de impresión de cada vidrio:
  - La **patente se muestra en tipografía grande** para validación visual del operador
  - Nombre del vidrio actual (ej: "Vidrio 3 de 6 - Lateral Derecho")
  - Botón "Imprimir" (envía al printer via Bluetooth)
  - Botón "Siguiente Vidrio" / "Finalizar" (en el último)
- Cada impresión incrementa el contador
- Al finalizar todos los vidrios, retorna a Home

### 5.6 Reportes
- Reporte diario: resumen de todas las patentes grabadas en el día
- Reporte mensual: acumulado del mes
- Formato CSV (generado en backend, descargable o enviado por email)
- Filtrable por operador, tipo de movimiento, fechas

---

## 6. APIs y Sincronización

### 6.1 Endpoints Principales

```
POST   /api/auth/login/          → { token, refresh_token, offline_token, config }
POST   /api/auth/refresh/        → { token }
POST   /api/sync/upload/         → Batch upload de cola local
GET    /api/sync/status/         → Estado de sincronización
GET    /api/config/              → Branding, leyes activas, configuración
GET    /api/reportes/diario/     → CSV del día
GET    /api/reportes/mensual/    → CSV del mes
GET    /api/grabados/            → Lista paginada (para admin/supervisor)
```

### 6.2 Payload de Sincronización (Batch Upload)

```json
{
  "device_id": "abc-123",
  "batch": [
    {
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
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
      "vidrios": [
        {
          "uuid": "...",
          "numero_vidrio": 1,
          "nombre_vidrio": "Parabrisas",
          "fecha_impresion": "2026-03-04T10:32:00-03:00",
          "cantidad_impresiones": 1
        }
      ]
    }
  ]
}
```

### 6.3 Resolución de Conflictos

- El dispositivo es la fuente de verdad para datos de grabado.
- El servidor acepta el batch y marca con `fecha_sincronizacion` al recibirlo.
- Si el UUID ya existe en el servidor, se actualiza solo si `fecha_creacion_local` del batch es más reciente.
- Los campos `fecha_creacion_local` (timestamp del dispositivo) y `fecha_sincronizacion` (timestamp del servidor) son siempre distintos y ambos se preservan.

### 6.4 Lógica de Reportes

- **Reporte diario:** Generado via Celery beat (cron) a las 23:59 hora local (Chile, UTC-3/-4). Filtra por `fecha_creacion_local` del día.
- **Registros tardíos:** Si un dispositivo sincroniza datos de días anteriores, el backend regenera los reportes afectados y notifica al supervisor.
- **Campos del CSV:** `uuid, patente, operador, fecha_registro, cantidad_impresiones, fecha_impresion, tipo_movimiento, tipo_vehiculo, device_id, estado_sync, fecha_sincronizacion`

---

## 7. Bluetooth e Impresión

### 7.1 Abstracción del PrintService

```typescript
interface PrintService {
  discover(): Promise<PrinterDevice[]>;
  connect(deviceId: string): Promise<void>;
  disconnect(): Promise<void>;
  print(payload: PrintPayload): Promise<PrintResult>;
  getStatus(): Promise<PrinterStatus>;
}

interface PrintPayload {
  patente: string;
  formato: 'horizontal' | 'vertical';
  tipoVehiculo: 'auto' | 'moto';
  copias: number;
}
```

### 7.2 Protocolo Recomendado

- **ESC/POS** para impresoras térmicas/etiquetadoras estándar (mayor compatibilidad)
- **ZPL** como alternativa para impresoras Zebra industriales
- La implementación debe ser agnóstica al protocolo: el `PrintService` recibe un `PrintPayload` y la implementación concreta traduce al protocolo de la impresora conectada.

### 7.3 Fases de Implementación Bluetooth

- Fase 1 (MVP): Sin Bluetooth. La pantalla de impresión muestra la patente en grande para grabado manual.
- Fase 2: Integración Bluetooth via Web Bluetooth API (PWA) o Capacitor BLE plugin (Ionic).
- Fase 3: Soporte multi-protocolo (ESC/POS + ZPL).

---

## 8. Cámara y OCR

- Punto de extensión para escaneo futuro de VIN/Patente via cámara del dispositivo.
- Fase 1: No implementado. El UI incluye un botón de cámara deshabilitado con tooltip "Próximamente".
- Fase 2: Integrar via Capacitor Camera plugin. El OCR puede ser local (Tesseract.js) o via API backend.
- La interfaz debe definirse desde el inicio:

```typescript
interface ScannerService {
  scanPatente(): Promise<{ valor: string; confianza: number }>;
  scanVIN(): Promise<{ valor: string; confianza: number }>;
}
```

---

## 9. Multi-Tenant y White-Labeling

- Cada tenant tiene su branding (nombre, logo, colores) almacenado en la tabla `Tenant`.
- Al hacer login, la configuración de branding se descarga y cachea localmente.
- Cero valores hardcodeados: "GrabaKar" es el nombre default del tenant, no una constante en el código.
- CSS variables o un theme provider de React para inyectar colores dinámicamente.
- El logo se carga desde URL configurable.

---

## 10. Roles y Permisos

| Rol | Permisos |
|---|---|
| `operador` | Crear grabados, imprimir, ver sus propios registros |
| `supervisor` | Todo lo de operador + ver registros de todos los operadores del tenant + descargar reportes |
| `admin` | Todo lo de supervisor + gestionar usuarios + configurar tenant/branding |

---

## 11. Testing

### 11.1 Testing Automático

**Frontend (`grabakar-frontend`):**
- Unit tests: Vitest + React Testing Library
- Tests de componentes: cada componente de formulario y flujo multi-vidrio
- Tests de integración: flujo completo de login → crear grabado → imprimir (con mocks)
- Tests offline: verificar que IndexedDB almacena correctamente cuando no hay red
- E2E: Playwright para flujos críticos (login, crear grabado, sincronización)
- Coverage mínimo target: 70% en lógica de negocio (sync, validaciones, cola)

**Backend (`grabakar-backend`):**
- Unit tests: pytest + Django test framework
- Tests de API: cada endpoint con casos de éxito y error
- Tests de sincronización: batch upload con datos válidos, duplicados, conflictos
- Tests de reportes: generación de CSV, manejo de datos tardíos
- Tests de permisos: verificar que cada rol accede solo a lo permitido
- Coverage mínimo target: 80%
- Fixtures: factory_boy para datos de prueba

**CI/CD:**
- GitHub Actions ejecuta tests en cada PR
- Lint (ruff para Python, ESLint para TypeScript) como gate obligatorio
- Tests deben pasar antes de merge

### 11.2 Testing Manual

**Protocolo de pruebas manuales (antes de cada release):**

1. **Login/Sesión:**
   - Login con credenciales válidas (online)
   - Intentar login sin conexión (debe fallar con mensaje claro)
   - Verificar que el token offline funciona después de perder conexión
   - Verificar expiración de sesión (3h) y comportamiento offline post-expiración

2. **Creación de Grabado:**
   - Crear grabado con todos los campos
   - Intentar guardar sin campos obligatorios (validación)
   - Ingresar patente con menos de 6 caracteres (validación)
   - Ingresar patente que no coincide en confirmación (validación)
   - Ingresar patente duplicada (alerta)

3. **Flujo Multi-Vidrio:**
   - Completar flujo completo (todos los vidrios)
   - Verificar que la patente se muestra en grande
   - Verificar que el contador de impresiones incrementa

4. **Offline:**
   - Crear 5+ grabados sin conexión
   - Reconectar y verificar sincronización automática
   - Verificar que los registros aparecen en el backend con timestamps correctos

5. **Impresión:**
   - (Fase 2+) Conectar impresora Bluetooth, imprimir en formato horizontal y vertical
   - Verificar formato auto vs. moto

6. **Reportes:**
   - Verificar que el CSV diario incluye todos los registros del día
   - Verificar que registros tardíos (offline) se incluyen correctamente
   - Verificar filtros por operador y tipo de movimiento

7. **Exploit de reimpresión:**
   - Después de guardar, intentar modificar patente (debe estar bloqueada)
   - Verificar que el botón cambia de "Guardar" a "Imprimir"

---

## 12. Plan de Implementación (Backlog por Fases)

### Fase 0: Infraestructura y Setup
- Crear repositorios (frontend, backend, docs)
- Configurar Docker Compose para desarrollo local (Django + PostgreSQL + Redis)
- Configurar CI/CD básico (GitHub Actions: lint + tests)
- Crear `.cursorrules` por repositorio
- Crear estructura de documentación

### Fase 1: MVP (Core Offline)
- Login/autenticación (JWT + token offline)
- Formulario de creación de grabado (validaciones)
- Almacenamiento local (IndexedDB)
- Lista de grabados (home)
- Flujo multi-vidrio (sin Bluetooth, solo visual)
- Purga automática de 30 días
- API de sincronización (batch upload)
- Multi-tenant básico (branding via config)

### Fase 2: Sincronización + Reportes
- Sync automático en segundo plano
- Detección de conectividad
- Resolución de conflictos
- Generación de reportes CSV (diario y mensual)
- Envío de reportes por email (Celery)
- Dashboard de supervisor

### Fase 3: Bluetooth
- Integración con impresoras via Bluetooth
- Soporte ESC/POS
- Configuración de formato (horizontal/vertical, auto/moto)

### Fase 4: Cámara + OCR
- Escaneo de patente via cámara
- Escaneo de VIN via cámara
- Validación asistida por OCR

### Fase 5: Ionic/Mobile
- Exportar webapp a Ionic/Capacitor
- Compilar para Android (target principal)
- Optimizar UX para tablet
- Publicar en Play Store (distribución interna o pública según decisión de negocio)

---

## 13. Organización de Documentación (`grabakar-docs`)

```
grabakar-docs/
├── README.md                           # Índice general y guía de navegación
├── producto/                           # Descripción general del producto
│   ├── VISION.md                       # Visión, misión, propuesta de valor
│   ├── GLOSARIO.md                     # Terminología (Patente, Grabado, etc.)
│   └── ROLES_USUARIOS.md              # Perfiles de usuario y permisos
├── features/                           # Especificaciones de features
│   ├── LOGIN_SESION.md
│   ├── GRABADO_PATENTE.md
│   ├── FLUJO_MULTI_VIDRIO.md
│   ├── SINCRONIZACION_OFFLINE.md
│   ├── REPORTES.md
│   ├── BLUETOOTH_IMPRESION.md
│   ├── CAMARA_OCR.md
│   └── WHITE_LABELING.md
├── tecnico/                            # Especificaciones técnicas
│   ├── ARQUITECTURA.md                # Diagrama de arquitectura, decisiones
│   ├── MODELO_DATOS.md                # ERD, entidades, índices
│   ├── API_CONTRACTS.md               # Endpoints, payloads, autenticación
│   ├── SYNC_STRATEGY.md              # Offline-first, resolución de conflictos
│   ├── TESTING.md                     # Estrategia de testing automático y manual
│   ├── DEPLOYMENT.md                  # Docker Compose, GCP, CI/CD
│   └── SEGURIDAD.md                   # Autenticación, tokens, cifrado local
├── planificacion/                      # Planning y roadmap
│   ├── ROADMAP.md                     # Fases, milestones, fechas tentativas
│   ├── BACKLOG.md                     # Backlog priorizado
│   └── ciclos/                        # PDCA cycles
│       └── PDCA_YYYY-MM-DD_<tema>.md
├── prompts/                            # Prompts para agentes de desarrollo
│   ├── AGENT_RULES.md                 # Reglas generales para agentes IA
│   ├── FRONTEND_AGENT.md             # Prompt base para tareas de frontend
│   ├── BACKEND_AGENT.md              # Prompt base para tareas de backend
│   └── REVIEW_AGENT.md               # Prompt para revisión de código
└── referencias/                        # Material de referencia
    ├── LEY_20580.md                   # Resumen de la normativa aplicable
    └── CURSOR_WORKFLOW.md             # Metodología de trabajo con Cursor
```

---

## 14. Reglas de Desarrollo (Agentes y Desarrolladores)

### 14.1 Metodología

- **PDCA:** CHECK (evaluar estado actual) → ACT (diagnosticar problemas) → PLAN (diseñar solución) → DO (implementar, testear, desplegar).
- Cada ciclo de mejora se documenta en `grabakar-docs/planificacion/ciclos/`.
- Desarrollo orientado a evaluación: primero definir criterio de éxito, luego implementar.

### 14.2 Código

- Minimizar líneas de código. Sin bloat.
- Sin logs excepto para debugging activo.
- Menos archivos con más funcionalidad, no muchos archivos con poco.
- Organizar en carpetas: `utils/`, `scripts/`, `prompts/`.
- Comentarios solo para intención no obvia, trade-offs, o restricciones.

### 14.3 Trabajo Paralelo con Agentes

El proyecto está diseñado para permitir trabajo paralelo con múltiples agentes de IA:

- **Repositorios separados:** Frontend y backend son independientes. Dos agentes pueden trabajar simultáneamente sin conflictos.
- **Feature specs como contratos:** Cada feature en `grabakar-docs/features/` define el comportamiento esperado. Un agente de frontend y uno de backend pueden implementar la misma feature en paralelo usando el spec como contrato.
- **API contracts primero:** Los contratos de API (`grabakar-docs/tecnico/API_CONTRACTS.md`) se definen antes de implementar. Frontend mockea las APIs; backend implementa contra el contrato.
- **`.cursorrules` por repositorio:** Cada repo tiene su propio archivo `.cursorrules` con contexto específico (stack, convenciones, ubicación de docs).
- **Tasks atómicas:** El backlog se descompone en tareas que un solo agente puede completar en una sesión (max ~1 hora de trabajo). Cada tarea tiene: descripción, criterio de aceptación, archivos relevantes, dependencias.
- **Branch por tarea:** Cada tarea se desarrolla en una branch separada. Naming: `feature/<fase>-<nombre-corto>` (ej: `feature/f1-login-auth`).

### 14.4 Herramientas

- **@mantic** para navegación de código (rápido, estructural).
- **@exa** para referencias externas (docs de librerías, ejemplos de GitHub).
- **Cursor browser MCP** para testing de frontend.
- Preferir herramientas especializadas sobre comandos de terminal.

### 14.5 Documentación

- La documentación es de importancia crítica para este proyecto.
- Toda decisión arquitectónica se documenta con razón y alternativas consideradas.
- Features se especifican antes de implementar.
- El repositorio de docs (`grabakar-docs`) es la fuente de verdad para requirements y specs.

---

## 15. Supuestos y Decisiones Pendientes

| # | Supuesto / Decisión | Estado | Alternativas |
|---|---|---|---|
| 1 | La lista de vidrios por vehículo es configurable (no fija) | **Supuesto** | A: lista fija de 6 vidrios estándar. B: configurable por tipo de vehículo. |
| 2 | El formato de impresión para moto difiere en tamaño, no en contenido | **Supuesto** | Verificar con negocio qué exactamente cambia entre auto y moto. |
| 3 | Los reportes se envían por email al supervisor | **Supuesto** | A: email. B: descarga desde la app. C: ambos. |
| 4 | La Ley 20.580 es la normativa principal aplicable | **Verificar** | Confirmar con el área legal del cliente. |
| 5 | El token offline de 72 horas es suficiente | **Verificar** | Puede necesitar ser configurable por tenant. |
| 6 | El repositorio de infraestructura se necesita | **Pendiente** | Crear solo si surge la necesidad (analytics DB, IaC complejo). |
| 7 | Base de datos separada para analytics | **Pendiente** | A: misma PostgreSQL con schema separado. B: BigQuery/otra BD analítica. Decidir en Fase 2. |
| 8 | Distribución de la app: Play Store vs. distribución interna (APK) | **Pendiente** | Decidir en Fase 5. |

---

## 16. Scaffold de Código (Interfaces Core)

Estas interfaces se proveen como referencia para la implementación. No son código final.

### 16.1 SyncManager (Frontend)

```typescript
interface SyncManager {
  getQueueSize(): Promise<number>;
  getPendingRecords(): Promise<Grabado[]>;
  enqueue(grabado: Grabado): Promise<void>;
  syncAll(): Promise<SyncResult>;
  isOnline(): boolean;
  onConnectivityChange(callback: (online: boolean) => void): void;
}

interface SyncResult {
  exitoso: number;
  fallido: number;
  errores: SyncError[];
}
```

### 16.2 OfflineAuthService (Frontend)

```typescript
interface OfflineAuthService {
  login(username: string, password: string): Promise<AuthResult>;
  validateOfflineToken(): Promise<boolean>;
  getStoredUser(): Promise<Usuario | null>;
  logout(): Promise<void>;
  isSessionValid(): boolean;
}
```

### 16.3 Backend Sync View (Django)

```python
# grabakar-backend/api/views/sync.py
class SyncUploadView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        """Recibe batch de grabados desde dispositivo."""
        serializer = SyncBatchSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        resultado = SyncService.procesar_batch(
            device_id=serializer.validated_data['device_id'],
            batch=serializer.validated_data['batch'],
            usuario=request.user
        )
        return Response(resultado, status=status.HTTP_200_OK)
```
