# 01 · Arquitectura general

## TL;DR

El proyecto **parece** seguir *Clean Architecture* (carpetas `core/`, `data/`,
`domain/`, `features/`, `presentation/`), pero **ese andamiaje está vacío**.
Lo que realmente gobierna la app son `main.dart`, dos *shells* gigantes
(`mobile_shell.dart` y `tv_shell.dart`) y ~20 singletons estáticos.

---

## 1. Arquitectura declarada vs. arquitectura real

### 1.1 Lo que existe en disco

```
lib/
├── main.dart                     679 líneas  ← bootstrap + Splash  (REAL)
├── app/                            6 ficheros, 46 líneas  ← SCAFFOLDING
├── core/                          35 ficheros, 1053 líneas ← CASCA (80 %)
│   ├── constants/                 ← tmdb_apis.dart, sources.dart, versiones.dart USADOS
│   ├── errors/                    ← 5 clases, 0 usos
│   ├── network/                   ← 5 clases, 0 usos
│   ├── result/                    ← Result/Success/Failure, 0 usos
│   ├── services/                  ← session_watch_timer.dart USADO
│   ├── storage/                   ← 4 clases, 0 usos
│   └── utils/                     ← display_refresh.dart USADO; 10 más, 0 usos
├── data/                          85 ficheros, ~29.000 líneas ← REAL (la lógica)
├── domain/                        44 ficheros, 237 líneas ← CASCA TOTAL
│   ├── models/                    38 POJOs, nadie los importa
│   ├── repositories/              9 clases abstract vacías
│   └── services/                  6 clases abstract vacías
├── features/                      143 ficheros, ~116.000 líneas ← REAL (78 % del código)
├── presentation/                  3 ficheros, ~5300 líneas ← REAL (los shells)
└── supabase/                      8 ficheros, 1127 líneas ← REAL
```

### 1.2 Verificación

Las siguientes afirmaciones se comprobaron buscando imports en todo `lib/`:

| Capa | ¿Se importa? | Detalle |
|---|---|---|
| `lib/app/` | ❌ No | Salvo `app_theme.dart → app_constants.dart`. `app.dart` importa `main.dart` (import inverso) |
| `lib/domain/**` | ❌ **Nunca** | 0 resultados de `domain/repositories\|domain/services\|domain/models` fuera de `domain/` |
| `lib/core/result/` | ❌ No | `Result<` solo aparece en sus propias 15 líneas |
| `lib/core/errors/` | ❌ No | 0 `AppException`/`NetworkException`/... fuera de sus ficheros |
| `lib/core/network/` | ❌ No | `http_client.dart` dice literalmente: *"Current implementations still use Dio / http directly inside scrapers"* |
| `lib/core/storage/` | ❌ No | `AppDatabase` es `class AppDatabase { AppDatabase._(); }` |
| `lib/core/constants/tmdb_apis.dart` | ✅ Sí | 12 usos |
| `lib/core/constants/sources.dart` | ✅ Sí | 4 usos (solo UI de Ajustes) |
| `lib/core/constants/versiones.dart` | ✅ Sí | 6 usos |
| `lib/core/services/session_watch_timer.dart` | ✅ Sí | 2 usos |
| `lib/core/utils/display_refresh.dart` | ✅ Sí | 2 usos |

**≈ 4.800 líneas de código muerto.**

### 1.3 Dependencias declaradas sin usar

| Paquete | pubspec | ¿Importado? |
|---|---|---|
| `provider` | `:15` | ❌ 0 imports de `package:provider` en todo `lib/` |
| `go_router` | `:19` | ❌ No hay ni un `GoRouter` ni un `GoRoute` |
| `media_kit` / `_video` / `_libs_video` | `:23-25` | ❌ 0 imports |
| `fvp` | `:35` | ❌ 0 imports |
| `chewie` | `:36` | ❌ 0 imports (toda la UI de controles es custom) |
| `libtorrent_flutter` | `:37` | ❌ 0 imports (torrent/MKV se detecta y **filtra**, nunca se reproduce) |
| `permission_handler` | `:45` | ❌ |
| `intl` | `:29` | ❌ |
| `json_annotation` | `:30` | ❌ |
| `equatable` | `:30` | ❌ |
| `flutter_svg` | `:32` | ❌ |

Paquetes **sí** usados: `http`, `dio` (solo descargas), `shared_preferences`,
`path_provider`, `cached_network_image`, `video_player`, `webview_flutter`,
`flutter_js`, `uuid`, `url_launcher`, `collection`, `encrypt`, `crypto`
(parcial), `open_filex`, `qr_flutter`, `flutter_local_notifications`,
`dart_cast`, `supabase_flutter`, `timezone`, `flutter_foreground_task`,
`device_info_plus`, `package_info_plus`, `wakelock_plus`, `flutter_js`,
`android_intent_plus`, `html`, `cupertino_icons`.

---

## 2. Arquitectura real

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            CAPA DE UI (100% del negocio)                 │
│                                                                          │
│  main.dart  ──►  SplashScreen  ──►  mobile.MainHome | tv.MainHome        │
│                                        │                                 │
│                          ┌─────────────┴──────────────┐                  │
│                          ▼                            ▼                  │
│              mobile_shell.dart (2023 L)      tv_shell.dart (2600 L)       │
│              5 secciones, bottom nav        9 secciones, rail/top + D-pad│
│                          │                            │                  │
│                          └─────────────┬──────────────┘                  │
│                                        ▼                                 │
│                    StatefulWidgets gigantes por feature                  │
│      (player_page 2930 L · tv_player_page 4505 L · content_page 3037 L)  │
│                                                                          │
│  Sin capa de providers. Sin router. Sin BLoC. DI = singletons estáticos. │
└──────────────────────────────────────────────────────────────────────────┘
              │            │             │              │            │
              ▼            ▼             ▼              ▼            ▼
   ┌────────────┐ ┌─────────────┐ ┌────────────┐ ┌────────────┐ ┌───────────┐
   │ data/      │ │ data/       │ │ data/      │ │ data/      │ │supabase/  │
   │ addons/    │ │ extractors/ │ │ scrapers/  │ │ tmdb/ + ai/│ │ (cuenta,  │
   │ (JS+Stremio│ │ (13 prov.)  │ │ (5 sitios) │ │            │ │ perfiles, │
   │  +Nuvio)   │ │             │ │            │ │            │ │  sync)    │
   └────────────┘ └─────────────┘ └────────────┘ └────────────┘ └───────────┘
              │            │             │              │
              └────────────┴─────────────┴──────────────┘
                                     ▼
                    ┌────────────────────────────────┐
                    │ SharedPreferences + HTTP +      │
                    │ Filesystem (Documents/) +       │
                    │ WebView + MethodChannel         │
                    └────────────────────────────────┘
```

---

## 3. Gestión de dependencias: singletons estáticos

No hay `provider`, ni `get_it`, ni `InheritedWidget`, ni Riverpod/Bloc/GetX.
El mecanismo real es `static final X instance = X._()` (patrón *singleton*).

### 3.1 Los ~20 singletons

| Singleton | Fichero | Tipo | Responsabilidad |
|---|---|---|---|
| `AddonManager.instance` | `data/addons/addon_manager.dart:18` | `ChangeNotifier` | Addons JS + Stremio + Nuvio |
| `AddonSourceAdapter.instance` | `data/addons/addon_source_adapter.dart:14` | — | Puente addons → modal de servidores |
| `JsAddonRuntime.instance` | `data/addons/services/js_runtime.dart:15` | — | Runtime JS (QuickJS) |
| `CatalogJsBridge.instance` | `data/addons/services/catalog_js_bridge.dart:7` | — | Puente JS ↔ catálogos |
| `StremioAddonRepository.instance` | `data/addons/stremio/stremio_addon_repository.dart:13` | `ChangeNotifier` | Catálogos Stremio |
| `StremioCollectionRepository.instance` | `.../stremio_collection_repository.dart:15` | `ChangeNotifier` | Colecciones importadas |
| `StreamBadgeRepository.instance` | `.../stream_badge_repository.dart:33` | `ChangeNotifier` | Badges/logos de streams |
| `AppSupabase` | `supabase/supabase_client.dart` | static class | Cliente Supabase (PKCE) |
| `SupabaseConfig` / `SupabaseAuth` / `SupabaseData` / `SupabaseProfiles` / `SupabaseTvLink` | `supabase/*.dart` | static classes | Sesión, guardados, perfiles, pairing TV |
| `DownloadManager.instance` | `features/downloads/presentation/download_manager.dart:123` | `ChangeNotifier` | Descargas HLS segmentadas + foreground service |
| `MiniPlayerService.instance` | `features/player/presentation/widgets/mini_player_service.dart:14` | `ChangeNotifier` | Mini-player global (handoff de controller) |
| `CastManager()` | `.../cast_manager.dart:9` | factory singleton | Sesión Chromecast + notificación persistente |
| `ServidoresPreloaderService.instance` | `features/servers/presentation/server_preloader_service.dart:32` | — | Precarga silenciosa de servidores |
| `SessionWatchTimer.instance` | `core/services/session_watch_timer.dart:9` | — | Aviso a las 4 h de uso |
| `UpdateNotificationService.instance` | `features/settings/presentation/updates/...:17` | — | Notificación de actualización |
| `EpisodeReminderService.instance` | `.../notifications/episode_reminder_service.dart:68` | — | Recordatorios de estreno |
| `AiUsageTracker.instance` | `data/ai/ai_usage_tracker.dart:74` | — | Cuota IA (200 req/h) |
| `TvChanelRepository.instance` | `features/tvchanel/data/tvchanel_repository.dart:17` | — | Listas M3U + addons de canales |
| `VersionService` | `core/constants/versiones.dart:4` | static class | Semver + `version_code` |

### 3.2 Patrón de consumo

Como no hay `context.watch<T>()`, hay tres patrones repetidos masivamente:

```dart
// 1) Instancia global leída directamente
final _dm = DownloadManager.instance;
_dm.addListener(_onDm);   // initState

// 2) setState manual como respuesta al ChangeNotifier
void _onMiniPlayer() { if (mounted) setState(() {}); }
MiniPlayerService.instance.addListener(_onMiniPlayer);
// ...y el removeListener en dispose

// 3) AnimatedBuilder para sub-widgets reactivos
AnimatedBuilder(
  animation: MiniPlayerService.instance,
  builder: (context, _) => _MiniPlayerBar(),
)
```

⚠️ **Riesgo**: alta probabilidad de *listeners* duplicados o fugas, porque
cada pantalla se registra a mano y es fácil olvidar el `removeListener`.

### 3.3 Buses de eventos informales

Como no hay event bus, hay `ValueNotifier<int>` estáticos:

```dart
class DownloadNavBus {                 // mobile_shell.dart:36
  static final ValueNotifier<int> version = ValueNotifier<int>(0);
  static void bump() => version.value++;
}
```

Otros: `MenuPositionPref.version`, `GuardadosBus.version`,
`HomeAlgorithmBus.version`, `TasteOnboardingGate`.

---

## 4. Navegación

**No hay router.** `MaterialApp` se crea sin `routes:`, sin `onGenerateRoute:`
y sin `routerConfig:` (`main.dart:80-89`). Hay ~99 llamadas a
`Navigator.push` / `pushReplacement` / `pushAndRemoveUntil`.

```dart
// main.dart:80-89
return MaterialApp(
  title: 'lolplustv',
  debugShowCheckedModeBanner: false,
  theme: ThemeData(brightness: Brightness.dark, useMaterial3: true,
                   scaffoldBackgroundColor: Colors.black),
  home: const SplashScreen(),
);
```

Dos hechos importantes:

1. **El tema está hardcodeado dark.** No existe `ThemeMode` en el repo
   (`rg ThemeMode` → 0 resultados). El ajuste de "Apariencia" solo cambia
   entre shell móvil ↔ shell TV.
2. **Los deep links nativos están muertos.** `MainActivity.kt:168-213` recibe
   intents `lol://user/<id>`, `lol://content/<id>`, `lol://list/<filter>` y
   llama `invokeMethod("openDeepLink")`, pero **no hay ningún
   `setMethodCallHandler` con ese nombre en Dart**.

Detalle completo en [03-arranque-y-navegacion.md](03-arranque-y-navegacion.md).

---

## 5. Manejo de errores

### 5.1 Diseñado pero no implementado

```dart
// lib/core/result/result.dart
sealed class Result<T> {
  bool get isSuccess => this is Success<T>;
  bool get isFailure => this is Failure<T>;
  T? get dataOrNull => this is Success<T> ? (this as Success<T>).data : null;
}

// lib/core/errors/app_exception.dart
class AppException implements Exception { ... }
//   ├── NetworkException
//   ├── ScraperException
//   ├── ExtractorException
//   └── PlayerException
```

Bien diseñado, **0 consumidores**. Además `result.dart` y `success.dart`
contienen el mismo código literal (15 líneas duplicadas), y `failure.dart` solo
hace `export 'result.dart'`.

### 5.2 Lo que se usa de verdad

- ~200 ocurrencias de `try { } catch (_) { }` **silencioso**
- `try { } catch (e) { debugPrint(...) }`
- Campos de estado `String? _error` + re-render

### 5.3 El único patrón funcional de errores

Está en `data/ai/model_memory.dart:111-127` y sí se usa con *pattern matching*:

```dart
sealed class ModelFailure {
  factory ModelFailure.rateLimited([int? retryAfterMs]) => RateLimited(retryAfterMs);
  static const server = ServerFailure();
  static const unreadable = UnreadableFailure();
}
```

Y *gates* con ventana rodante en `data/recommendations/for_you_generator.dart:67`:

```dart
static const windowAfterFailureMs = 15 * 60 * 1000;  // 15 min de espera tras fallar
static bool isDue(int lastAttemptMs, bool lastWasModelFailure, int nowMs) => ...;
```

---

## 6. Sistema de diseño

No hay tema centralizado. La paleta está **duplicada en cada feature**:

| Constante | Valor | Ubicaciones |
|---|---|---|
| Rojo acento | `0xFFE50914` | `app_constants.dart:5`, `config_shared.dart:4`, `mobile_shell.dart:34`, `tv_shell.dart:35`, `pag.dart:11`, `history_page.dart:13`, `favorites_page.dart`… |
| Morado acento | `0xFF7B5CFF` | `app_constants.dart:6` (**nunca usada**), `tv_shell.dart:36` (**declarada y sin usar** → warning) |
| Morado alternativo | `Colors.purpleAccent` | `home_page.dart:19`, `favorites_page.dart:17` (¡inconsistente con el rojo del resto!) |
| Card bg | `0xFF1C1C1E` / `0xFF1a1a2e` | **varía entre features** |

---

## 7. Qué está bien diseñado

1. **El registro de fuentes** (`data/scrapers/base/registry.dart`) es un punto
   de extensión limpio: añadir una fuente = implementar el scraper + 1 entrada
   en `fuentesRegistry`. `pag.dart` lo consume automáticamente.
2. **La máquina de foco D-pad** del shell TV (`tv_shell.dart:650-703`) es
   madura: 12 reintentos × 40 ms para transferir el foco al contenido, con
   *fallback* al menú.
3. **El arranque es explícito y legible**: todas las inicializaciones antes de
   `runApp`, y degradación silenciosa si Supabase falla.
4. **La verificación de servidores con semáforo** (`_VerifyGate`) limita los
   WebViews concurrentes y cachea resultados por URL.
5. **El handoff del `VideoPlayerController`** al mini-player evita el clásico
   "MediaCodec Decoder init failed" al minimizar.