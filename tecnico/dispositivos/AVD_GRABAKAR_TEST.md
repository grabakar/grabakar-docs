# Dispositivo de prueba — GrabaKar_Test AVD

> Documento generado el 2026-03-30 mediante inspección directa por ADB.
> ADB serial: `EP10022120100030`
> Conexión: USB (transport_id: 5)

---

## 1. Identificación del dispositivo

| Propiedad | Valor |
|-----------|-------|
| **Tipo** | Android Emulator (AVD) — **NO es hardware físico** |
| **Nombre AVD** | `GrabaKar_Test` (`ro.boot.qemu.avd_name`) |
| **Serial ADB** | `EP10022120100030` |
| **Serial emulador** | `EMULATOR36X4X10X0` |
| **Fabricante** | Google (`ro.product.manufacturer`) |
| **Modelo** | `sdk_gphone64_arm64` (`ro.product.model`) |
| **Producto** | `Q2` (ADB product name) |
| **Dispositivo interno** | `emu64a` (`ro.product.device`) |
| **Hardware** | `ranchu` (QEMU Goldfish/Ranchu — emulado) |
| **SoC** | `AOSP ranchu` (`ro.soc.manufacturer` / `ro.soc.model`) |
| **Placa base** | `goldfish_arm64` (`ro.product.board`) |

> **Nota importante:** El usuario mencionó "MediaTek" pero el dispositivo conectado es un AVD de Android Studio. El nombre de producto `Q2` reportado por ADB corresponde al nombre de la configuración del AVD, no a hardware real.

---

## 2. Sistema operativo y build

| Propiedad | Valor |
|-----------|-------|
| **Android version** | 14 |
| **API level (SDK)** | 34 |
| **Build ID** | `UE1A.230829.050` |
| **Build type** | `userdebug` |
| **Build tags** | `dev-keys` |
| **Build date** | Thu Jul 11 18:21:51 UTC 2024 |
| **Incremental** | `12077443` |
| **Security patch** | 2023-09-05 |
| **Fingerprint** | `google/sdk_gphone64_arm64/emu64a:14/UE1A.230829.050/12077443:userdebug/dev-keys` |
| **Build characteristics** | `emulator` |
| **ro.debuggable** | `1` (modo debug activo) |
| **ro.adb.secure** | `0` (ADB sin autenticación requerida) |
| **ro.test_harness** | `1` |
| **ro.monkey** | `1` |

---

## 3. CPU y arquitectura

| Propiedad | Valor |
|-----------|-------|
| **Arquitectura** | `arm64-v8a` |
| **ABI list** | `arm64-v8a` (solo 64-bit; sin soporte 32-bit) |
| **Núcleos** | 4 (processors 0–3) |
| **CPU implementer** | `0x61` (Apple VZ virtual CPU bajo QEMU) |
| **CPU architecture** | ARMv8 |
| **Kernel version** | 6.1 |
| **BogoMIPS por core** | 48.00 |

### Extensiones ARM disponibles
`fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm sb paca pacg dcpodp sve2 flagm2 frint svei8mm svebf16 bf16 afp smei16i64 smef64f64 smei8i32 smef16f32 smeb16f32 smef32f32`

Relevante para el proyecto: **AES y SHA acelerados por hardware** (confirma que `SubtleCrypto` AES-GCM en WebView tendrá soporte nativo).

---

## 4. Memoria y almacenamiento

### Memoria RAM

| Métrica | Valor |
|---------|-------|
| Total RAM | ~3.9 GB (4.016.816 kB) |
| RAM libre al momento de inspección | ~570 MB |
| RAM disponible (libre + reclaimable) | ~2.1 GB |
| Swap total | ~2.9 GB |
| Swap libre | ~2.5 GB |
| Heap máximo Dalvik | 576 MB (`dalvik.vm.heapsize`) |
| Heap growth limit | 192 MB (`dalvik.vm.heapgrowthlimit`) |

### Almacenamiento

| Partición | Tamaño | Usado | Libre | Uso% |
|-----------|--------|-------|-------|------|
| `/` (sistema) | 789 MB | 725 MB | 48 MB | 94% |
| `/vendor` | 92 MB | 91 MB | 0 | 100% |
| `/product` | 1.9 GB | 1.8 GB | 0 | 100% |
| `/system_ext` | 153 MB | 152 MB | 0 | 100% |
| `/data` (datos usuario) | **7.7 GB** | 1.2 GB | **6.3 GB** | 17% |
| `/storage/emulated` | 7.7 GB | 1.2 GB | 6.3 GB | 17% |

> `/data` tiene 6.3 GB libres — espacio suficiente para IndexedDB (Dexie) y el APK.

---

## 5. Pantalla

| Propiedad | Valor |
|-----------|-------|
| **Resolución física** | 1080 × 2400 px |
| **Densidad** | 420 dpi (xxhdpi) |
| **Skin configurado** | `1080x2400` (`ro.boot.qemu.skin`) |
| **Vsync** | 60 Hz |
| **OpenGL ES version** | 3.0 (`ro.opengles.version: 196608`) |
| **Renderer** | Skia GL (`debug.hwui.renderer: skiagl`) |
| **Vulkan** | Soportado (ranchu virtual) |
| **HDR** | No (`ro.surface_flinger.has_HDR_display: false`) |
| **Wide color** | No (`ro.surface_flinger.has_wide_color_display: false`) |

Relevante para impresión: `OffscreenCanvas` estará disponible en este WebView (Chromium/Skia GL), lo que significa que el **path bitmap de ESC/POS está habilitado** en este emulador.

---

## 6. Bluetooth

### Estado general

| Propiedad | Valor |
|-----------|-------|
| **Bluetooth encendido** | Sí (`bluetooth_on: 2`) |
| **Dirección BT** | `BB:BB:BB:00:00:02` (dirección sintética de emulador) |
| **Device class** | `[90, 2, 12]` |
| **Puerto virtual** | `/dev/vport8p2` (`vendor.qemu.vport.bluetooth`) |

### Perfiles habilitados

| Perfil | Estado | Relevancia para GrabaKar |
|--------|--------|--------------------------|
| GATT (BLE) | ✅ habilitado | Futuro (Phase 4+) |
| A2DP source | ✅ habilitado | No relevante (audio) |
| AVRCP target | ✅ habilitado | No relevante (audio) |
| HFP AG | ✅ habilitado | No relevante (manos libres) |
| HID device | ✅ habilitado | No relevante |
| HID host | ✅ habilitado | No relevante |
| MAP server | ✅ habilitado | No relevante |
| MCP server | ✅ habilitado | No relevante |
| OPP | ✅ habilitado | No relevante |
| PAN NAP | ✅ habilitado | No relevante |
| PAN PANU | ✅ habilitado | No relevante |
| PBAP server | ✅ habilitado | No relevante |
| **SPP / RFCOMM** | ❌ **NO listado** | **Crítico — es el que usa el plugin de impresión** |
| BAP broadcast assist | ❌ deshabilitado | — |
| BAP unicast client | ❌ deshabilitado | — |
| BAS client | ❌ deshabilitado | — |
| CCP server | ❌ deshabilitado | — |
| CSIP set coordinator | ❌ deshabilitado | — |
| HAP client | ❌ deshabilitado | — |
| VCP controller | ❌ deshabilitado | — |

> **Conclusión crítica para impresión:** El perfil **SPP (Serial Port Profile / RFCOMM)** — que es el que usa `@kduma-autoid/capacitor-bluetooth-printer` para comunicarse con la impresora térmica — **no está disponible en este emulador**. El Bluetooth del AVD es puramente virtual y no puede conectarse a hardware físico externo.

---

## 7. Servicios de impresión del sistema Android

| Componente | Estado |
|------------|--------|
| `cmd print list-print-services` | **Vacío** — ningún servicio registrado |
| `cmd print list-print-jobs` | **Vacío** |
| `com.android.printspooler` | Instalado (sistema) |
| `com.google.android.printservice.recommendation` | Instalado (sistema) |
| `com.android.bips` (Built-in Print Service) | Instalado (`/system/priv-app/BuiltInPrintService/`) |

No hay servicios de impresión activos ni configurados.

---

## 8. Paquetes instalados relevantes para el proyecto

### Bluetooth / Impresión
| Paquete | Origen |
|---------|--------|
| `com.android.bluetooth` | Sistema |
| `com.google.android.bluetooth` | Google |
| `com.android.bluetoothmidiservice` | Sistema |
| `com.android.printspooler` | Sistema |
| `com.google.android.printservice.recommendation` | Sistema |
| `com.android.bips` | Sistema (`BuiltInPrintService`) |

**No hay instaladas** apps de fabricantes de impresoras (Zebra, Brother, Epson, Star, Sunmi, Bixolon, Rongta, Xprinter, ESC/POS).

### Aplicaciones GrabaKar instaladas
Ninguna al momento de la inspección. El APK aún no ha sido instalado en este AVD.

---

## 9. Red y conectividad

| Propiedad | Valor |
|-----------|-------|
| **WiFi** | Virtio WiFi activo (`ro.boot.qemu.virtiowifi: 1`) |
| **DHCP WiFi** | Servicio corriendo |
| **Modo avión** | Activo (`persist.radio.airplane_mode_on: 1`) |
| **SIM** | No disponible (`gsm.sim.state: NOT_READY`) |
| **Red GSM** | Desconocida |

> **Modo avión activo**: puede afectar pruebas de sincronización online. Desactivar en Settings del AVD antes de probar el flujo de sync.

---

## 10. Timezone y locale

| Propiedad | Valor |
|-----------|-------|
| **Timezone sistema** | `America/Santiago` ✅ |
| **Timezone QEMU** | `America/Santiago` ✅ |
| **Locale** | `en-US` (la app debería funcionar igual, la UI es es-CL por código) |

Timezone correcto para el proyecto.

---

## 11. USB y modo ADB

| Propiedad | Valor |
|-----------|-------|
| **USB config** | `adb` |
| **USB state** | `adb` |
| **USB controller** | `dummy_udc.0` (virtual) |
| **ADB secure** | `0` (no requiere autorización) |
| **adbd** | Running |

---

## 12. Servicios del sistema en ejecución (relevantes)

| Servicio | Estado |
|----------|--------|
| `adbd` | running |
| `audioserver` | running |
| `bt_vhci_forwarder` | running (BT virtual) |
| `cameraserver` | running |
| `dhcpclient_wifi` | running |
| `goldfish-logcat` | running |
| `gpu` | running |
| `keystore2` | running |
| `surfaceflinger` | running |
| `wificond` | running |
| `wpa_supplicant` | running |
| `zygote` | running |

---

## 13. Implicaciones para el desarrollo y testing de GrabaKar

### Lo que SÍ funciona en este AVD

| Feature | Estado |
|---------|--------|
| Instalación del APK | ✅ Compatible (arm64-v8a, API 34) |
| Login y auth JWT | ✅ WiFi virtual disponible |
| Escritura en IndexedDB (Dexie) | ✅ `/data` tiene 6.3 GB libres |
| Cifrado AES-GCM (SubtleCrypto) | ✅ AES acelerado por hardware virtual |
| OffscreenCanvas (bitmap ESC/POS) | ✅ WebView Chromium/Skia GL disponible |
| Flujo de grabado (form → IndexedDB) | ✅ |
| Sincronización con backend | ✅ Requiere desactivar modo avión |
| Duplicate detection (30 días) | ✅ |
| Timezone America/Santiago | ✅ Preconfigurado |
| Camera (future OCR) | ✅ Cámara virtual disponible |
| `debug_printer` bypass | ✅ Simula impresión sin hardware |

### Lo que NO funciona en este AVD

| Feature | Razón |
|---------|-------|
| **Impresión Bluetooth SPP real** | SPP/RFCOMM no disponible en emulador; BT es virtual |
| **Conexión a impresora física** | El BT virtual no puede conectar con hardware externo |
| **`listPairedPrinters()`** | Retornará lista vacía o error |
| **`BluetoothPrinter.connectAndPrint()`** | Fallará con error de conexión |

### Workaround para testing de impresión

Usar el usuario especial `debug_printer` definido en `ImpresionPage.tsx:9`:

```typescript
const DEBUG_PRINTER_BYPASS_USERS = new Set(['debug_printer']);
```

Con este usuario, el botón "Imprimir" está habilitado sin impresora configurada y simula una impresión exitosa (incrementa el contador sin enviar datos al hardware).

---

## 14. Recomendaciones para pruebas en producción real

Para testing completo del flujo de impresión Bluetooth se requiere:

1. **Dispositivo Android físico** con Android 8+ (API 26+)
2. **Bluetooth Classic habilitado** con perfil SPP
3. **Impresora térmica compatible** con ESC/POS y SPP (ej: Rongta RP410, Xprinter XP-58, Goojprt PT-210)
4. **Emparejar la impresora** en Ajustes → Bluetooth del dispositivo físico antes de abrir la app
5. En la app: Menú → Impresora Bluetooth → seleccionar el dispositivo

---

*Generado por inspección ADB directa — 2026-03-30*
