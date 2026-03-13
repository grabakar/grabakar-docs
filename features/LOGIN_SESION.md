# LOGIN_SESION — Login y Gestión de Sesión

## Descripción

Autenticación de usuarios mediante credenciales (username + password) con JWT. Retorna tokens de acceso, refresco y offline, junto con la configuración del tenant (branding, leyes activas, configuración de vidrios). El token offline permite operar la app sin conexión a internet.

## Reglas de Negocio

- Login **requiere conexión a internet**. Sin conexión, mostrar: _"Se requiere conexión a internet para iniciar sesión."_
- Endpoint: `POST /api/v1/auth/login/`
- Request body:

```json
{ "username": "operador1", "password": "s3cret" }
```

- Response exitosa (200):

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "offline_token": "eyJ...",
  "usuario": {
    "id": 42,
    "username": "operador1",
    "nombre_completo": "Juan Pérez",
    "rol": "operador",
    "tenant_id": 1
  },
  "tenant_config": {
    "nombre": "VidrioPro Chile",
    "logo_url": "https://cdn.ejemplo.com/vidriopro/logo.png",
    "color_primario": "#1A56DB",
    "color_secundario": "#F97316"
  },
  "leyes_activas": [
    { "id": 1, "codigo": "LEY_19.866", "nombre": "Ley 19.866 Grabado de Patente", "vigente": true }
  ],
  "vidrios_por_tipo_vehiculo": {
    "auto": ["Parabrisas", "Luneta", "Delantera Izquierda", "Delantera Derecha", "Trasera Izquierda", "Trasera Derecha"],
    "moto": ["Parabrisas"]
  }
}
```

- **Duración de tokens**:
  | Token | Duración | Propósito |
  |-------|----------|-----------|
  | `access_token` | 3 horas | Autenticación de requests API |
  | `refresh_token` | 7 días | Renovar `access_token` sin re-login |
  | `offline_token` | 72 horas | Operar app sin internet |

- **Auto-refresh**: Si hay conexión, renovar `access_token` automáticamente cuando le falten < 5 minutos para expirar. Endpoint: `POST /api/v1/auth/refresh/` con `refresh_token`.
- **Expiración offline**: Si `offline_token` expira y no hay conexión, redirigir a pantalla de login con mensaje: _"Tu sesión offline ha expirado. Conéctate a internet para iniciar sesión nuevamente."_
- **Logout**: Eliminar todos los tokens del almacenamiento local. **NO eliminar registros no sincronizados** de IndexedDB. Endpoint: `POST /api/v1/auth/logout/` (invalidar refresh token en servidor si hay conexión).
- **Roles**: `operador`, `supervisor`, `admin`. El rol determina permisos en la app.
- **Intentos fallidos**: Después de 5 intentos fallidos consecutivos, bloquear login por 5 minutos. Mensaje: _"Demasiados intentos fallidos. Intenta nuevamente en X minutos."_

## Flujo de Usuario

1. Usuario abre la app → Verificar si existe `offline_token` válido en almacenamiento local.
2. **Si existe token válido**: Cargar config cacheada de IndexedDB → Navegar a Home.
3. **Si no existe o expiró**: Mostrar pantalla de login.
4. Usuario ingresa username y password → Tap "Iniciar Sesión".
5. **Sin conexión**: Mostrar error inline _"Se requiere conexión a internet para iniciar sesión."_
6. **Con conexión**: Enviar `POST /api/v1/auth/login/`.
7. **Credenciales inválidas (401)**: Mostrar _"Usuario o contraseña incorrectos."_
8. **Login exitoso**: Guardar tokens encriptados en localStorage, guardar `tenant_config`, `leyes_activas` y `vidrios_por_tipo_vehiculo` en IndexedDB → Navegar a Home.

## Frontend

### Almacenamiento de Tokens

- Guardar `access_token`, `refresh_token`, `offline_token` en localStorage encriptados con `crypto.subtle` (AES-GCM). La clave de encriptación se deriva del device fingerprint.
- Guardar `tenant_config`, `leyes_activas`, `vidrios_por_tipo_vehiculo` en tabla `configuracion` de IndexedDB (Dexie.js).

### AuthProvider (React Context)

```typescript
interface AuthState {
  usuario: Usuario | null;
  isAuthenticated: boolean;
  isOffline: boolean;
  rol: 'operador' | 'supervisor' | 'admin';
}

// Hook expuesto
function useAuth(): AuthState & {
  login: (username: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
};
```

### Pantalla de Login

- Campos: username (text), password (password con toggle visibilidad).
- Botón "Iniciar Sesión" deshabilitado hasta que ambos campos tengan valor.
- Indicador de estado de conexión visible (badge verde/rojo).
- Logo del tenant si hay config cacheada, sino logo default GrabaKar.

### Auto-refresh

- `setInterval` que verifica expiración de `access_token` cada 60 segundos.
- Si faltan < 5 minutos y hay conexión → llamar refresh endpoint.
- Si refresh falla con 401 → verificar `offline_token`. Si válido, continuar offline. Si no, redirigir a login.

## Backend

### Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/api/v1/auth/login/` | Login, retorna tokens + config |
| POST | `/api/v1/auth/refresh/` | Renovar access token |
| POST | `/api/v1/auth/logout/` | Invalidar refresh token |

### Modelos

```python
# Usuario extiende AbstractUser de Django
class Usuario(AbstractUser):
    tenant = models.ForeignKey('Tenant', on_delete=models.CASCADE)
    rol = models.CharField(max_length=20, choices=[('operador','Operador'), ('supervisor','Supervisor'), ('admin','Admin')])
```

### Generación de Tokens

- Usar `djangorestframework-simplejwt` con claims personalizados: `user_id`, `tenant_id`, `rol`.
- `offline_token` es un JWT firmado con claim `tipo: "offline"` y expiración 72h.
- El `offline_token` NO se almacena en servidor; solo se valida su firma y expiración.

### Login Response

- Serializar `tenant_config` desde modelo `Tenant` del usuario.
- Filtrar `leyes_activas` por `Ley.objects.filter(vigente=True, tenant=user.tenant)`.
- Retornar `vidrios_por_tipo_vehiculo` desde `ConfiguracionVidrio.objects.filter(tenant=user.tenant)`.

## Validaciones

| Campo | Regla | Mensaje de error |
|-------|-------|-----------------|
| username | Requerido, no vacío | _"El nombre de usuario es obligatorio."_ |
| password | Requerido, min 6 caracteres | _"La contraseña debe tener al menos 6 caracteres."_ |
| Conexión | Disponible para login | _"Se requiere conexión a internet para iniciar sesión."_ |
| Credenciales | Válidas en backend | _"Usuario o contraseña incorrectos."_ |
| Cuenta bloqueada | < 5 intentos fallidos | _"Demasiados intentos fallidos. Intenta nuevamente en {minutos} minutos."_ |

## Criterio de Aceptación

1. Login exitoso con credenciales válidas almacena 3 tokens y config del tenant en dispositivo.
2. Login sin conexión muestra mensaje de error y no permite continuar.
3. Credenciales inválidas muestran error sin revelar si fue usuario o contraseña.
4. Con `offline_token` válido, app carga directamente al Home sin login.
5. `access_token` se renueva automáticamente antes de expirar cuando hay conexión.
6. Al expirar `offline_token` sin conexión, usuario es redirigido a login con mensaje explicativo.
7. Logout elimina tokens pero preserva registros no sincronizados en IndexedDB.
8. Config del tenant (branding, leyes, vidrios) se cachea y está disponible offline.

## Casos Edge

- **Cambio de password en otro dispositivo**: El `refresh_token` del dispositivo anterior falla → forzar login. El `offline_token` sigue válido hasta expirar (72h max).
- **Reloj del dispositivo desincronizado**: Validar expiración de `offline_token` comparando con `Date.now()`. Si el reloj está muy desfasado, el token podría parecer válido/inválido incorrectamente. No hay solución completa sin conexión.
- **Tenant desactivado**: Backend retorna 403 en login con mensaje _"Tu cuenta ha sido desactivada. Contacta al administrador."_. Si ya estaba offline, el usuario sigue operando hasta que el `offline_token` expire.
- **Múltiples pestañas/instancias**: Usar `BroadcastChannel` para sincronizar logout entre pestañas.
- **Almacenamiento lleno**: Si `localStorage` o IndexedDB falla al guardar, mostrar _"Error al guardar sesión. Libera espacio en el dispositivo."_
