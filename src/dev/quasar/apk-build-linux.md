# APK Build Guide — Linux

This guide explains how to set up a **Linux** environment and build a
signed (release) APK for the **Mercado Local** app (Quasar + Capacitor).

## Requirements

| Tool | Version | Source |
|---|---|---|
| Android CLI (agent) | latest | https://developer.android.com/tools/agents/android-cli |
| Gradle | 8.14.3 | https://gradle.org/releases/ |
| OpenJDK | 21 (LTS) | https://openjdk.org/ |
| Node.js | ≥ 22.12 | https://nodejs.org/ |

---

## 1. Install OpenJDK 21

Via repository (Ubuntu/Mint):

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
```

Or, from the official site (`https://openjdk.org/`), download the tarball:

```bash
curl -L -o /tmp/jdk21.tar.gz \
  "https://download.java.net/java/GA/jdk21.0.2/f2283984656d49d69e91c558476027ac/36/GPL/openjdk-21.0.2_linux-x64_bin.tar.gz"

sudo mkdir -p /usr/lib/jvm
sudo tar -xzf /tmp/jdk21.tar.gz -C /usr/lib/jvm
```

> In this project the installed path is `/usr/lib/jvm/java-21-openjdk-amd64`.

## 2. Install Gradle 8.14.3

```bash
curl -L -o /tmp/gradle.zip \
  "https://services.gradle.org/distributions/gradle-8.14.3-bin.zip"

sudo unzip -q /tmp/gradle.zip -d /opt
```

> The result will be `/opt/gradle-8.14.3`.

## 3. Install the Android CLI (agent)

```bash
curl -fsSL https://dl.google.com/android/cli/latest/linux_x86_64/install.sh | bash
```

Verify:

```bash
android --version
```

Install the Android SDK components:

```bash
android sdk install "platform-tools" "platforms/android-35" "build-tools;34.0.0"
```

> To list available packages: `android sdk list --all`.

## 4. Set up the environment variables

Example already used in this project — add to **`~/.zshrc`** (or `~/.bashrc`):

```bash
# Android SDK
export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export JAVA_HOME="/usr/lib/jvm/java-21-openjdk-amd64"
export GRADLE_HOME="/opt/gradle-8.14.3"
export PATH="$GRADLE_HOME/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/build-tools/34.0.0:$ANDROID_HOME/cmdline-tools/latest/bin:$JAVA_HOME/bin:$PATH"
```

Apply and verify:

```bash
source ~/.zshrc
java -version          # should show 21
gradle -v              # should show 8.14.3
adb version
apksigner version      # should show the apksigner version
```

---

## 5. Install the project and add Capacitor

```bash
cd path/to/quasar-project
npm install
npx quasar mode add capacitor --app-id "com.mercadolocal.app" --app-name "Mercado Local"
```

---

## 6. Generate the keystore (first time only)

```bash
keytool -genkeypair -v \
  -keystore my-release-key.jks \
  -alias my-key-alias \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass "my-secure-password123" \
  -keypass "my-secure-password123" \
  -dname "CN=Android Developer, O=MyCompany, C=MZ"
```

> Keep the `.jks` file and passwords in a safe place. Losing them prevents
> future updates of the app.

## 7. Build (Quasar + Capacitor)

```bash
npx quasar build -m capacitor -T android
```

This compiles the SPA, syncs it with the Android project
(`src-capacitor/android`) and attempts `gradlew assembleRelease`.

> If the `gradlew` (wrapper) fails because it cannot download Gradle, go on
> to step 8 — we use the **system Gradle** (`gradle`).

## 8. Build with the system Gradle

```bash
cd src-capacitor/android
gradle assembleRelease
cd ../..
```

Unsigned APK:

```
src-capacitor/android/app/build/outputs/apk/release/app-release-unsigned.apk
```

## 9. Locate `apksigner`

The `apksigner` lives inside one of the SDK's `build-tools` folders:

```bash
ls "$ANDROID_HOME/build-tools/"
```

The output shows the installed versions (e.g. `34.0.0`). The full path to
the binary is then:

```bash
export APKSIGNER="$ANDROID_HOME/build-tools/34.0.0/apksigner"
"$APKSIGNER" version
```

> If step 4 already added `$ANDROID_HOME/build-tools/34.0.0` to the `PATH`,
> you can just use `apksigner` directly.

## 10. Sign the APK

```bash
cd src-capacitor/android

UNSIGNED_APK="app/build/outputs/apk/release/app-release-unsigned.apk"
SIGNED_APK="app/build/outputs/apk/release/app-release-signed.apk"

rm -f "$SIGNED_APK"

"$APKSIGNER" sign --ks "../../my-release-key.jks" \
                  --ks-key-alias "my-key-alias" \
                  --ks-pass "pass:my-secure-password123" \
                  --key-pass "pass:my-secure-password123" \
                  --out "$SIGNED_APK" \
                  "$UNSIGNED_APK"
```

Verify the signature:

```bash
"$APKSIGNER" verify "$SIGNED_APK"
```

## 11. Final APK

The signed APK is at:

```
src-capacitor/android/app/build/outputs/apk/release/app-release-signed.apk
```

---

## Reference script

The logic of steps 6–8 and 10 is summarized in `p.txt`. It can be turned into
an executable script (`bash p.txt`) after adjusting the passwords and setting
`APKSIGNER` as in step 9.

## Troubleshooting

- **`command not found: gradle`** → `GRADLE_HOME/bin` is not on the `PATH`.
- **`command not found: apksigner`** → `$ANDROID_HOME/build-tools/<version>` is
  missing from the `PATH`; use the full path as in step 9.
- **`Failed to find target with hash string 'android-35'`** → missing
  `android sdk install "platforms/android-35"`.
- **`java version 17/26`** → Gradle/AGP needs JDK 21; make sure `JAVA_HOME`
  points to JDK 21 and `java -version` shows 21.
- **Slow gradle-all download** → in `src-capacitor/android/gradle/wrapper`,
  replace `gradle-8.14.3-all.zip` with `gradle-8.14.3-bin.zip`.