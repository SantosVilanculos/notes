# APK Build Guide — Windows

This guide explains how to set up a **Windows** environment and build a
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

Download the **MSI** installer from https://openjdk.org/ (Windows x64) and
run it. When done, note the install path, usually:

```
C:\Program Files\Java\jdk-21.0.2
```

## 2. Install Gradle 8.14.3

1. Download `gradle-8.14.3-bin.zip` from https://gradle.org/releases/.
2. Extract it to `C:\gradle-8.14.3`.

## 3. Install the Android CLI (agent)

In **PowerShell** (or CMD):

```cmd
curl.exe -fsSL https://dl.google.com/android/cli/latest/windows_x86_64/install.cmd -o "%TEMP%\i.cmd" && "%TEMP%\i.cmd"
```

Verify and install the SDK components:

```cmd
android --version
android sdk install platform-tools platforms/android-35 build-tools;34.0.0
```

> To list available packages: `android sdk list --all`.
> Note: in the Windows preview the `android emulator` subcommand is disabled;
> everything else works as expected.

## 4. Set up the environment variables

In **PowerShell** (run as the current user):

```powershell
setx ANDROID_HOME "$env:USERPROFILE\Android\Sdk"
setx ANDROID_SDK_ROOT "$env:USERPROFILE\Android\Sdk"
setx JAVA_HOME "C:\Program Files\Java\jdk-21.0.2"
setx GRADLE_HOME "C:\gradle-8.14.3"
```

Add the `bin` folders to the user's `PATH` (PowerShell):

```powershell
$old = [Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::SetEnvironmentVariable("Path",
  "$old;$env:USERPROFILE\Android\Sdk\platform-tools;$env:USERPROFILE\Android\Sdk\cmdline-tools\latest\bin;%JAVA_HOME%\bin;%GRADLE_HOME%\bin;$env:USERPROFILE\Android\Sdk\build-tools\34.0.0",
  "User")
```

Open a **new** terminal and verify:

```cmd
java -version          rem  should show 21
gradle -v              rem  should show 8.14.3
adb version
apksigner version      rem  should show the apksigner version
```

---

## 5. Install the project and add Capacitor

```cmd
cd path\to\quasar-project
npm install
npx quasar mode add capacitor --app-id "com.mercadolocal.app" --app-name "Mercado Local"
```

## 6. Generate the keystore (first time only)

```cmd
keytool -genkeypair -v ^
  -keystore my-release-key.jks ^
  -alias my-key-alias ^
  -keyalg RSA -keysize 2048 -validity 10000 ^
  -storepass "my-secure-password123" ^
  -keypass "my-secure-password123" ^
  -dname "CN=Android Developer, O=MyCompany, C=MZ"
```

> Keep the `.jks` file and passwords in a safe place. Losing them prevents
> future updates of the app.

## 7. Build (Quasar + Capacitor)

```cmd
npx quasar build -m capacitor -T android
```

This compiles the SPA, syncs it with the Android project
(`src-capacitor\android`) and attempts `gradlew assembleRelease`.

> If the `gradlew` (wrapper) fails because it cannot download Gradle, go on
> to step 8 — we use the **system Gradle** (`gradle`).

## 8. Build with the system Gradle

```cmd
cd src-capacitor\android
gradle assembleRelease
cd ..\..
```

Unsigned APK:

```
src-capacitor\android\app\build\outputs\apk\release\app-release-unsigned.apk
```

## 9. Locate `apksigner`

The `apksigner` lives inside one of the SDK's `build-tools` folders:

```cmd
dir /b "%ANDROID_HOME%\build-tools"
```

The output shows the installed versions (e.g. `34.0.0`). The full path to
the binary is `apksigner.bat`:

```cmd
set APKSIGNER=%ANDROID_HOME%\build-tools\34.0.0\apksigner.bat
"%APKSIGNER%" version
```

> If step 4 already added `%ANDROID_HOME%\build-tools\34.0.0` to the `PATH`,
> you can just use `apksigner` directly.

## 10. Sign the APK

```cmd
cd src-capacitor\android

set UNSIGNED_APK=app\build\outputs\apk\release\app-release-unsigned.apk
set SIGNED_APK=app\build\outputs\apk\release\app-release-signed.apk

del /f "%SIGNED_APK%"

"%APKSIGNER%" sign --ks "..\..\my-release-key.jks" ^
                   --ks-key-alias "my-key-alias" ^
                   --ks-pass "pass:my-secure-password123" ^
                   --key-pass "pass:my-secure-password123" ^
                   --out "%SIGNED_APK%" ^
                   "%UNSIGNED_APK%"
```

Verify the signature:

```cmd
"%APKSIGNER%" verify "%SIGNED_APK%"
```

## 11. Final APK

The signed APK is at:

```
src-capacitor\android\app\build\outputs\apk\release\app-release-signed.apk
```

---

## Reference script

The logic of steps 6–8 and 10 is summarized in the `p.txt` file (bash
version). On Windows simply reproduce the commands above in a terminal.

## Troubleshooting

- **`'gradle' is not recognized`** → `%GRADLE_HOME%\bin` is not on the `PATH`.
- **`'apksigner' is not recognized`** → `%ANDROID_HOME%\build-tools\<version>`
  is missing from the `PATH`; use the full path as in step 9.
- **`Failed to find target with hash string 'android-35'`** → missing
  `android sdk install platforms/android-35`.
- **`java version 17/26`** → Gradle/AGP needs JDK 21; check `JAVA_HOME` and
  reopen the terminal after `setx`.
- **Slow gradle-all download** → in `src-capacitor\android\gradle\wrapper`,
  replace `gradle-8.14.3-all.zip` with `gradle-8.14.3-bin.zip`.