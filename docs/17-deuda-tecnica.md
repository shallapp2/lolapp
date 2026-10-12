# 17 · Problemas conocidos y deuda técnica

Inventario priorizado de todo lo que hay que arreglar en el proyecto.

---

## 1. Bloqueantes de publicación

| # | Problema | Ubicación | Impacto |
|---|---|---|---|
| 1 | **`applicationId = "com.example.lol"`** | `android/app/build.gradle.kts` | No publicable en Play Store |
| 2 | **Release firmado con la clave de debug** | `android/app/build.gradle.kts` | La build "release" no se puede publicar |
| 3 | **Credenciales de Supabase vacías** | `lib/supabase/supabase_constants.dart:3-4` | Toda la sincronización desactivada; la app corre solo en modo local |
| 4 | **El SQL de Supabase no está en el repo** | `INSTRUCCIONES.md:20` lo referencia | No se pueden crear las tablas → sync imposible |
| 5 | **Cero tests** | `test/widget_test.dart` está vacío (0 bytes) | Ninguna red de seguridad ante refactors |

---

## 2. Bugs y trampas activas

| # | Problema | Detalle |
|---|---|---|
| 6 | **`MainFuentes` declarada dos veces** | `source_aggregator.dart:596` y `language_aggregator.dart:529`. Importar ambas = error de compilación. Además divergen: 13 vs 9 fuentes |
| 7 | **Deep links muertos** | `MainActivity.kt:168-213` invoca `openDeepLink` pero **no hay `setMethodCallHandler` en Dart**. Los `lol://` se pierden |
| 8 | **`_goAuthAndOnboarding()` recursivo** | `main.dart:248-252`. Si el usuario no elige nada se llama a sí mismo |
| 9 | **Riesgo de stack overflow** | Si la `ProfileSelectionPage` con `allowDismiss: false` permitiera pop, el ciclo se repetiría |
| 10 | **`native_sources_enabled = false` por defecto** | Instalación limpia = **0 fuentes**. El usuario ve un modal vacío y no sabe por qué |
| 11 | **`hackstore_enabled` y `pelisplus_enabled` muertos** | Están en `kRegisteredSources` (toggle en Ajustes) pero **no** en `FuentesConfig.fuentesActivas`. El toggle no hace nada |
| 12 | **`show_unreleased_episodes` ignorado** | `tmdb_content.dart:64` fuerza `showUnreleasedEps: false` siempre |
| 13 | **`torrent_mkv_enabled` engañoso** | Filtra servidores torrent/MKV que **nunca se reproducen** (`libtorrent_flutter` sin uso) |
| 14 | **`SessionWatchTimer.shouldContinue`** | `session_watch_timer.dart:90` — getter que **siempre devuelve `true`** |
| 15 | **Sin gestión de cookies** | Ningún cookie jar. Los CDNs que exigen cookies fallan siempre |
| 16 | **`badCertificateCallback => true`** | `base_detail_scraper.dart:58` desactiva completamente la validación TLS |
| 17 | **`findWebView` con firma nullable** | `MainActivity.kt:131` declara `view: View?` pero usa `view.childCount` (`:159`). Compila por *smart cast* de `val`, pero es frágil |
| 18 | **Éxito silencioso al fallar** | `SupabaseProfiles.create()` devuelve `null` sin explicación si el trigger de 5 perfiles revienta (`supabase_profiles_page.dart:66-68`) |

---

## 3. Código muerto (≈ 4.800 líneas + duplicados)

### 3.1 Capas enteras sin usar

| Capa | Líneas | Verificación |
|---|---|---|
| `lib/domain/**` | 237 | 0 imports fuera de `domain/`. Los 9 repositorios y 6 servicios son `abstract class X {}` de 1 línea |
| `lib/core/result/` | 31 | `Result<` solo aparece en sus propias 15 líneas |
| `lib/core/errors/` | 23 | 0 `AppException`/`NetworkException`/... fuera de sus ficheros |
| `lib/core/network/` | 33 | 5 clases, 0 usos |
| `lib/core/storage/` | 21 | 4 clases placeholder |
| `lib/app/` | 46 | 6 ficheros placeholder; `app_router.dart` dice *"Placeholder for future go_router"* |

### 3.2 Archivos vacíos (0 bytes)

```
lib/data/scrapers/servers/cinehax_server_scraper.dart
lib/data/scrapers/servers/cuevana_server_scraper.dart
lib/data/scrapers/servers/pelisplus_server_scraper.dart
lib/data/scrapers/servers/serieskao_server_scraper.dart
lib/data/scrapers/servers/tioplus_server_scraper.dart
lib/features/player/presentation/tv/server_loader.dart
lib/features/player/presentation/tv/tv_extractor_screen.dart
lib/core/utils/prueba.dart
lib/data/datasources/remote/tmdb/sedpnFtyH     ← sin extensión, blob e69de29
```

### 3.3 Código duplicado literalmente

| Duplicado | Detalle |
|---|---|
| `extractor.dart` vs `extractor_mobil.dart` | **Byte-idénticos** (1325 líneas cada uno) — verificado con `diff` |
| `pelispedia_extractor.dart` / `seriesmetro_extractor.dart` / `smartpelis_extractor.dart` | **Clones**: solo cambian `_kBase` y el nombre de la clase |
| `result.dart` vs `success.dart` | Mismo `sealed class Result` + `Success` + `Failure` (15 líneas) |
| `NativeResolvers` en 3 sitios | `hls/hls_extractor.dart`, `features/player/data/native_resolvers.dart`, `features/player/data/extractor.dart` |
| `detectServer` con listas de espejos | En 3 sitios |
| `_detectDeviceAbi`, `_checkForUpdate`, `_downloadAndInstallApk`, `_urlForAbi`, `_labelAbi`, `_buildUpdateBanner` | ~200 líneas entre `mobile_shell.dart` y `tv_shell.dart` |
| `_selectLogo` de TMDB | 4 sitios casi idénticos |
| `tipos/Sinopsis/etc.` en scrapers de detalle | 5 ficheros con el mismo algoritmo |
| `extractors/providers/content/*` (6 ficheros) | Restos de una refactorización. 0 imports |
| `content/tioplus.dart` | Idéntico a `providers/tioplus_extractor.dart` |
| `app_mode` / `disclaimer_accepted` | Triplicados (`main.dart`, `app_config.dart`, `storage_keys.dart`) |
| Player completo | 4 variantes: móvil 2930 + TV 4505 + discover 2943 + webview 1821 = **12.199 líneas** |
| UI móvil/TV en general | >25.000 líneas duplicadas |

### 3.4 Dependencias declaradas sin usar

| Paquete | pubspec | Ahorro |
|---|---|---|
| `media_kit` + `media_kit_video` + `media_kit_libs_video` | `:23-25` | **~10-15 MB de APK** |
| `libtorrent_flutter` | `:37` | ~3-5 MB |
| `fvp` | `:35` | ~1 MB |
| `chewie` | `:36` | ~0.5 MB |
| `provider` | `:15` | Dart puro |
| `go_router` | `:19` | Dart puro |
| `permission_handler` | `:45` | nativo |
| `intl`, `json_annotation`, `equatable`, `flutter_svg` | `:29,30,32` | Dart puro |

### 3.5 Constantes muertas

| Constante | Ubicación | Nota |
|---|---|---|
| `PlayerConstants.speeds` | `core/constants/player_constants.dart` | No hay control de velocidad en la UI |
| `StorageKeys` | `core/constants/storage_keys.dart` | 4 constantes, 0 imports |
| `CacheManager` | `core/storage/cache_manager.dart` | 0 usos |
| `Logger` | `core/utils/logger.dart` | Todo el proyecto usa `debugPrint` |
| `sideAccent` | `app_constants.dart:6`, `tv_shell.dart:36` | **Declarada y nunca usada** → warning del analizador |
| `HttpHeadersHelper` | `core/network/http_headers.dart` | El único con contenido, sin uso |

---

## 4. Mantenibilidad

### 4.1 Ficheros gigantes

| Fichero | Líneas | Problema |
|---|---|---|
| `tv_player_page.dart` | **4505** | Imposible de mantener |
| `tv_content_page.dart` | 3943 | Ídem |
| `player_page.dart` | 2930 | Ídem |
| `tv_discover_player.dart` | 2943 | 3ª copia de la lógica de player |
| `tv_servers_modal.dart` | 2160 | Copia del modal |
| `tv_subtitle_selector.dart` | 2055 | Copia |
| `domain/mobile/player_screen.dart` | 2060 | 4ª copia |
| `mobile_shell.dart` | 2023 | Shell con lógica de negocio |
| `tv_shell.dart` | 2600 | Shell con lógica de negocio |
| `servers_modal.dart` | 1923 | UI + caché + verificación en un archivo |
| `tv_player_webview.dart` | 1821 | 5ª copia |
| `subtitle_selector.dart` | 1758 | UI + API + caché + SRT→VTT |
| `content_page.dart` | 3037 | UI + TMDB + `GuardadosBus` |
| `domain/mobile/pag.dart` | 1165 | `ServiciosPage` dentro de `domain/` |
| `home_page.dart` | 1669 | |
| `tv_home_page.dart` | 1970 | |
| `js_runtime.dart` | 1260 | |
| `settings_page.dart` | 1321 | |
| `sources_section.dart` | 1048 | |
| `extractor.dart` | 1325 | |
| `AddonLoader` | 952 | |
| `source_aggregator.dart` | 1196 | |
| `discovery/derivar.dart` | 1418 | |

**Ningún fichero supera las 600 líneas con una responsabilidad clara.**

### 4.2 UI dentro de carpetas `domain/`

`features/discover/domain/` contiene:
- `mobile/pag.dart` (1165) → `ServiciosPage`
- `mobile/player_screen.dart` (2060)
- `mobile/servidores_modal.dart` (938)
- `mobile/hls_quality_widget.dart` (690)
- `mobile/extractor.dart` (979)
- `mobile/derivar.dart` (677)
- `extractor.dart` (1327)
- `derivar.dart` (1418)

**8 de 13 ficheros de esta feature son UI** contradiciendo la arquitectura
declarada.

---

## 5. Rendimiento y red

| # | Problema | Detalle | Impacto |
|---|---|---|---|
| 1 | **Sin caché HTTP de TMDB** | `tmdb_apis.dart` usa rotación de claves, no caché. ~40 requests por entrada al Home | Latencia y cuota |
| 2 | **11 requests en paralelo al Home** | `tmdb_home_api.dart:617` | Se lanza 1 por cada sección × 3 claves rotadas |
| 3 | **Búsqueda de fuentes secuencial** | `registry.dart:129-172` — 5 fuentes en serie | +5 s de latencia |
| 4 | **`getFirstStream` secuencial** | `source_service.dart:59-83` — timeout 20 s por addon | +20 s si el primer addon falla |
| 5 | **Sin paginación del `TextField`** | — | — |
| 6 | **`getAllM3uCategories()` en serie** | `tvchanel_repository.dart:97-112` con `continue` silencioso | Una lista caída no avisa |
| 7 | **`rawCode` limitado a 80 KB** | `storage_service.dart:27-38` | Los addons grandes se re-descargan cada arranque |
| 8 | **Verificación HLS con WebView** | `_VerifyGate` limita a 1-5, pero sigue siendo caro | Cada verificación abre un WebView |

---

## 6. Seguridad y privacidad

| # | Problema | Detalle |
|---|---|---|
| 1 | **4 API keys de TMDB hardcodeadas** | `tmdb_apis.dart:13-19` + `lolbot_page.dart:121` + `for_you_generator.dart:81` |
| 2 | **API key de OMDb hardcodeada** (×3) | `5b0e8a3e`, `1289a38c`, `eb1327da` en `tmdb_content.dart` y `embed69_extractor.dart` |
| 3 | **Todo en `SharedPreferences` sin cifrar** | Progreso de visualización, historial, PIN de perfil, credenciales de API personalizadas |
| 4 | **TLS deshabilitado en un scraper** | `base_detail_scraper.dart:58` |
| 5 | **Runtime JS ejecuta código de terceros con red** | `JsAddonRuntime` — por diseño, pero implica que un addon malicioso puede enviar cualquier petición |
| 6 | **Pairing TV por código de 6 dígitos** | 10⁶ combinaciones, sin rate limiting |
| 7 | **`usesCleartextTraffic=true`** | Necesario para HTTP, pero permite interceptación |
| 8 | **Historial enviado a un tercero** | Títulos vistos + géneros + likes van a `api.kilo.ai` para las recomendaciones |
| 9 | **Policies RLS de `tv_link` abiertas** | Hay que diseñarlas con cuidado (ver [09-supabase.md §12](09-supabase.md)) |

---

## 7. Funcionalidad anunciada pero ausente

| Funcionalidad | Estado |
|---|---|
| **EPG de TV** | ❌ Solo un Snackbar "EPG próximamente" (`home_tvchanel_tv.dart:978-987`) |
| **Control de velocidad de reproducción** | ❌ `PlayerConstants.speeds` existe pero no hay UI |
| **Reproducción torrent/MKV** | ❌ Se detecta y filtra, `libtorrent_flutter` sin uso |
| **Tema claro** | ❌ `ThemeMode` no aparece en el repo. `user_settings.dart` tiene `themeMode` pero el modelo **nunca se instancia** |
| **Búsqueda de canales en móvil** | ❌ Solo TV |
| **Favoritos de canales en móvil** | ❌ Solo TV |
| **Guía de canales en móvil** | ❌ Solo TV (existe en el player TV) |
| **Deep links `lol://`** | ❌ El canal Kotlin no tiene receptor Dart |
| **`core/result` y `core/errors`** | ❌ Diseñados, nunca usados |
| **Router declarativo** | ❌ `go_router` declarado y sin usar |

---

## 8. Calidad de código

| # | Problema | Detalle |
|---|---|---|
| 1 | **~200 `catch (_) {}` silenciosos** | Los errores son invisibles en producción |
| 2 | **`debugPrint` en vez de `Logger`** | `core/utils/logger.dart` existe y no se usa |
| 3 | **`print()` en `versiones.dart:35`** | Violación del lint |
| 4 | **Todas las reglas de lint comentadas** | `analysis_options.yaml` — `linter.rules` está vacío |
| 5 | **`flutter analyze` no se ejecuta** | Con tantos avisos, nadie lo revisa |
| 6 | **Strings de prefs hardcodeados en ~70 ficheros** | Sin namespacing. Riesgo de colisión |
| 7 | **`Listenable` registration manual** | `ChangeNotifier` + `addListener` sin provider: riesgo de listeners duplicados y fugas |
| 8 | **Doble arquitectura del player** | La lógica de caché/headers/resume está 4 veces duplicada |
| 9 | **Rompe el "Source of Truth"** | `web/index.html` con `<title>lol2</title>` y `manifest.json` con el azul por defecto de Flutter |

---

## 9. Basura en el repositorio

| Elemento | Ubicación |
|---|---|
| **5 logs de crash de JVM** | `android/hs_err_pid13168.log`, `hs_err_pid4540.log`, `hs_err_pid16256.log`, `hs_err_pid13564.log`, `hs_err_pid5464.log` |
| `sedpnFtyH` (0 bytes, blob `e69de29…`) | `lib/data/datasources/remote/tmdb/` |
| `lol2.iml` | raíz |
| `.idea/`, `.vscode/` commiteados | raíz |
| `assets/js/crypto-js.min.js` | ¿se usa? El runtime usa `crypto-js.bundle.js` + CDN |
| `.flutter-plugins-dependencies` (25 KB) | Generated |
| 8 ficheros Dart vacíos | ver §3.2 |
| 6 ficheros duplicados en `providers/content/` | restos de refactorización |
| `web/` y `windows/` completos | scaffolding que no compila |

---

## 10. `.gitignore` insuficiente

```
.gradle/
.dart_tool/
build/
*.hprof
*.apk
```

**Falta**:
```
android/key.properties
*.jks
*.keystore
local.properties
.flutter-plugins-dependencies
android/app/debug/
android/app/profile/
android/hs_err_pid*.log
.idea/workspace.xml
.idea/tasks.xml
```

---

## 11. Plan de refactor sugerido

### Fase 1 — Higiene (1 día)

1. Borrar ficheros vacíos, duplicados y basura (`sedpnFtyH`, 5 `hs_err_*.log`,
   `providers/content/`, `extractor_mobil.dart`, `web/`, `windows/`, `lol2.iml`)
2. Eliminar `media_kit*`, `fvp`, `chewie`, `libtorrent_flutter` y el switch
   `torrent_mkv_enabled`
3. Ampliar `.gitignore`
4. Activar `flutter_lints` con reglas reales y arreglar lo que salga
5. Documentar los hacks (`_isCurrentRoute` en `main.dart:204`)

### Fase 2 — Tests (3-5 días)

1. Tests unitarios de los parsers puros:
   - `m3u_parser.dart` (`groupByCategory`, `_extractAttr`)
   - `normalizeIdioma` (`source_aggregator.dart:765`)
   - `HlsQualityParser.parse`
   - `HlsHeadersBuilder` (`player_page.dart:668`)
   - `unwrapStreamKey` (`source_service.dart:243`)
   - `parseId` (`tmdb_service.dart:477`)
2. Tests golden de los providers con HTML capturado de cada sitio
3. Un test de integración del flujo: `TMDB id → MainFuentes → PlayerScreen`

### Fase 3 — Unificar la arquitectura (2-4 semanas)

1. Introducir `provider` (ya declarado) y eliminar los `addListener` manuales
2. Eliminar `lib/domain/` o implementarlo de verdad
3. Introducir `go_router` (ya declarado) y eliminar los 99 `Navigator.push`
4. Extraer los 4 players a un `PlayerCore` compartido
5. Extraer las shells: `_detectDeviceAbi`, `_checkUpdate`, etc.
6. Crear un `StorageKeys` real y migrar las ~70 claves hardcodeadas
7. Empezar a dividir los ficheros > 1000 líneas por responsabilidad

### Fase 4 — Seguridad (1 semana)

1. Mover las API keys a `flutter_secure_storage` o a un build config
2. Quitar `badCertificateCallback`
3. Añadir rate limiting al pairing TV
4. Redactar los logs en release

---

## 12. Métricas del proyecto

| Métrica | Valor |
|---|---|
| Ficheros Dart | 325 |
| Líneas Dart | ~153.000 |
| Líneas de código muerto | ~4.800 (core+domain+app) + ~10.000 (duplicados) |
| Ficheros > 1000 líneas | ~25 |
| Ficheros vacíos | 9 |
| Funciones/preferencias distintas | ~45 paquetes declarados, ~22 usados |
| Dependencias declaradas sin usar | 11 |
| Tests | 0 |
| Clases abstract vacías | 15 |
| Bugs activos identificados | 18 |
| Funcionalidades anunciadas y ausentes | 10 |