# WHITE_LABELING — Multi-Tenant Branding / White-Labeling

## Descripción

La app soporta múltiples clientes (tenants), cada uno con su marca visual propia: nombre, logo, colores. La configuración de branding se obtiene en el login, se cachea en IndexedDB, y se aplica dinámicamente vía CSS variables. No existe ninguna referencia hardcodeada a "GrabaKar" en el código de componentes.

## Reglas de Negocio

- Cada tenant tiene: `nombre`, `logo_url`, `color_primario`, `color_secundario`, `configuracion_json` (overrides adicionales).
- La config se retorna en la respuesta de login (`tenant_config`) y se cachea en IndexedDB bajo la clave `tenant_branding`.
- CSS variables se aplican en `document.documentElement.style`:
  - `--color-primary` → `color_primario`
  - `--color-secondary` → `color_secundario`
  - `--logo-url` → `url(logo_url)`
  - `--app-name` → `nombre` (solo para uso en JS, no CSS content)
- **Valores default** (cuando no hay config cacheada):
  | Variable | Valor default |
  |----------|---------------|
  | `--color-primary` | `#1A56DB` |
  | `--color-secondary` | `#F97316` |
  | `--logo-url` | `/assets/grabakar-logo.svg` |
  | `--app-name` | Valor de env `VITE_APP_NAME` o `"GrabaKar"` |
- **Zero hardcoded brand names**: Ningún componente React referencia "GrabaKar" directamente. El nombre se lee de `useAppConfig().appName`.
- El logo se carga dinámicamente desde la URL en la config. Si falla, se muestra el logo default.
- La config se puede refrescar manualmente vía `GET /api/v1/config/` cuando hay conexión.

## Flujo de Configuración

1. **Login exitoso** → Backend retorna `tenant_config` → Frontend guarda en IndexedDB.
2. **App init** → `ThemeProvider` lee config desde IndexedDB → Aplica CSS variables.
3. **Sin config cacheada** (primer uso, datos borrados) → Aplicar valores default.
4. **Refresh manual** → `GET /api/v1/config/` → Actualizar IndexedDB → Re-aplicar CSS variables.

## Frontend

### Almacenamiento en IndexedDB

```typescript
interface TenantBranding {
  clave: 'tenant_branding';
  valor: {
    nombre: string;
    logo_url: string;
    color_primario: string;
    color_secundario: string;
    configuracion_json: Record<string, unknown>;
    fecha_cache: string; // ISO 8601
  };
}

// Guardar en login
await db.configuracion.put({
  clave: 'tenant_branding',
  valor: { ...tenantConfig, fecha_cache: new Date().toISOString() }
});
```

### ThemeProvider

```typescript
const DEFAULTS: TenantBranding['valor'] = {
  nombre: import.meta.env.VITE_APP_NAME || 'GrabaKar',
  logo_url: '/assets/grabakar-logo.svg',
  color_primario: '#1A56DB',
  color_secundario: '#F97316',
  configuracion_json: {},
  fecha_cache: '',
};

interface AppConfig {
  appName: string;
  logoUrl: string;
  colorPrimario: string;
  colorSecundario: string;
  config: Record<string, unknown>;
}

const AppConfigContext = createContext<AppConfig>(/* from DEFAULTS */);

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [config, setConfig] = useState<AppConfig>(mapDefaults(DEFAULTS));

  useEffect(() => {
    db.configuracion.get('tenant_branding').then(cached => {
      const branding = cached?.valor ?? DEFAULTS;
      const root = document.documentElement.style;
      root.setProperty('--color-primary', branding.color_primario);
      root.setProperty('--color-secondary', branding.color_secundario);
      root.setProperty('--logo-url', `url(${branding.logo_url})`);
      setConfig({
        appName: branding.nombre,
        logoUrl: branding.logo_url,
        colorPrimario: branding.color_primario,
        colorSecundario: branding.color_secundario,
        config: branding.configuracion_json,
      });
    });
  }, []);

  return (
    <AppConfigContext.Provider value={config}>
      {children}
    </AppConfigContext.Provider>
  );
}

function useAppConfig(): AppConfig {
  return useContext(AppConfigContext);
}
```

### Uso en Componentes

```typescript
// Header
function AppHeader() {
  const { appName, logoUrl } = useAppConfig();
  return (
    <header>
      <img src={logoUrl} alt={appName} onError={e => e.currentTarget.src = '/assets/grabakar-logo.svg'} />
      <h1>{appName}</h1>
    </header>
  );
}
```

### CSS Variables en Tailwind / CSS

```css
:root {
  --color-primary: #1A56DB;
  --color-secondary: #F97316;
}

.btn-primary {
  background-color: var(--color-primary);
}
.btn-secondary {
  background-color: var(--color-secondary);
}
.text-brand {
  color: var(--color-primary);
}
```

### Refresh de Config

```typescript
async function refreshConfig(): Promise<void> {
  const response = await fetch('/api/v1/config/', {
    headers: { 'Authorization': `Bearer ${getAccessToken()}` }
  });
  if (!response.ok) return;
  const data = await response.json();
  await db.configuracion.put({
    clave: 'tenant_branding',
    valor: { ...data, fecha_cache: new Date().toISOString() }
  });
  // Re-aplicar CSS variables
  const root = document.documentElement.style;
  root.setProperty('--color-primary', data.color_primario);
  root.setProperty('--color-secondary', data.color_secundario);
  root.setProperty('--logo-url', `url(${data.logo_url})`);
}
```

## Backend

### Modelo Tenant

```python
class Tenant(models.Model):
    nombre = models.CharField(max_length=100)
    logo_url = models.URLField(blank=True)
    color_primario = models.CharField(max_length=7, default='#1A56DB')  # hex
    color_secundario = models.CharField(max_length=7, default='#F97316')
    configuracion_json = models.JSONField(default=dict, blank=True)
    activo = models.BooleanField(default=True)

    def __str__(self):
        return self.nombre
```

### Endpoint de Config

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/api/v1/config/` | Retorna config del tenant del usuario autenticado |

```python
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def tenant_config(request):
    tenant = request.user.tenant
    return Response({
        'nombre': tenant.nombre,
        'logo_url': tenant.logo_url,
        'color_primario': tenant.color_primario,
        'color_secundario': tenant.color_secundario,
        'configuracion_json': tenant.configuracion_json,
    })
```

### Scope por Tenant

Todas las queries de la API deben filtrar por el tenant del usuario autenticado:

```python
class TenantQuerySetMixin:
    def get_queryset(self):
        return super().get_queryset().filter(tenant=self.request.user.tenant)
```

Aplicar como mixin en todos los ViewSets y vistas basadas en clase.

## Validaciones

| Campo | Regla | Mensaje |
|-------|-------|---------|
| color_primario | Hex válido (#RRGGBB) | _"El color primario debe ser un código hexadecimal válido."_ |
| color_secundario | Hex válido (#RRGGBB) | _"El color secundario debe ser un código hexadecimal válido."_ |
| logo_url | URL válida o vacía | _"La URL del logo no es válida."_ |
| nombre | Requerido, max 100 chars | _"El nombre del tenant es obligatorio."_ |

## Criterio de Aceptación

1. Tras login, la app muestra el nombre y logo del tenant, no "GrabaKar".
2. Los colores primario y secundario del tenant se aplican en botones, headers, y acentos.
3. Sin config cacheada, la app muestra branding default (GrabaKar).
4. Si el logo del tenant falla al cargar, se muestra el logo default.
5. `useAppConfig()` retorna la config del tenant en cualquier componente.
6. `GET /api/v1/config/` retorna la config actualizada del tenant.
7. Zero strings "GrabaKar" hardcodeadas en código de componentes React (verificable con grep).
8. La config persiste en IndexedDB y se aplica en recargas de la app sin internet.

## Casos Edge

- **Logo URL caducada o CDN caído**: `onError` en `<img>` hace fallback a logo default. No se muestra imagen rota.
- **Colores con bajo contraste**: El backend no valida contraste. Si un tenant configura colores ilegibles, es responsabilidad del admin. Posible mejora futura: validar ratio WCAG AA.
- **Config JSON malformada**: `configuracion_json` se parsea con fallback a `{}`. Si el JSON es inválido en IndexedDB, ignorar y usar defaults.
- **Cambio de tenant del usuario**: Si un admin reasigna un usuario a otro tenant, el próximo login sobrescribe la config cacheada. Mientras esté offline con la config vieja, el usuario ve la marca anterior.
- **Múltiples tenants en el mismo dispositivo**: Si diferentes usuarios de distintos tenants usan el mismo dispositivo, la config se sobrescribe en cada login. No hay conflicto porque se cachea bajo la misma clave.
- **Caracteres especiales en nombre de tenant**: Escapar para HTML (`&`, `<`, `>`). React maneja esto por defecto con JSX.
