# Seguridad

## Autenticación

### JWT

- **Access token**: duración 3 horas. Se envía en header `Authorization: Bearer <token>`.
- **Refresh token**: duración 7 días. Solo se usa para obtener un nuevo access token via `POST /api/v1/auth/refresh/`.
- **Offline token**: duración configurable (default 72h). Permite operar el dispositivo sin conexión después de que el access token expire.

```python
# config/settings/base.py
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=int(os.getenv('JWT_ACCESS_TOKEN_LIFETIME_HOURS', 3))),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=int(os.getenv('JWT_REFRESH_TOKEN_LIFETIME_DAYS', 7))),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

### Offline Token

El offline token se genera en el servidor al login exitoso y se almacena cifrado en el dispositivo.

**Claims del offline token:**
- `user_id`, `tenant_id`, `rol`: identificación básica.
- `exp`: timestamp de expiración (72h desde emisión).
- `scope: "offline"`: limita operaciones. No puede hacer requests al servidor — solo permite operar la app localmente.

**Almacenamiento en dispositivo:**
- El offline token se cifra con AES-256 usando una clave derivada del dispositivo antes de guardarlo en IndexedDB.
- Al reconectarse, se fuerza re-autenticación completa (access + refresh token nuevos). El offline token se invalida.

```typescript
async function guardarOfflineToken(token: string): Promise<void> {
  const key = await deriveDeviceKey();
  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: crypto.getRandomValues(new Uint8Array(12)) },
    key, new TextEncoder().encode(token)
  );
  await db.configuracion.put({ clave: 'offline_token', valor: encrypted });
}
```

### Flujo de auth según conectividad

```
ONLINE + token válido   → Operación normal
ONLINE + token expirado → Refresh automático con refresh_token
ONLINE + refresh expira → Redirigir a login
OFFLINE + offline token → Operación local (sin sync)
OFFLINE + offline expir → Bloquear app, mostrar "Conecta para iniciar sesión"
```

## Aislamiento Multi-Tenant

Toda query a la base de datos se filtra por `tenant_id` del usuario autenticado. No hay excepciones.

```python
class TenantQuerySetMixin:
    def get_queryset(self):
        qs = super().get_queryset()
        if hasattr(self.request, 'user') and self.request.user.is_authenticated:
            return qs.filter(tenant=self.request.user.tenant)
        return qs.none()
```

Aplicar este mixin a **todas** las views que acceden a datos de tenant (Grabado, ImpresionVidrio, Usuario, LeyCaso, Reportes).

**Validaciones adicionales:**
- En sync upload: verificar que `ley_caso_id` y `usuario_responsable_id` pertenecen al tenant del usuario autenticado.
- En reportes: el CSV solo contiene registros del tenant.
- Django admin: filtrar por tenant del admin user (si se expone admin a clientes).

**Test de aislamiento:**

```python
def test_usuario_no_ve_grabados_de_otro_tenant(api_client, otro_tenant):
    Grabado.objects.create(tenant=otro_tenant, ...)
    response = api_client.get('/api/v1/grabados/')
    assert len(response.data['results']) == 0
```

## Seguridad de API

### Rate Limiting

```python
# api/views/auth.py
from django_ratelimit.decorators import ratelimit

class LoginView(APIView):
    @ratelimit(key='ip', rate='10/m', method='POST', block=True)
    def post(self, request):
        ...
```

| Endpoint | Rate limit | Razón |
|---|---|---|
| `/auth/login/` | 10/min por IP | Prevenir brute force |
| `/auth/refresh/` | 30/min por IP | Limitar token rotation abuse |
| `/sync/upload/` | 60/min por user | Prevenir flood de sync |
| Otros endpoints | 120/min por user | Rate limit general |

### CORS

```python
# config/settings/base.py
CORS_ALLOWED_ORIGINS = os.getenv('CORS_ALLOWED_ORIGINS', '').split(',')
CORS_ALLOW_CREDENTIALS = True
```

Solo el dominio del frontend está permitido. En desarrollo: `http://localhost:5173`. En producción: `https://app.grabakar.cl` (o dominio del tenant).

### HTTPS

- Obligatorio en staging y producción. Django `SECURE_SSL_REDIRECT = True` en settings de producción.
- En desarrollo local, HTTP es aceptable.
- HSTS habilitado en producción:

```python
# config/settings/production.py
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## Validación de Datos

- **Todo input pasa por DRF serializers.** No hay parsing manual de `request.data`.
- **Sin raw SQL.** Todas las queries via ORM de Django. Si se necesita SQL avanzado, usar `RawSQL` con parámetros (nunca string interpolation).
- **Parameterized queries** son el default del ORM. Ejemplo de lo que NO hacer:

```python
# MAL - SQL injection vulnerable
Grabado.objects.raw(f"SELECT * FROM api_grabado WHERE patente = '{patente}'")

# BIEN - Parameterizado
Grabado.objects.raw("SELECT * FROM api_grabado WHERE patente = %s", [patente])

# MEJOR - ORM
Grabado.objects.filter(patente=patente)
```

## Confianza del Dispositivo

`device_id` es auto-reportado por el dispositivo. No hay verificación criptográfica en el MVP.

**Limitación conocida:** un actor malicioso puede falsificar `device_id` y enviar registros que parecen venir de otro dispositivo. Esto es aceptable para MVP porque:
1. Los usuarios son operadores de empresas clientes (B2B), no público general.
2. Cada registro lleva `usuario_responsable` autenticado — la trazabilidad real es por usuario, no por dispositivo.
3. `device_id` se usa principalmente para detección de duplicados, no para seguridad.

**Mitigación futura (post-MVP):** device attestation via Play Integrity API (Android) o similar.

## Gestión de Secrets

### En desarrollo

- Archivo `.env` local (nunca commiteado).
- `.env.example` con placeholders en el repositorio.
- `.gitignore` incluye `.env`, `*.pem`, `*.key`.

### En producción (GCP)

- **Secret Manager** para todas las credenciales (`DJANGO_SECRET_KEY`, `DB_PASSWORD`, credenciales SMTP, etc.).
- Cloud Run monta secrets como env vars en runtime.
- Sin archivos de credenciales en el filesystem del container.

### Checklist de secrets

| Secret | Local | Staging/Prod | Rotar cada |
|---|---|---|---|
| `DJANGO_SECRET_KEY` | `.env` | Secret Manager | 90 días |
| `DB_PASSWORD` | `.env` | Secret Manager | 90 días |
| `JWT_SIGNING_KEY` | Django SECRET_KEY | Secret Manager | 90 días |
| `EMAIL_HOST_PASSWORD` | `.env` | Secret Manager | 90 días |
| `GCP_SA_KEY` (CI) | N/A | GitHub Secrets | 180 días |

## OWASP Top 10 — Consideraciones

| Vulnerabilidad | Mitigación |
|---|---|
| **XSS** | React escapa output por defecto. No usar `dangerouslySetInnerHTML`. CSP headers en producción. |
| **CSRF** | No aplica: autenticación por JWT en header, no cookies de sesión. Django CSRF middleware se puede desactivar para API endpoints que usan JWT. |
| **SQL Injection** | ORM de Django con queries parametrizadas. Sin raw SQL en código de negocio. |
| **Broken Authentication** | JWT con expiración corta (3h). Refresh token rotation. Rate limit en login. |
| **Broken Access Control** | Tenant filtering en todas las queries. Permission classes por endpoint. Tests de aislamiento. |
| **Security Misconfiguration** | DEBUG=False en producción. ALLOWED_HOSTS configurado. Secrets en Secret Manager. |
| **Sensitive Data Exposure** | HTTPS obligatorio. Offline token cifrado en dispositivo. Sin datos sensibles en logs. |
| **Insecure Deserialization** | DRF serializers validan estructura y tipos. No se acepta pickle ni formatos binarios. |
