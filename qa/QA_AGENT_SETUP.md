# QA Agent Setup — Android Emulator & Tooling

Step-by-step guide to set up the environment for the QA Tester Agent to run on macOS.

---

## 1. Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Java JDK | 17+ | `brew install openjdk@17` |
| Android SDK Command-line Tools | Latest | Manual download |
| Node.js | 20+ | `brew install node` |
| `gh` CLI | Latest | `brew install gh` |
| Python | 3.12+ | `brew install python@3.12` |

---

## 2. Java Setup

```bash
# Install JDK 17
brew install openjdk@17

# Set JAVA_HOME
export JAVA_HOME=$(/usr/libexec/java_home -v 17)

# Add to ~/.zshrc
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 17)' >> ~/.zshrc
```

---

## 3. Android SDK Setup

```bash
# Set ANDROID_HOME
export ANDROID_HOME=$HOME/Library/Android/sdk

# Add to ~/.zshrc
cat >> ~/.zshrc << 'EOF'
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/emulator
EOF

source ~/.zshrc

# Install SDK components via sdkmanager
sdkmanager --install \
  "platform-tools" \
  "platforms;android-34" \
  "system-images;android-34;google_apis;arm64-v8a" \
  "emulator" \
  "build-tools;34.0.0"

# Accept licenses
sdkmanager --licenses
```

---

## 4. Create Android Virtual Device (AVD)

```bash
# Create AVD named "GrabaKar_Test"
avdmanager create avd \
  --name "GrabaKar_Test" \
  --package "system-images;android-34;google_apis;arm64-v8a" \
  --device "pixel_6" \
  --force

# Verify AVD
avdmanager list avd
```

### Recommended AVD Config

Edit `~/.android/avd/GrabaKar_Test.avd/config.ini`:

```ini
hw.lcd.density=420
hw.lcd.height=2400
hw.lcd.width=1080
hw.ramSize=4096
disk.dataPartition.size=8G
hw.keyboard=yes
hw.gpu.enabled=yes
hw.gpu.mode=auto
showDeviceFrame=no
```

---

## 5. Boot Emulator

```bash
# Start with display (interactive testing)
emulator -avd GrabaKar_Test -no-audio -no-boot-anim

# Start headless (CI/automated testing)
emulator -avd GrabaKar_Test -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect &

# Wait for boot
adb wait-for-device
adb shell getprop sys.boot_completed    # returns "1" when ready
```

---

## 6. Build & Install APK

### Option A: Build from source

```bash
cd /Users/franciscocollarte/Documents/grabado-patente-app/grabakar-frontend

# Install dependencies
npm install

# Build web assets
npm run build

# Sync with Capacitor
npx cap sync android

# Build debug APK
cd android
./gradlew assembleDebug

# APK location
ls -la app/build/outputs/apk/debug/app-debug.apk
```

### Option B: Use existing APK

```bash
# If APK already built, install directly
adb install -r /path/to/app-debug.apk
```

### Verify Installation

```bash
# Check app is installed
adb shell pm list packages | grep grabakar

# Launch app
adb shell am start -n com.grabakar.app/.MainActivity
```

---

## 7. Backend Setup (Local)

```bash
cd /Users/franciscocollarte/Documents/grabado-patente-app/grabakar-backend

# Start all services
docker compose up -d

# Run migrations
docker compose exec django python manage.py migrate

# Load seed data
docker compose exec django python manage.py loaddata api/fixtures/initial.json

# Create test users (if not in fixtures)
docker compose exec django python manage.py shell -c "
from api.models import Tenant, Usuario
t = Tenant.objects.first()
for role, name, pwd in [('operador', 'operador1', 'password123'),
                        ('supervisor', 'supervisor1', 'password123'),
                        ('admin', 'admin1', 'password123')]:
    u, created = Usuario.objects.get_or_create(username=name, defaults={
        'tenant': t, 'rol': role, 'first_name': name.title()
    })
    if created:
        u.set_password(pwd)
        u.save()
        print(f'Created {role}: {name}')
"

# Verify backend
curl http://localhost:8000/api/v1/health/
```

### Emulator → Local Backend Networking

The emulator accesses the host machine via `10.0.2.2`:

```bash
# From inside emulator, host:8000 is reachable at:
# http://10.0.2.2:8000

# The frontend .env should point to this:
# VITE_API_URL=http://10.0.2.2:8000/api/v1
```

---

## 8. GitHub CLI Setup

```bash
# Install
brew install gh

# Authenticate
gh auth login

# Verify
gh auth status

# Setup repo context
cd /Users/franciscocollarte/Documents/grabado-patente-app/grabakar-frontend
gh repo set-default

# Create QA labels (one-time)
gh label create "qa-auto" --color "0E8A16" --description "Automated QA finding"
gh label create "severity-S1" --color "D73A4A" --description "Critical - app crash, data loss"
gh label create "severity-S2" --color "FF9800" --description "Major - feature broken"
gh label create "severity-S3" --color "FFC107" --description "Minor - cosmetic, UX"
gh label create "severity-S4" --color "4CAF50" --description "Trivial - enhancement"
gh label create "blocker" --color "B60205" --description "Blocks release"
```

---

## 9. Verification Checklist

Run these checks to verify the full QA environment is ready:

```bash
echo "=== QA Environment Verification ==="

# Java
echo -n "Java: "; java -version 2>&1 | head -1

# Android SDK
echo -n "ADB: "; adb version | head -1
echo -n "Emulator: "; emulator -version | head -1

# AVD
echo -n "AVD: "; avdmanager list avd 2>/dev/null | grep "Name:"

# gh CLI
echo -n "gh CLI: "; gh --version | head -1
echo -n "gh auth: "; gh auth status 2>&1 | grep "Logged in"

# Backend
echo -n "Backend: "; curl -s http://localhost:8000/api/v1/health/ | head -1

# Node
echo -n "Node: "; node --version

echo "=== Verification Complete ==="
```

---

## 10. Quick Start (TL;DR)

```bash
# 1. Boot emulator
emulator -avd GrabaKar_Test -no-audio &
adb wait-for-device

# 2. Start backend
cd /path/to/grabakar-backend && docker compose up -d

# 3. Build and install APK
cd /path/to/grabakar-frontend
npm run build && npx cap sync android
cd android && ./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk

# 4. Launch app
adb shell am start -n com.grabakar.app/.MainActivity

# 5. Ready for testing!
```
