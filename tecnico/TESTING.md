# Estrategia de Testing

## Stack de Testing

| Capa | Herramienta | Propósito |
|---|---|---|
| Frontend unit | Vitest | Lógica de negocio, utils, servicios |
| Frontend components | React Testing Library | Renderizado, interacción, formularios |
| Frontend E2E | Playwright | Flujos completos en browser real |
| Backend unit/integration | pytest + Django test client | Endpoints, serializers, services |
| Backend fixtures | factory_boy | Generación de datos de prueba |
| QA Test Agent | ADB + Android Emulator + gh CLI | Pruebas end-to-end automatizadas del .apk en emulador |
| Lint frontend | ESLint + TypeScript strict | Calidad de código |
| Lint backend | ruff | Calidad de código Python |

## Coverage Targets

- **Backend**: 80% mínimo en `api/` (modelos, views, serializers, services).
- **Frontend**: 70% mínimo en lógica de negocio (sync, validaciones, auth). No se mide coverage de UI pura.

## Qué Testear — Frontend

### Unit tests (Vitest)

**Validación de formularios:**
- Normalización de patente (mayúsculas, sin espacios/guiones).
- Regex de patente (`/^[A-Z0-9]{6,8}$/` post-normalización).
- Validación de VIN (17 chars alfanuméricos o vacío).
- Match de patente vs confirmación.

**IndexedDB / Dexie:**
- CRUD de grabados en IndexedDB (usar `fake-indexeddb` para tests).
- Detección de duplicados (patente existente en últimos 30 días).
- Purga de registros sincronizados >30 días.
- Transacciones atómicas (grabado + impresiones_vidrio).

**SyncManager:**
- Encolamiento de registros pendientes.
- Construcción del payload de batch (max 100 registros).
- Manejo de respuesta parcial (algunos exitosos, algunos fallidos).
- Backoff exponencial: verificar intervalos 5s → 15s → 45s → 2min → 5min → 30min.
- Reset de backoff al cambiar offline → online.

**AuthService:**
- Almacenamiento y recuperación de tokens.
- Validación de expiración de access token.
- Fallback a offline token cuando no hay red.
- Logout limpia tokens y datos sensibles.

**Detección de duplicados:**

```typescript
describe('detectarDuplicado', () => {
  it('retorna grabado si patente existe en últimos 30 días', async () => {
    await db.grabados.add(mockGrabado({ patente: 'BBDF12', fecha_creacion_local: hace(5, 'dias') }));
    const resultado = await detectarDuplicado('BBDF12');
    expect(resultado).not.toBeNull();
    expect(resultado!.patente).toBe('BBDF12');
  });

  it('retorna null si patente existe pero tiene más de 30 días', async () => {
    await db.grabados.add(mockGrabado({ patente: 'BBDF12', fecha_creacion_local: hace(35, 'dias') }));
    expect(await detectarDuplicado('BBDF12')).toBeNull();
  });

  it('retorna null si patente no existe', async () => {
    expect(await detectarDuplicado('ZZZZ99')).toBeNull();
  });
});
```

### Component tests (React Testing Library)

- Formulario de grabado: campos requeridos, validación en blur, deshabilitación post-guardado.
- Flujo multi-vidrio: progreso, botón imprimir incrementa contador, finalización.
- Indicador de conectividad: cambio de estado online/offline.
- Diálogo de duplicados: muestra fecha del registro anterior, permite continuar.

## Qué Testear — Backend

### Endpoints API (pytest + Django test client)

**Sync upload (`POST /api/v1/sync/upload/`):**

```python
class TestSyncUpload:
    def test_batch_valido_crea_grabados(self, api_client, usuario_operador):
        payload = build_sync_payload(count=3)
        response = api_client.post('/api/v1/sync/upload/', payload, format='json')
        assert response.status_code == 200
        assert response.data['exitosos'] == 3
        assert Grabado.objects.count() == 3

    def test_batch_con_uuid_duplicado_actualiza_si_mas_reciente(self, api_client, grabado_existente):
        payload = build_sync_payload(uuid=grabado_existente.uuid, fecha_creacion_local=ahora())
        response = api_client.post('/api/v1/sync/upload/', payload, format='json')
        assert response.data['exitosos'] == 1

    def test_batch_con_errores_parciales_no_detiene_otros(self, api_client):
        payload = build_sync_payload(count=3, invalid_indices=[1])
        response = api_client.post('/api/v1/sync/upload/', payload, format='json')
        assert response.data['exitosos'] == 2
        assert response.data['fallidos'] == 1

    def test_batch_vacio_retorna_400(self, api_client):
        response = api_client.post('/api/v1/sync/upload/', {'device_id': 'x', 'batch': []}, format='json')
        assert response.status_code == 400

    def test_sin_autenticacion_retorna_401(self, anonymous_client):
        response = anonymous_client.post('/api/v1/sync/upload/', {}, format='json')
        assert response.status_code == 401
```

**Serializer validations:**
- Patente: min 6, max 10, alfanumérico.
- VIN: exactamente 17 chars o null.
- `ley_caso_id` pertenece al tenant del usuario.
- `tipo_movimiento`, `tipo_vehiculo`, `formato_impresion` son valores válidos del enum.

**Permisos:**
- Operador no puede acceder a endpoints de reportes.
- Supervisor puede ver registros de todos los operadores de su tenant.
- Admin puede gestionar usuarios de su tenant.
- Ningún rol puede acceder a datos de otro tenant.

**Reportes:**
- Reporte diario filtra por `fecha_creacion_local` del día.
- Late-arriving records disparan regeneración de reportes.
- CSV contiene los campos esperados en el orden correcto.

## Qué NO Testear

- **Estilos CSS / layout visual**: no testear posiciones, colores, tamaños de fuente. Eso se verifica manualmente o con visual regression (fuera de scope MVP).
- **Internals de librerías**: no testear que Dexie guarde correctamente en IndexedDB, ni que `react-hook-form` valide correctamente. Testear el comportamiento de tu código que usa estas librerías.
- **Wording exacto de errores**: testear que un error se muestra, no que dice exactamente "La patente debe tener entre 6 y 8 caracteres alfanuméricos." Esto cambia frecuentemente.
- **Django admin**: funcionalidad CRUD del admin no requiere tests custom.

## Protocolo de Testing Manual / QA automatizado

El sistema cuenta con una extensa [suite de QA (`grabakar-docs/qa/`)](../qa/QA_MASTER_PLAN.md) y documentación de un Agente Tester QA automatizado vía emulador Android. Estas pruebas deben ejecutarse, manual o automáticamente, antes de cada release. Cada categoría es pass/fail.

### 1. Login / Sesión

- [ ] Login con credenciales válidas (requiere conexión).
- [ ] Login con credenciales inválidas muestra error.
- [ ] Login sin conexión muestra mensaje claro.
- [ ] Token offline permite acceso post-desconexión.
- [ ] Expiración de sesión (3h) redirige a login (si online).
- [ ] Expiración offline: token offline sigue funcionando hasta 72h.

### 2. Creación de Grabado

- [ ] Crear grabado con todos los campos completos.
- [ ] Validación: campos obligatorios vacíos muestran error.
- [ ] Validación: patente <6 chars muestra error.
- [ ] Validación: confirmación no coincide muestra error.
- [ ] Patente se normaliza a mayúsculas sin espacios.
- [ ] Post-guardado: patente y VIN quedan read-only.
- [ ] Botón cambia de "Guardar" a "Continuar a Grabado de Vidrios".
- [ ] UUID se genera y es visible en datos del registro.

### 3. Flujo Multi-Vidrio

- [ ] Auto muestra 6 vidrios. Moto muestra 1.
- [ ] Patente se muestra en formato grande en cada paso.
- [ ] Progreso indica vidrio actual / total.
- [ ] "Imprimir" incrementa contador visible.
- [ ] "Omitir" muestra confirmación y marca omitido.
- [ ] Último vidrio muestra "Finalizar" en vez de "Siguiente".
- [ ] Al finalizar, navega a Home.

### 4. Offline

- [ ] Crear 5+ grabados sin conexión (modo avión).
- [ ] Reconectar: sincronización automática ocurre.
- [ ] Registros aparecen en backend con timestamps correctos.
- [ ] Indicador de conectividad cambia visualmente.
- [ ] Cola de sync muestra registros pendientes.

### 5. Impresión

- [ ] (Fase 1) Botón "Imprimir" registra evento sin Bluetooth.
- [ ] (Fase 2+) Conexión Bluetooth, impresión horizontal y vertical.
- [ ] Formato auto vs moto se aplica correctamente.

### 6. Reportes

- [ ] Reporte diario CSV contiene registros del día.
- [ ] Registros tardíos (offline) aparecen en reporte correcto.
- [ ] Filtros por operador y tipo de movimiento funcionan.
- [ ] Supervisor puede descargar; operador no.

### 7. Exploit de Reimpresión

- [ ] Post-guardado, patente no es editable.
- [ ] Cada impresión incrementa `cantidad_impresiones`.
- [ ] No hay forma de cambiar patente y reimprimir en otro vidrio.

## CI — GitHub Actions

### Backend (`grabakar-backend/.github/workflows/ci.yml`) — Implementado

```yaml
name: Backend CI
on:
  pull_request:
    branches: [main]
    paths: ['grabakar-backend/**']
jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: grabakar-backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('grabakar-backend/requirements.txt') }}
          restore-keys: ${{ runner.os }}-pip-
      - run: pip install -r requirements.txt
      - run: ruff check .
      - env:
          DJANGO_SETTINGS_MODULE: config.settings.test
          SECRET_KEY: ci-test-secret-key
        run: pytest --cov=api --cov-fail-under=80 -q
```

Usa SQLite in-memory (`config.settings.test`) para velocidad en CI. No requiere servicios Postgres/Redis. 61 tests, ~80% cobertura en `api/`.

### Frontend (pendiente)

```yaml
# .github/workflows/frontend.yml — Crear cuando exista grabakar-frontend
name: Frontend CI
on:
  pull_request:
    paths: ['src/**', 'package.json']
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm run test -- --coverage
      - run: npm run build
```

## E2E (Playwright)

Ejecutar cuando backend y frontend estén integrados. Scenarios clave:

1. **Login → Crear grabado → Multi-vidrio → Verificar en Home**: flujo completo happy path.
2. **Offline flow**: crear grabados, simular desconexión (Service Worker intercept), reconectar, verificar sync.
3. **Duplicado**: crear grabado, crear otro con misma patente, verificar alerta y flag `es_duplicado`.
4. **Permisos**: login como operador, intentar acceder a reportes (debe fallar). Login como supervisor, acceder a reportes (debe funcionar).

Playwright corre en CI solo si hay cambios en frontend Y backend (o manualmente via workflow dispatch).
