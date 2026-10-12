# 16 · Build y despliegue Android

Target único: **Android** (móvil + Android TV). No existe carpeta `ios/`.

---

## 1. Toolchain

| Herramienta | Versión |
|---|---|
| Flutter | 3.35.x stable (`revision: 6a19cca56475dbfba1478ee68d7bd0c2ef891da1`) |
| Dart SDK | `^3.13.4` |
| JDK | 17 |
| compileSdk / targetSdk | 36 |
| minSdk | `flutter.minSdkVersion` (24 por defecto en Flutter actual) |

`.metadata` (36 líneas):

```yaml
version:
  revision: "6a19cca56475dbfba1478ee68d7bd0c2ef891da1"
  channel: "stable"
project_type: app
migration:
  platforms: [root, android, web, windows]   # todas con la misma revisión
unmanaged_files:
  - 'lib/main.dart'
  - 'ios/Runner.xcodeproj/project.pbxproj'
```

`lib/main.dart` está marcado como *unmanaged* → el tool de migración nunca lo
sobrescribirá. Referencia a un `ios/` que **no existe** (residuo de plantilla).

---

## 2. `android/app/build.gradle.kts` (51 líneas)

```kotlin
namespace     = "com.example.lol"
compileSdk    = 36
ndkVersion    = flutter.ndkVersion
minSdk        = flutter.minSdkVersion
targetSdk     = 36
isCoreLibraryDesugaringEnabled = true
Java/Kotlin   = 17
applicationId = "com.example.lol"

buildTypes {
  release {
    signingConfig = signingConfigs.getByName("debug")   // ⚠️ FIRMA CON DEBUG KEY
    isMinifyEnabled = true
    isShrinkResources = true
    proguardFiles(default + "proguard-rules.pro")
  }
}

dependencies {
  coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.4")
}
```

### 2.1 Problemas críticos

| # | Problema | Impacto |
|---|---|---|
| 1 | ⚠️ **`applicationId = "com.example.lol"`** | **No publicable en Play Store** |
| 2 | ⚠️ **Release firmado con la clave de debug** (`signingConfigs.getByName("debug")`) | La build "release" **no se puede publicar** |
| 3 | `minSdk` no fijado explícitamente | Depende de la versión de Flutter |
| 4 | Sin `ndk.abiFilters` | Aunque hay selector de ABI en las pantallas de actualización, Gradle empaqueta todas |

### 2.2 Cómo arreglar la firma

```kotlin
// android/key.properties (NO subir al repo)
storePassword=...
keyPassword=...
keyAlias=upload
storeFile=/ruta/a/upload-keystore.jks
```

```kotlin
// android/app/build.gradle.kts
val keystoreProperties = java.util.Properties()
val keystorePropertiesFile = rootProject.file("key.properties")
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(keystorePropertiesFile.inputStream())
}

android {
    signingConfigs {
        create("release") {
            keyAlias = keystoreProperties["keyAlias"] as String
            keyPassword = keystoreProperties["keyPassword"] as String
            storeFile = file(keystoreProperties["storeFile"] as String)
            storePassword = keystoreProperties["storePassword"] as String
        }
    }
    buildTypes {
        release { signingConfig = signingConfigs.getByName("release") }
    }
}
```

### 2.3 Fijar ABIs

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a", "armeabi-v7a")
        }
    }
}
```

`flutter build apk --release --split-per-abi` genera además
`app-armeabi-v7a-release.apk`, `app-arm64-v8a-release.apk` y
`app-universal-release.apk` — que es exactamente lo que espera el selector de
ABI de las pantallas de actualización.

### 2.4 `android/gradle.properties` ⚠️

Tiene **`org.gradle.jvmargs` DUPLICADO** (líneas 1 y 7). El último gana.

---

## 3. `AndroidManifest.xml` (144 líneas)

### 3.1 Permisos

| Permiso | Línea | Motivo |
|---|---|---|
| `INTERNET` | `:3` y `:6` (**duplicado**) | Red |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | `:4` | Descargas |
| `READ_EXTERNAL_STORAGE` (maxSdk 32) | `:5` | Descargas |
| `ACCESS_NETWORK_STATE` | `:7` | Red |
| `POST_NOTIFICATIONS` | `:9` | Notificaciones Android 13+ |
| `REQUEST_INSTALL_PACKAGES` | `:11` | Instalar APK de actualización |
| `WRITE_EXTERNAL_STORAGE` (maxSdk 28) | `:13` | Legacy |
| **`ACCESS_WIFI_STATE`** | `:16` | **Chromecast** |
| **`CHANGE_WIFI_MULTICAST_STATE`** | `:17` | **Chromecast (mDNS)** |
| **`NEARBY_WIFI_DEVICES`** (neverForLocation) | `:19-20` | Cast Android 12+ |
| `SCHEDULE_EXACT_ALARM` | `:31` | Recordatorios de capítulos |
| `USE_EXACT_ALARM` | `:32` | Ídem |
| `RECEIVE_BOOT_COMPLETED` | `:33` | Reprogramar tras reboot |
| `FOREGROUND_SERVICE` | `:34` | Descargas + Cast |
| `FOREGROUND_SERVICE_MEDIA_PLAYBACK` | `:35` | Cast (Android 14+) |
| `FOREGROUND_SERVICE_DATA_SYNC` | `:37` | Descargas |
| `WAKE_LOCK` | `:38` | Reproductor |

### 3.2 `uses-feature`

```xml
<!-- :22-24 -->
<uses-feature android:name="android.software.leanback"  android:required="false" />
<!-- :25-27 -->
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />
```

Una sola APK para móvil y TV.

### 3.3 `application`

```xml
android:label="lol"
android:banner="@drawable/banner"           <!-- :45, Android TV -->
android:enableOnBackInvokedCallback="true"
android:usesCleartextTraffic="true"          <!-- :47, necesario para HTTP -->
android:hardwareAccelerated="true"
<meta-data android:name="com.google.android.tv" android:value="true" />  <!-- :51-53 -->
```

### 3.4 `activity`

- `launchMode=singleTop`, `taskAffinity=""`
- `configChanges` con 12 flags (`:61`)
- `windowSoftInputMode=adjustResize`
- **Dos intent-filters**: `LAUNCHER` (`:72-76`) + **`LEANBACK_LAUNCHER`** (`:77-80`)

### 3.5 Componentes registrados

| Componente | Línea |
|---|---|
| `FileProvider` (para `open_filex`) | `:87-95` |
| `ScheduledNotificationReceiver` + `ScheduledNotificationBootReceiver` | `:98-110` |
| `ForegroundService` de `flutter_local_notifications` | `:113-117` |
| `ForegroundService` de `flutter_foreground_task` (`dataSync\|mediaPlayback`) | `:122-126` |

### 3.6 `queries`

`PROCESS_TEXT` y `application/vnd.android.package-archive` (`:133-143`) —
necesario para que el diálogo "Abrir con" aparezca al instalar el APK.

---

## 4. `MainActivity.kt` (214 líneas)

`class MainActivity : FlutterActivity()` con **3 MethodChannels**:

| Canal | Dirección | Propósito |
|---|---|---|
| `com.example.lol/intent` | Kotlin → Dart | Deep links `lol://user/<id>`, `lol://content/<id>`, `lol://list/<filter>` → `openDeepLink` (`:168-213`). ⚠️ **Sin receptor en Dart** |
| `tv_webview/click` | Dart → Kotlin | `injectClick(x, y)`: busca el `WebView` en el árbol de vistas y **sintetiza `MotionEvent` DOWN/UP** (`:129-162`) |
| `app/display_refresh` | Dart → Kotlin | `requestHighestRefreshRate` / `getRefreshRate` |

### 4.1 Refresh rate máximo

```kotlin
// MainActivity.kt:26-31 — se pide ANTES de que arranque Flutter
override fun onCreate(savedInstanceState: Bundle?) {
    requestHighestRefreshRate()
    super.onCreate(savedInstanceState)
}

// :75-110
private fun requestHighestRefreshRate() {
    val display = display ?: return
    val modes = display.supportedModes
    val best = modes.maxByOrNull { it.refreshRate } ?: return
    window.attributes = window.attributes.apply { preferredDisplayModeId = best.modeId }
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        val tx = SurfaceControl.Transaction()
        tx.setFrameRate(window.decorView.rootView, best.refreshRate, ...)
        display.transaction?.commit(tx)
    }
}
```

Llamado desde Dart en `mobile_shell.dart:105` y `tv_shell.dart:161` vía
`DisplayRefresh.requestHighest()`.

### 4.2 Click sintético en WebView (`:129-162`)

```kotlin
private fun findWebView(view: View?): WebView? {
    if (view is WebView) return view
    if (view is ViewGroup) {
        for (i in 0 until view.childCount) {
            findWebView(view.getChildAt(i))?.let { return it }
        }
    }
    return null
}
```

Se llama desde Dart cuando el player TV necesita interactuar con el WebView
embebido (los clicks del control remoto no llegaban).

---

## 5. Proguard

```
android/app/proguard-rules.pro
android/app/proguard_notifications.pro
```

`isMinifyEnabled = true` y `isShrinkResources = true` en release.

⚠️ La app usa **reflexión** en varios sitios (objetivos dinámicos en QuickJS,
`StremioCollectionSource.fromJson`, `(state as dynamic).refresh()`). Si la
ofuscación rompe algo, hay que añadir `-keep class com.example.lol.** { *; }`
y las reglas de los plugins.

---

## 6. Auto-actualización

La app **se actualiza sola**:

1. `UpdateNotificationService.instance.checkAndNotify()` en `main.dart:42`
   (fire-and-forget) consulta `https://www.modlyo.com/lol/versiones.php?tipo=lol`
2. Compara con `currentVersionName = "1.0.3"` / `currentVersionCode = 7`
3. Si hay versión nueva, muestra un **banner** en el shell (móvil:
   `showModalBottomSheet` `:358`; TV: `showDialog` + `_AbiPickerDialog` `:1423`)
4. El usuario elige la **ABI** (arm64 / armeabi / x86 / universal)
5. `_downloadAndInstallApk` (`:540` móvil, `:377` TV) descarga el APK
6. `open_filex` lo abre con el `FileProvider` → diálogo del sistema para instalar

**Permiso necesario**: `REQUEST_INSTALL_PACKAGES` — el usuario debe concederlo
manualmente en Ajustes del sistema la primera vez.

⚠️ **El código de descarga está duplicado literalmente** entre `mobile_shell.dart`
y `tv_shell.dart` (`_detectDeviceAbi`, `_checkForUpdate`, `_downloadAndInstallApk`,
`_urlForAbi`, `_labelAbi`, `_buildUpdateBanner`).

---

## 7. Optimización de tamaño

### 7.1 Eliminar dependencias muertas

Estas 4 paquetes están declaradas pero **no se importan** en ningún sitio:

```bash
# Verificar antes de eliminar
grep -rn "package:media_kit\|package:chewie\|package:fvp\|package:libtorrent_flutter" lib/
```

| Paquete | Ahorro estimado |
|---|---|
| `media_kit` + `media_kit_video` + `media_kit_libs_video` | ~10-15 MB |
| `libtorrent_flutter` | ~3-5 MB |
| `fvp` | ~1 MB |
| `chewie` | ~0.5 MB |
| `provider`, `go_router` (sin usar, pero Dart puro) | ~0 MB |

> ⚠️ `torrent_mkv_enabled` y el filtro `_isTorrentOrMkv` quedarían sin sentido
> si se elimina `libtorrent_flutter` — hay que quitar también el switch de Ajustes.

### 7.2 Split por ABI

```bash
flutter build apk --release --split-per-abi
```

Reduce el APK ~60 % frente al universal.

### 7.3 Imágenes

- `assets/images/` contiene PNG de logos de proveedores (Netflix, Disney+,
  Crunchyroll, etc.) y logos de metacrítics (IMDb, Metacritic, Letterboxd,
  Rotten Tomatoes). Todos en PNG sin optimizar.
- `assets/images/proveedores/` tiene 9 JPEG de logos de servicios.
- `assets/logo.png` (~?) se usa tanto en el launcher como en la pantalla de login.

Optimizar con `flutter_launcher_icons` ya configurado:

```yaml
flutter_launcher_icons:
  android: true
  ios: false
  image_path: "assets/logo.png"
  adaptive_icon_background: "#ffffff"
  adaptive_icon_foreground: "assets/logo.png"
  min_sdk_android: 21
```

---

## 8. Compilados y target

### 8.1 Target real: Android

### 8.2 `web/` — scaffolding muerto ⚠️

4 ficheros: `index.html` (46 líneas, **plantilla original** con
`<title>lol2</title>` y `description: "A new Flutter project."`),
`manifest.json` (18 líneas, `background_color: #0175C2` = azul por defecto de
Flutter), `favicon.png`, `icons/`.

**Imposible que funcione**: el código usa `dart:io` (`main.dart:2`,
`tvchanel_repository.dart:2`, `network_checker.dart`, …) y plugins nativos
(`video_player`, `webview_flutter`, `wakelock_plus`, `supabase_flutter`,
`dart_cast`).

### 8.3 `windows/` — scaffolding muerto

`CMakeLists.txt`, `flutter/generated_plugin_registrant.cc`, `runner/` con
`main.cpp`, `flutter_window.cpp`, `win32_window.cpp`, `utils.cpp`, `Runner.rc`
(con los iconos por defecto de Flutter), `runner.exe.manifest`.

Nunca compilado.

---

## 9. Basura en el repo

| Elemento | Ubicación | Acción |
|---|---|---|
| **5 logs de crash de JVM** | `android/hs_err_pid*.log` | Borrar |
| `sedpnFtyH` (0 bytes) | `lib/data/datasources/remote/tmdb/` | Borrar (blob git `e69de29…`) |
| `lol2.iml` | raíz | Residual de IntelliJ |
| `.idea/`, `.vscode/` | raíz | Commitear solo `.vscode/` si aporta algo |
| `lib/core/utils/prueba.dart` (0 bytes) | `lib/core/utils/` | Borrar |
| `lib/features/player/presentation/tv/server_loader.dart` (0 bytes) | player | Borrar |
| `lib/features/player/presentation/tv/tv_extractor_screen.dart` (0 bytes) | player | Borrar |
| 5 `*_server_scraper.dart` (0 bytes) | `lib/data/scrapers/servers/` | Borrar |
| `assets/js/crypto-js.min.js` | assets | ¿Se usa? El runtime usa `crypto-js.bundle.js` + CDN. Verificar |
| `.flutter-plugins-dependencies` | raíz | Generated; debería estar en `.gitignore` |

### 9.1 `.gitignore` actual

```
.gradle/
.dart_tool/
build/
*.hprof
*.apk
```

**Falta**: `android/key.properties`, `*.jks`, `*.keystore`, `local.properties`,
`.flutter-plugins-dependencies`, `android/app/debug/`, `android/app/profile/`,
`.idea/workspace.xml`, `hs_err_pid*.log`.

---

## 10. Checklist antes de publicar

- [ ] Cambiar `applicationId` a un dominio propio
- [ ] Configurar firma de release con keystore real
- [ ] `minSdk` explícito (24 o el que necesites)
- [ ] `ndk.abiFilters` si usas split
- [ ] Fijar `versionName` / `versionCode` en `versiones.dart` **y** en
      `pubspec.yaml`
- [ ] Actualizar el endpoint de versiones con la URL real del APK
- [ ] Eliminar dependencias muertas
- [ ] Limpiar logs de crash del repo
- [ ] Completar `key.properties` en `.gitignore`
- [ ] Probar en **dispositivo físico**: los WebView, el D-pad y Chromecast no
      funcionan bien en emulador
- [ ] Verificar `flutter analyze` (habrá muchos avisos por los `catch` silenciosos)
- [ ] Probar el flujo completo de reproducción en un dispositivo real con
      distintos CDN (los headers anti-hotlink son críticos)
- [ ] Verificar permisos de notificaciones (Android 13+) y de instalación de
      paquetes