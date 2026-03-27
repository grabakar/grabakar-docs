# APK Variants Pipeline (Local + GCP)

Runbook para generar APKs Android de `grabakar-frontend` apuntando a dos backends distintos:

- **Local**: `http://10.0.2.2:8000`
- **GCP Staging**: `https://grabakar-backend-1089044937741.us-central1.run.app`

---

## Objetivo

Evitar errores de entorno (instalar un APK local cuando se quería staging, o viceversa) y estandarizar:

1. Build local reproducible.
2. Build en GitHub Actions con artefactos descargables.
3. Verificación explícita de URL backend embebida.

---

## Requisitos Técnicos

### Java

Capacitor/Gradle en este repo requiere **Java 21** para compilar Android.

- Error típico con Java 17:
  - `invalid source release: 21`

En macOS (Homebrew), usar:

```bash
export JAVA_HOME="/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
java -version
```

### Node

- Node 20.x (alineado con workflows de CI/CD).

---

## Scripts locales (grabakar-frontend)

`package.json` incluye:

- `build:android:local`
- `build:android:gcp`
- `build:android:variants`

### 1) APK local

```bash
cd grabakar-frontend
npm run build:android:local
```

### 2) APK staging GCP

```bash
cd grabakar-frontend
npm run build:android:gcp
```

### 3) Ambos APKs (pipeline local)

```bash
cd grabakar-frontend
npm run build:android:variants
```

Este comando ejecuta `scripts/build-apk-variants.sh` y deja:

- `android/app/build/outputs/apk/debug/app-debug-local.apk`
- `android/app/build/outputs/apk/debug/app-debug-gcp.apk`

---

## Pipeline GitHub Actions

Workflow: `.github/workflows/build-apks.yml`

- Trigger: `workflow_dispatch`
- Matrix:
  - `variant=local` (`VITE_API_URL=http://10.0.2.2:8000`)
  - `variant=gcp` (`VITE_API_URL=https://grabakar-backend-1089044937741.us-central1.run.app`)
- Runtime:
  - `actions/setup-node@v4` (Node 20)
  - `actions/setup-java@v4` (Temurin 21)
- Build:
  - `npm ci`
  - `npm run build:android` (inyectando `VITE_API_URL` por variante)
- Artifacts:
  - `apk-local` -> `app-debug-local.apk`
  - `apk-gcp` -> `app-debug-gcp.apk`

---

## Verificación post-build (obligatoria)

No basta con que Gradle termine OK; hay que validar que el bundle embebió la URL correcta.

Ejemplo para variante GCP:

```bash
python3 - <<'PY'
from pathlib import Path
p = Path("grabakar-frontend/dist/assets")
js = next(p.glob("index-*.js"))
s = js.read_text(errors="ignore")
print("has_gcp_url=", "grabakar-backend-1089044937741.us-central1.run.app" in s)
print("has_local_10_0_2_2=", "10.0.2.2:8000" in s)
PY
```

Esperado para build GCP:

- `has_gcp_url=True`
- `has_local_10_0_2_2=False`

---

## Paths importantes

- APK debug base (último build):
  `grabakar-frontend/android/app/build/outputs/apk/debug/app-debug.apk`

- APK local nombrado:
  `grabakar-frontend/android/app/build/outputs/apk/debug/app-debug-local.apk`

- APK GCP nombrado:
  `grabakar-frontend/android/app/build/outputs/apk/debug/app-debug-gcp.apk`

---

## Troubleshooting

### "Unable to locate a Java Runtime"

El sistema no tiene `JAVA_HOME` válido en shell actual. Exportar Java 21 y reintentar.

### "invalid source release: 21"

Se está compilando con Java < 21. Forzar Java 21.

### APK compila pero apunta al backend incorrecto

La variable `VITE_API_URL` usada al build no era la esperada. Rebuild + verificación de bundle.

### El APK no refleja cambios recientes de frontend

Asegurar que el flujo complete:

1. `npm run build`
2. `npx cap sync android`
3. `./gradlew assembleDebug`

(El script `build:android` ya encapsula esas tres etapas).
