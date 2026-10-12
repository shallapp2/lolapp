# 00 · Instalación y puesta en marcha

## Requisitos

| Herramienta | Versión | Notas |
|---|---|---|
| **Flutter** | 3.35.x stable | Verificar con `flutter --version`. La revisión exacta del proyecto es `6a19cca56475dbfba1478ee68d7bd0c2ef891da1` (`.metadata`) |
| **Dart SDK** | `^3.13.4` | Requisito declarado en `pubspec.yaml` |
| **JDK** | 17 | `android/app/build.gradle.kts`: `jvmTarget = "17"`, `sourceCompatibility = "17"` |
| **Android SDK** | compileSdk 36, targetSdk 36 | `minSdk` = `flutter.minSdkVersion` (24 por defecto en Flutter actual) |
| **Android Studio** | Cualquiera reciente | Solo para el IDE; la CLI de Flutter suffit |

> **La app es Android-only.** No existe carpeta `ios/`. Las carpetas `web/` y
> `windows/` son *scaffolding* por defecto de Flutter que **no compilarían**
> (el código usa `dart:io` y plugins nativos). Ignóralas.

## Clonar el repositorio

```bash
git clone https://github.com/jostinsantos/lolapp.git
cd lolapp
```

Este repo es un **fork** de un proyecto original; la historia de commits es
corta (6 commits) y termina en `7d02f18 login supabase`.

## Dependencias

```bash
flutter pub get
```

### Assets requeridos

`pubspec.yaml` declara 3 bundles JavaScript que son **imprescindibles** para el
sistema de addons:

| Asset | Tamaño | Para qué |
|---|---|---|
| `assets/js/crypto-js.bundle.js` | 56 KB | Criptografía para addons JS (AES, etc.) |
| `assets/js/crypto-js.min.js` | 47 KB | Red de seguridad para crypto-js |
| `assets/js/cheerio.bundle.js` | 187 KB | Parseo HTML/selectors para addons JS |

Ambos bundles están **pre-compilados** con el patrón
`var __libExport = (()=>{ ... })()` y se cargan en el runtime con un truco
(`js_runtime.dart:_libLoaderJs`). Si desaparecen, los addons que usen `cheerio`
o `crypto-js` dejan de funcionar — y **cheerio no tiene fallback CDN**.

## Configuración obligatoria antes de compilar

### 1. Versión de la app

`lib/core/constants/versiones.dart:9-10`:

```dart
static const String currentVersionName = "1.0.3"; // versión aceptada
static const int    currentVersionCode = 7;      // version_code (parche)
static const String apiUrl = "https://www.modlyo.com/lol/versiones.php?tipo=lol";
```

La app consulta ese endpoint para saber si hay actualización y **se actualiza
sola descargando e instalando el APK** (`open_filex` + permiso
`REQUEST_INSTALL_PACKAGES`). Bajar el valor aquí desactiva las actualizaciones.

### 2. Supabase (opcional pero recomendado)

`lib/supabase/supabase_constants.dart:3-4` viene **vacío**:

```dart
const String kSupabaseUrl     = '';   // → https://TU_PROYECTO.supabase.co
const String kSupabaseAnonKey = '';   // → eyJ...
```

Mientras esté vacío, `isSupabaseConfigured == false`, la app arranca **en modo
invitado 100 % local** y toda la sincronización se desactiva. Ver
[09-supabase.md](09-supabase.md) para el schema SQL completo a recrear.

### 3. API key de TMDB (opcional)

Hay **3 claves hardcodeadas** en `lib/core/constants/tmdb_apis.dart:13-19` con
rotación aleatoria por petición. Para usar la tuya:

```dart
prefs.setBool('tmdb_use_custom_api', true);
prefs.setString('tmdb_custom_api_key', 'TU_KEY');
```

También se puede hacer desde *Ajustes → Contenido → API de TMDB*.

## Compilar y ejecutar

```bash
# Depuración en dispositivo/emulador conectado
flutter run

# APK release
flutter build apk --release

# Split por ABI (recomendado: reduce el tamaño ~60 %)
flutter build apk --release --split-per-abi

# App Bundle para Play Store
flutter build appbundle --release
```

### Compilar en TV (Android TV)

Es la **misma APK**. El `AndroidManifest.xml` declara:

- `android.software.leanback` con `required="false"` → aparece en el launcher de TV
- `LEANBACK_LAUNCHER` como intent-filter adicional
- `android.hardware.touchscreen` con `required="false"` → funciona sin pantalla táctil
- `android:banner` → banner obligatorio en TVs

El usuario elige si la app se comporta como móvil o como TV en el **primer
arranque** (pantalla `_buildModeSelector` en `main.dart:363`), y el cambio se
guarda en `SharedPreferences['app_mode']`.

## Primer arranque (flujo de onboarding)

```
1. Splash (2 s de delay artificial, main.dart:138)
2. ¿Existe 'app_mode'?
   ├─ NO → Selector "Móvil" / "TV"  (D-pad funcional desde el primer frame)
   └─ SÍ → aplica orientación (landscape para TV) y sigue
3. ¿'disclaimer_accepted' == true?
   ├─ NO → Aviso legal; "Rechazar" = SystemNavigator.pop() (cierra la app)
   └─ SÍ → sigue
4. ¿Hay perfil activo o modo invitado?
   ├─ NO → ProfileSelectionPage (allowDismiss: false)
   │        → Invitado | Login/Registro | Perfil existente (+ PIN)
   └─ SÍ → sigue
5. ¿Primer arranque? ('taste_onboarding_done_v1' + AddonsOnboardingPage.shouldShow())
   ├─ SÍ → TasteOnboardingPage (elegir posters de gusto)
   │     → AddonsOnboardingPage (instalar fuentes/catálogos)
   └─ NO → _openHome()
6. MainHome (mobile_shell.dart o tv_shell.dart)
```

> ⚠️ Si rechazas el disclaimer, la app **se cierra** en Android
> (`main.dart:205-209`). No hay forma de volver atrás.

## Configuración inicial del reproductor

Al entrar por primera vez se muestra `PlaybackSetupModal` (obligatorio):

- **Modo**: `Auto` (primer servidor que funciona) o `Manual` (eliges tú)
- **Idioma de audio**: `Latino` (`es_MX`) / `Castellano` (`es_ES`) / `Subtitulado` (`en_US`)
- Si es Manual: un toggle por cada una de las 14 fuentes registradas

Guarda: `seleccionar_servidores`, `idioma_audio_predeterminado`,
`verificar_servidores=true`, `un_servidor_por_idioma=false`,
`mostrar_servidores_player=true` y los flags `<fuente>_enabled`.

En TV este modal **no se puede saltar** (`tv_shell.dart:165`).

## Problemas frecuentes al arrancar

| Síntoma | Causa | Solución |
|---|---|---|
| La app tarda mucho en el splash | `AddonManager.init()` encadena 3 repos de red | Normal la primera vez; se cachea en `SharedPreferences` |
| Crash en `flutter_js` | Falta `assets/js/cheerio.bundle.js` | Verificar assets en `pubspec.yaml` |
| "Stream encontrado" pero no reproduce | Falta `Referer`/`Origin` en el CDN | Ver [10-reproductor.md § Headers](10-reproductor.md) |
| No aparecen fuentes en el modal | `native_sources_enabled == false` (default) | Activar en *Ajustes → Fuentes*, o instalar addons |
| Error de compilación en `sedpnFtyH` | Archivo vacío sin extensión | Ignorarlo; no se importa en ningún sitio |

## Lint y análisis

```bash
flutter analyze
```

`analysis_options.yaml` es el de por defecto de Flutter, con **todas las reglas
personalizadas comentadas**. El proyecto tiene ~200 `catch (_) {}` silenciosos
y varios `print()` (`versiones.dart:35`), así que el analizador producirá
muchos avisos que no se han revisado nunca.

## Recursos externos que la app consulta

| Servicio | URL | Para qué |
|---|---|---|
| TMDB API | `https://api.themoviedb.org/3` | Metadatos, catálogos, imágenes |
| TMDB Images | `https://image.tmdb.org/t/p/{size}` | Pósters, backdrops, logos |
| OMDb API | `https://www.omdbapi.com/` | Ratings IMDB, Metascore, runtime |
| Versiones | `https://www.modlyo.com/lol/versiones.php?tipo=lol` | Auto-actualización |
| Fuentes custom | `https://modlyo.com/lolfuentes/api.php?idcodigo=` | APIs PHP del usuario |
| Subtítulos custom | `https://modlyo.com/subtitulo/contenido/{id}/es_MX.vtt` | VTT por defecto |
| FuegoCine API | `https://www.modlyo.com/api/servidores.php` | Servidores ya resueltos |
| OpenSubtitles | `https://opensubtitles-v3.strem.io/subtitles/...` | Subtítulos Stremio |
| IntroDB | `https://api.introdb.app/segments` | Marcas de intro/outro |
| SkipDB | `https://api.skipdb.tv` | Fallback de IntroDB |
| Kilo AI | `https://api.kilo.ai/api/gateway` | LLM gratuito (sin API key) |
| GitHub Search | `https://api.github.com/search/repositories` | Comunidad de addons |
| GitHub Raw | `https://raw.githubusercontent.com/...` | Descarga de `index.js` de addons |