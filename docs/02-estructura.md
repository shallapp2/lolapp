# 02 · Estructura de carpetas

Mapa completo de `lib/` con descripción de cada módulo.

## Resumen numérico

| Capa | Ficheros | Líneas aprox. |
|---|---|---|
| `lib/main.dart` | 1 | 679 |
| `lib/app/` | 6 | 46 |
| `lib/core/` | 35 | 1.053 |
| `lib/data/` | 85 | ~29.000 |
| `lib/domain/` | 44 | 237 |
| `lib/features/` | 143 | ~116.000 |
| `lib/presentation/` | 3 | ~5.300 |
| `lib/supabase/` | 8 | 1.127 |
| **Total** | **325** | **~153.000** |

---

## `lib/main.dart` (679 líneas)

Bootstrap de la aplicación + `SplashScreen` como máquina de estados.

| Bloque | Líneas | Contenido |
|---|---|---|
| Constantes de estado | 25-27 | `kModeKey='app_mode'`, `kDisclaimerKey`, `kTasteOnboardingKey` |
| `main()` | 29-73 | Orden de inicialización (ver [03-arranque](03-arranque-y-navegacion.md)) |
| `MyApp` | 75-91 | `MaterialApp` mínimo, tema dark hardcodeado |
| `SplashScreen` | 93-481 | Estados: `loading` → `mode` → `disclaimer` → auth → onboarding → home |
| `_ModeButton` | 485-599 | Botón con soporte D-pad del selector de modo |
| `_FocusButton` | 603-679 | Botón Aceptar/Rechazar del disclaimer |

Imports con alias que permiten dos `MainHome` homónimos:

```dart
import 'presentation/mobile/mobile_shell.dart' as mobile;
import 'presentation/tv/tv_shell.dart' as tv;
// ...
final home = mode == 'tv' ? const tv.MainHome() : const mobile.MainHome();
```

---

## `lib/app/` — 6 ficheros, 46 líneas ⚠️ SCAFFOLDING

| Fichero | Líneas | Contenido | ¿Se usa? |
|---|---|---|---|
| `app.dart` | 9 | `class App extends StatelessWidget` que devuelve `MyApp()` | ❌ (además crea dependencia `app → main`) |
| `app_config.dart` | 6 | `appName`, `modeKey`, `disclaimerKey` (duplicados) | ❌ |
| `app_router.dart` | 5 | `class AppRouter { AppRouter._(); }` con comentario *"Placeholder for future go_router"* | ❌ |
| `app_lifecycle.dart` | 9 | `WidgetsBindingObserver` vacío, sin sobrescribir `didChangeAppLifecycleState` | ❌ |
| `app_theme.dart` | 9 | `buildAppTheme()` idéntico al `ThemeData` inline de `main.dart` | ❌ |
| `app_constants.dart` | 8 | `accentColor 0xFFE50914`, `sideAccent 0xFF7B5CFF` | Solo por `app_theme.dart` |

---

## `lib/core/` — 35 ficheros, 1053 líneas

### `core/constants/` (7 ficheros)

| Fichero | Líneas | Contenido | ¿Activo? |
|---|---|---|---|
| `tmdb_apis.dart` | 92 | 3 API keys con rotación aleatoria, idioma global, géneros TV excluidos | ✅ |
| `sources.dart` | 443 | `FuenteId` enum + `kRegisteredSources` (14 fuentes) + `FuenteSeleccion` | ✅ |
| `versiones.dart` | 211 | `VersionService` (semver + `version_code` + URL de actualización) | ✅ |
| `storage_keys.dart` | 6 | 4 constantes de clave, **nunca importado** | ❌ |
| `app_urls.dart` | 4 | Placeholder | ❌ |
| `addon_constants.dart` | 3 | Solo `manifestFile = 'manifest.json'` | ❌ |
| `player_constants.dart` | 4 | `speeds = [0.5…2.0]` — **no hay control de velocidad en la UI** | ❌ |

### `core/errors/` (5 ficheros) — ❌ 100 % muerto

```
AppException (base, implements Exception)
├── NetworkException      network_exception.dart
├── ScraperException      scraper_exception.dart
├── ExtractorException    extractor_exception.dart
└── PlayerException       player_exception.dart
```

### `core/network/` (5 ficheros) — ❌ 100 % muerto

| Fichero | Contenido |
|---|---|
| `http_client.dart` | `class AppHttpClient { AppHttpClient._(); }` + comentario *"implementations still use Dio/http directly"* |
| `http_interceptor.dart` | `class HttpInterceptor {}` |
| `http_headers.dart` | `HttpHeadersHelper.defaultHeaders()` |
| `network_checker.dart` | `hasConnection()` con `InternetAddress.lookup` — **el shell móvil reimplementa esto en línea** |
| `request_cache.dart` | `Map` estático, sin uso |

> **No hay capa de red centralizada.** Cada scraper/extractor hace su propio
> `http.get` con sus propios headers y timeouts. `dio` solo se usa en
> `downloads/download_manager.dart`.

### `core/result/` (3 ficheros) — ❌ 100 % muerto

`result.dart` y `success.dart` contienen **el mismo código literal**
(`sealed class Result`, `Success<T>`, `Failure<T>`). `failure.dart` solo hace
`export 'result.dart'`.

### `core/storage/` (4 ficheros, 21 líneas) — ❌ 100 % muerto

```dart
class AppDatabase { AppDatabase._(); }      // "Placeholder for Hive / SQLite"
class SecureStorage { SecureStorage._(); }  // "Placeholder for flutter_secure_storage"
class CacheManager { /* Map en memoria */ }
class LocalStorage { /* wrapper de SharedPreferences */ }
```

> **No hay base de datos local.** Ni sqflite, ni Hive, ni Isar, ni Drift, ni
> Sembast. Todo va a `SharedPreferences` directamente desde ~70 ficheros.

### `core/services/` (1 fichero) ✅

| Fichero | Contenido |
|---|---|
| `session_watch_timer.dart` | `SessionWatchTimer.instance`, `warnAfter = 4 horas`, diálogo "Descanso recomendado" |

### `core/utils/` (11 ficheros)

| Fichero | Líneas | ¿Usado? | Contenido |
|---|---|---|---|
| `display_refresh.dart` | 50 | ✅ | `MethodChannel('app/display_refresh')` → `requestHighestRefreshRate()`. Requiere handler en `MainActivity.kt:57-68` |
| `prueba.dart` | 0 | ❌ | **Archivo vacío** |
| `image_utils.dart` | 7 | ❌ | `tmdbPoster()` — cada pantalla reimplementa su propio `_optimizeTmdbUrl` |
| `url_utils.dart` | 7 | ❌ | `isValid()` |
| `string_utils.dart` | 3 | ❌ | `nullIfEmpty` |
| `html_utils.dart` | 4 | ❌ | `stripTags` |
| `json_utils.dart` | 4 | ❌ | `asMap` |
| `date_utils.dart` | 6 | ❌ | `formatYear` |
| `logger.dart` | 5 | ❌ | `Logger.d/e` — todo el proyecto usa `debugPrint` |
| `platform_utils.dart` | 5 | ❌ | `isAndroid`/`isIOS` |

---

## `lib/data/` — 85 ficheros, ~29.000 líneas ← **LA LÓGICA REAL**

### `data/scrapers/` — Scraping de sitios web (catálogo)

```
scrapers/
├── base/
│   ├── registry.dart          172 L  ← PUNTO DE EXTENSIÓN
│   ├── scraper_context.dart    93 L  ← clase Fuente (descriptor funcional)
│   ├── base_home_scraper.dart  79 L  ← ScraperItem, ScraperResult, fetchHtml()
│   ├── base_detail_scraper.dart 92 L  ← duplica DTOs; fetchHtml con HttpClient
│   ├── buscador.dart          468 L  ← parsers de búsqueda (RegExp por sitio)
│   └── detalle_scraper.dart    56 L  ← dispatcher por id de servicio
├── home/      (5)  tioplus · pelisplus · serieskao · cinehax · cuevana
├── detail/    (5)  uno por sitio
└── servers/   (5)  ⚠️ TODOS VACÍOS (0 bytes)
```

→ Detalle en [04-scrapers.md](04-scrapers.md)

### `data/extractors/` — Resolución de URL de reproducción

```
extractors/
├── hls/hls_extractor.dart            1196 L  ← NativeResolvers + probe WebView 1x1
└── providers/                                  ← 13 providers
    ├── embed69_extractor.dart          797 L  ← PoW SHA-256 + AES-256-CBC (el más complejo)
    ├── pelispedia_extractor.dart       533 L  ┐
    ├── seriesmetro_extractor.dart      533 L  ├ clones: solo cambia _kBase y el nombre
    ├── smartpelis_extractor.dart       533 L  ┘
    ├── tioplus_extractor.dart          497 L  ← base64 → /player/<b64> → window.location
    ├── cuevana_extractor.dart          488 L  ← __NEXT_DATA__ + lista blanca de cyberlockers
    ├── poseidon_extractor.dart         465 L  ← clon de Cuevana con otro backend
    ├── pelisplus_extractor.dart        464 L  ← 3 métodos de extracción en cascada
    ├── hackstore_extractor.dart        372 L  ← API REST propia (WordPress)
    ├── cinecalidad_extractor.dart      340 L  ← base64 del token "zopass"
    ├── vidsrc_extractor.dart           305 L  ← base dominio dinámico + /rcp/<hash>
    ├── cinesrc_extractor.dart          302 L  ← 100 % constructivo, sin HTTP
    ├── unlimplay_extractor.dart        184 L  ← EMBEDS/finalizePlayer en JSON embebido
    ├── fuegocine_extractor.dart         68 L  ← API JSON externa (modlyo)
    └── content/                              ← ⚠️ DUPLICADOS MUERTOS (0 imports)
```

→ Detalle en [05-extractores.md](05-extractores.md)

### `data/aggregators/` — Orquestación de fuentes

| Fichero | Líneas | Clase | Estado |
|---|---|---|---|
| `source_aggregator.dart` | 1196 | `MainFuentes` | ✅ **Agregador vigente** (13 `FuenteId` + addons) |
| `language_aggregator.dart` | 975 | `MainFuentes` | ⚠️ **Legacy/TV** — declara las MISMAS clases con 9 fuentes y `_maxConcurrent = 1` |
| `server_aggregator.dart` | 392 | `ServidoresAggregator` | Agregador simple sin verificación |
| `main_fuentes_servidores.dart` | 272 | `MainFuentesServidores` | Ruta **por sitio** (`serieskao→embed69`, `cinehax→unlimplay`) |

> ⚠️ `MainFuentes` está **declarada dos veces** (`source_aggregator.dart:596` y
> `language_aggregator.dart:529`). No colisionan porque cada consumidor importa
> solo una. Importar ambas en un mismo library = error de compilación.

→ Detalle en [06-fuentes-y-agregadores.md](06-fuentes-y-agregadores.md)

### `data/addons/` — Sistema de addons (el más complejo)

```
addons/
├── addon_manager.dart            412 L  ChangeNotifier singleton
├── addon_source_adapter.dart     330 L  addons → modal de servidores (cola de workers)
├── README_ADDONS.md              53 L
├── models/
│   ├── addon.dart                757 L  AddonManifest, SourcePackage, PackageScraper
│   └── content.dart              342 L  ContentItem, StreamItem, SubtitleItem
├── services/
│   ├── addon_loader.dart         952 L  descarga/parseo de manifests (URL/GitHub/Nuvio)
│   ├── js_runtime.dart          1260 L  QuickJS + polyfills Node + bridges
│   ├── catalog_service.dart      441 L  home/search/discover/meta
│   ├── source_service.dart       529 L  resolución de streams
│   ├── tmdb_service.dart         509 L  TMDB "de addons" (key del propio addon)
│   ├── community_service.dart    181 L  GitHub Search API por topics
│   ├── storage_service.dart       77 L  SharedPreferences (podar rawCode > 80 KB)
│   └── catalog_js_bridge.dart     24 L  delegador thin
└── stremio/
    ├── stremio_addon_repository.dart     231 L
    ├── stremio_collection_repository.dart 304 L
    ├── stremio_stream_service.dart       225 L
    ├── stremio_manifest_parser.dart      163 L
    ├── stremio_models.dart               335 L
    ├── stremio_url_builder.dart          104 L
    ├── stremio_catalog_client.dart        57 L
    └── stream_badge_repository.dart      340 L
```

→ Detalle en [07-addons.md](07-addons.md)

### `data/datasources/remote/`

```
tmdb/
├── tmdb_home_api.dart            841 L  Home TV (11 endpoints en paralelo)
├── tmdb_home_mobile_api.dart     837 L  Home móvil + main_slider hero
├── tmdb_content.dart             854 L  Ficha completa + temporadas + OMDb
├── tmdb_player_api.dart          805 L  Datos del reproductor
├── tmdb_search_api.dart          214 L  /search/multi
├── tmdb_search.dart              419 L  ⚠️ DUPLICADO, no importado
├── tmdb_discover_api.dart        265 L  Descubrir por género
├── tmdb_recommendations_api.dart 223 L  Similares + colección
└── sedpnFtyH                       0 B  ⚠️ ARCHIVO VACÍO, blob git e69de29, sin refs
sources/
└── custom_api.dart               326 L  APIs PHP del usuario (modlyo)
```

→ Detalle en [08-tmdb.md](08-tmdb.md)

### `data/recommendations/` — Motor "Para ti"

| Fichero | Líneas | Contenido |
|---|---|---|
| `for_you_generator.dart` | 309 | 20 recomendaciones, gate de 20 h, verificación contra TMDB |
| `regional_top10.dart` | 334 | Top 10 LatAm por IA, TTL 24 h, umbrales anti-rareza |
| `daily_sections_generator.dart` | 274 | 4-6 secciones estilo Netflix, gate 24 h |
| `user_taste_profile.dart` | 142 | Perfil de gusto local (géneros + keywords con pesos) |
| `history_signals.dart` | 113 | Clasifica historial en `terminado` / `abandonado` / descartado |

### `data/ai/` — Cliente de IA

| Fichero | Líneas | Contenido |
|---|---|---|
| `ai_client.dart` | 231 | SSE streaming a `api.kilo.ai`, sin API key |
| `model_memory.dart` | 130 | Scoring de modelos + cooldowns |
| `ai_usage_tracker.dart` | 178 | Cuota 200 req/h, poda a 7 días |
| `daily_ai_gate.dart` | 58 | Gate de 24 h + `TasteOnboardingGate` |
| `lolbot_client.dart` | 127 | Guardrails del chat |

→ Detalle en [12-ia.md](12-ia.md)

### `data/models/scraper/detalle_model.dart` (91 líneas)

```dart
class DetalleServidor  { nombre, url, idioma, calidad }
class DetalleCapitulo  { temporada, numero, titulo, url, imagen, airDate, rating, duracion }
class DetalleTemporada  { numero, nombre, List<DetalleCapitulo> episodios }
class DetalleContenido { ok, error, servicio, titulo, tipo, anio, sinopsis, poster,
                         backdrop, logo, rating, tmdbId, imdbId, generos,
                         servidores, temporadas, extra }
```

---

## `lib/domain/` — 44 ficheros, 237 líneas ⚠️ CASCA TOTAL

```
domain/
├── models/          38 POJOs inmutables (Content, PlaybackState, UserSettings…)
│                    ⚠️ Nadie los importa. El contenido viaja como Map<String,dynamic>
├── repositories/     9 ficheros → abstract class XRepository {}   (1 línea cada uno)
└── services/        6 ficheros → abstract class XService {}      (1 línea cada uno)
```

Ejemplo literal de un repositorio:

```dart
// lib/domain/repositories/addon_repository.dart:1
abstract class AddonRepository {}
```

> **Confusión a evitar**: existe una clase `SourceService` **real** en
> `lib/data/addons/services/source_service.dart:13` que **no** implementa el
> `abstract class SourceService` de `lib/domain/services/`. Son homónimos
> sin relación alguna.

---

## `lib/features/` — 143 ficheros, ~116.000 líneas

| Feature | Ficheros | Descripción |
|---|---|---|
| `player/` | 28 | **El más grande.** `player_page.dart` (2930), `tv/tv_player_page.dart` (4505), `tv/tv_discover_player.dart` (2943), `tv/tv_player_webview.dart` (1821), calidad, subtítulos, cast, mini-player, skip, screensaver |
| `settings/` | 28 | 13 sub-secciones + variantes TV (ver abajo) |
| `discover/` | 13 | Catálogo por fuentes. ⚠️ `domain/mobile/` contiene **UI** (`pag.dart` 1165 L) |
| `tvchanel/` | 11 | IPTV: parser M3U, repositorio, home, player, config (móvil + TV) + `README.md` |
| `addons/` | 11 | Hub, onboarding, comunidad GitHub, packs Nuvio, Stremio |
| `content/` | 9 | `content_page.dart` (3037), `tv_content_page.dart` (3943), opciones, cast, temporadas |
| `profile/` | 9 | Selector estilo Netflix (móvil/TV), CRUD, PIN, página de perfil |
| `downloads/` | 8 | Descargas HLS segmentadas + `DownloadManager` + foreground service |
| `home/` | 7 | `home_page.dart` (1669), `tv_home_page.dart` (1970), secciones carousel |
| `servers/` | 7 | `servers_modal.dart` (1923), `tv_servers_modal.dart` (2160), precargadores |
| `foryou/` | 5 | "Para ti", onboarding de gustos, recap, botón "Me gusta" |
| `search/` | 2 | `search_page.dart` (1382), `tv_search_page.dart` (1106) |
| `auth/` | 2 | `login_register_page.dart` (565), `tv_login_register_page.dart` (961) |
| `favorites/` | 1 | `BibliotecaUnificadaPage` — tabs Guardados/Historial/Descargas |
| `history/` | 1 | `history_page.dart` — versión TV de la biblioteca |
| `lolbot/` | 1 | Chat de IA (solo móvil) |

### `features/settings/presentation/` — 13 sub-secciones

| Subcarpeta | Fichero | Contenido |
|---|---|---|
| `sources/` | `sources_section.dart` (1048) | **El más denso.** Master switch `native_sources_enabled`, toggle por fuente, verificación, TTL, concurrencia, APIs personalizadas |
| `updates/` | `tv_updates_tab.dart` (1703), `update_notification_service.dart` | Auto-actualización con selector de ABI |
| `actualizaciones/` | `actualizaciones_page.dart` (1008) | Equivalente móvil, `open_filex` |
| `player/` | `player_section.dart` (609) | Mini-player, tamaño de subtítulos, descarga IDM, estilo de subtítulo |
| `notifications/` | `notifications_section.dart` (696) | `EpisodeReminderService`, canal + hora (11:00 por defecto) |
| `downloads/` | `downloads_section.dart` (304) | Automático, concurrencia, calidad, carpeta |
| `content/` | `content_section.dart` (332) | TMDB API key, idioma metadata, secciones home, filtros |
| `cache/` | `cache_section.dart` (115) | Borrar historial / guardados / caché servidores / todo |
| `account/` | `account_settings_page.dart` (301) | Cambiar password, generar/vincular código TV |
| `ai/` | `ai_usage_section.dart` (303) | Cuota de IA (200/h) |
| `appearance/` | `appearance_section.dart` (108) | Cambiar modo móvil ↔ TV, posición del menú |
| `addons/` | `tv_addons_tab.dart` (177) | Acceso al hub de addons desde TV |
| `tvchanel/` | `tv_chanel_tab.dart` (223) | Listas M3U8 y addons de canales |
| `supabase/` | `supabase_section.dart` (96) | Estado de sesión y perfiles |

Raíz: `settings_page.dart` (1321, móvil), `tv_settings.dart` (356),
`tv_settings_page.dart` (329), `config_shared.dart` (48),
`tv_config_shared.dart` (1146 — 8 widgets de card con foco D-pad).

---

## `lib/presentation/` — 3 ficheros, ~5.300 líneas

| Fichero | Líneas | Contenido |
|---|---|---|
| `mobile/mobile_shell.dart` | 2023 | Shell móvil: 5 secciones, `IndexedStack`, bottom nav flotante, **fuerza vertical** en runtime |
| `tv/tv_shell.dart` | 2600 | Shell TV: 9 secciones, rail lateral O barra superior, **9 FocusNode de menú**, máquina de transferencia de foco |
| `shared/modals/playback_setup_modal.dart` | 677 | Modal inicial de configuración de reproducción (obligatorio en TV) |

---

## `lib/supabase/` — 8 ficheros, 1.127 líneas

| Fichero | Líneas | Responsabilidad |
|---|---|---|
| `supabase_constants.dart` | 15 | URL + anon key (**vacías**) + `isSupabaseConfigured` |
| `supabase_client.dart` | 49 | `AppSupabase.init()`, `currentUser`, `signOut()` |
| `supabase_config.dart` | 153 | Sesión de perfil, modo invitado, caché local de perfiles |
| `supabase_auth.dart` | 137 | signUp, signIn, signOut, resetPassword, updatePassword |
| `supabase_profiles.dart` | 186 | CRUD de perfiles (máx 5), selección con PIN, settings JSON |
| `supabase_data.dart` | 391 | Guardados / Historial / Likes con sync Supabase ↔ local |
| `guardados_service.dart` | 70 | Fachada legacy sobre `SupabaseData` |
| `supabase_tv_link.dart` | 126 | Pairing TV↔móvil con código de 6 dígitos |

→ Detalle en [09-supabase.md](09-supabase.md)

---

## `lib/docs/INSTRUCCIONES.md` (165 líneas)

Único documento de arquitectura **del autor original**. Describe el sistema de
cuentas/perfiles y los pasos para configurar Supabase. Contiene tablas de datos
que **no coinciden al 100 %** con el código actual (el código añade columnas
`updated_at`, `titulo`, `backdrop`, `duration_seconds` a `historial` que no
están documentadas ahí).

⚠️ Referencia a `sql/01_tablas_completas.sql`, un archivo **que no está en el
repo**. Hay que recrearlo (ver [09-supabase.md](09-supabase.md)).

---

## Carpetas raíz no-Lib

| Carpeta | Estado |
|---|---|
| `android/` | Target real. Kotlin, Gradle KTS, manifest, 3 MethodChannels |
| `assets/` | Logos de proveedores, bundless JS, `spiner.gif`, `example_source.json` |
| `test/` | ⚠️ **Un solo archivo de 0 bytes**: `widget_test.dart` |
| `web/`, `windows/` | ⚠️ Scaffolding por defecto de Flutter. **No compilarían** (el código usa `dart:io`) |
| `.idea/`, `.vscode/` | Config de IDE (`.idea/libraries/` con XML de librerías) |
| `lol2.iml` | Módulo IntelliJ residual |
| `android/hs_err_pid*.log` | ⚠️ **5 logs de crash de JVM commiteados** (basura) |

### `assets/images/` — logos de servicios
`imdb.png`, `metacritic.png`, `rotten tomatoes.png`, `letterboxd.png`,
`tmdb.png`, `vod.png`, `tvlive.png`, `vod.png` + `proveedores/` con Netflix,
Prime Video, Disney+, Crunchyroll, Paramount, Apple TV, MUBI, HBO Max.

### `assets/redes/` — redes sociales
`telegram.png`, `tiktok.png`, `instagram.png`, `github.png`