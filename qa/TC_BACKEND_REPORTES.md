# Casos de Prueba: Reportes Backend y Tareas en Segundo Plano

## Módulo: Reportes (Backend API & Celery)
**Nivel**: API Testing y Data Consistency
**Prioridad Base**: Alta (P1)

### TC-REPORTES-B-001: Generación Reporte Diario JSON/CSV (Tenant)
**Prioridad**: P1 | **Severidad**: S2
**Setup**: `api/tests/test_reportes.py`
**Precondiciones**:
- Base de datos con grabados de múltiples operadores (incluido uno con `fecha_creacion_local` de un día anterior).
- Usuario de tipo `supervisor` autenticado, perteneciente al tenant 1.
**Pasos**:
1. Hacer una petición `GET` a `/api/v1/reportes/diario/?format=json`.
2. Hacer una petición `GET` a `/api/v1/reportes/diario/?format=csv`.
**Resultado Esperado**:
- Para JSON, se retorna un array válido de diccionarios.
- Para CSV, se retorna un archivo CSV con las cabeceras correctas.
- Ambos formatos devuelven **exclusivamente** registros creados "hoy", pertenecientes **únicamente** al tenant del supervisor logueado.

### TC-REPORTES-B-002: Generación Reporte Plataforma XLSX (Cross-Tenant)
**Prioridad**: P0 | **Severidad**: S2
**Setup**: Endpoint XLSX con `openpyxl`.
**Precondiciones**:
- Base de datos con múltiples sucursales, grabados, y clientes repartidos en diferentes tenants.
- Tiempos de `fecha_sincronizacion` guardados de hoy y ayer.
- Usuario de tipo `admin` autenticado.
**Pasos**:
1. Hacer una petición `GET` a `/api/v1/reportes/plataforma/`.
2. Validar estructura del blob de la respuesta.
**Resultado Esperado**:
- El endpoint responde con HTTP 200 y el mimetype correspondiente a un XLSX (`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`).
- Descargar el blob genera un `.xlsx` no corrupto.
- El archivo contiene 3 pestañas estructurales como se define en `REPORTES.md`.
- El reporte agrupa registros basándose en `fecha_sincronizacion` (y no de creación local).

### TC-REPORTES-B-003: Aislamiento Rol-Dependiente de Plataforma
**Prioridad**: P0 | **Severidad**: S1
**Precondiciones**:
- Sistema corriendo.
- Peticiones por consola o Postman.
**Pasos**:
1. Con credenciales de un `operador`, hacer un GET a `/api/v1/reportes/diario/`.
2. Con credenciales de un `supervisor`, hacer un GET a `/api/v1/reportes/plataforma/`.
**Resultado Esperado**:
- Operador obtiene un `403 Forbidden` intentando acceder a cualquier reporte (diario, mensual, plataforma).
- Supervisor obtiene un `403 Forbidden` intentando acceder a `/api/v1/reportes/plataforma/` (exclusivo para Admins).

### TC-REPORTES-B-004: Ejecución Asíncrona (Celery Task Diario)
**Prioridad**: P2 | **Severidad**: S2
**Setup**: Worker de Celery corriendo + Redis (se puede emular llamando directo a la task `.delay()`).
**Precondiciones**:
- Entorno de Testing o Staging con infraestructura Celery operativa.
- Registros que cumplan criterios del reporte diario recién cargados.
**Pasos**:
1. Forzar la ejecución de la tarea periódica encargada de generar el reporte diario.
2. Comprobar los logs de Worker y Celery Beat.
3. Verificar si el Storage (`django-storages` S3/GCS o FileSystem) registra el archivado del XLSX.
**Resultado Esperado**:
- La tarea se ejecuta exitosamente (`状态: SUCCESS`).
- El archivo XLSX consolidado se aloja en el Object Storage.
- No ocurren pérdidas de contexto ni excepciones de base de datos durante el proceso batch asíncrono.
