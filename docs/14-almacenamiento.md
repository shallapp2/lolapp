# 14 · Almacenamiento y caché

## 1. ¿Qué base de datos usa?

**Ninguna.** Verificado: no hay `sqflite`, `hive`, `isar`, `drift`, `sembast`,
`objectbox` ni `flutter_secure_storage` en `pubspec.yaml`.

### 1.1 `lib/core/storage/` — 4 ficheros, 21 líneas, 100 % muerto

```dart
// database.dart (4 líneas)
/// Placeholder for local database (Hive / SQLite) if needed later.
class AppDatabase { AppDatabase._(); }

// secure_storage.dart (4 líneas)
/// Placeholder for flutter_secure_storage when needed.
class SecureStorage { SecureStorage._(); }

// cache_manager.dart (6 líneas)
class CacheManager {                             // Map en memoria, sin uso real
  static final Map<String, dynamic> _mem = {};
  static T? get<T>(String key) => _mem[key] as T?;
  static void set(String key, dynamic value) => _mem[key] = value;
  static void clear() => _mem.clear();
}

// local_storage.dart (8 líneas)
class LocalStorage {                             // wrapper de SharedPreferences
  static Future<SharedPreferences> get prefs => SharedPreferences.getInstance();
  static Future<String?> getString(String key) ...
  static Future<bool> setString(String key, String v) ...
  static Future<bool> remove(String key) ...
}
```

Todo el código real llama a `SharedPreferences.getInstance()` directamente desde
**~70 ficheros**.

---

## 2. Las 3 capas de persistencia reales

### 2.1 `SharedPreferences` (XML en Android)

Todo: progreso, preferencias, caché de servidores, subtítulos, favoritos,
addons, tema. **Sin cifrado.**

⚠️ Límite práctico: ~1 MB en Android. Por eso `StorageService` poda
`rawCode` a 80.000 caracteres.

### 2.2 Supabase (PostgreSQL)

Ver [09-supabase.md](09-supabase.md).

| Tabla | Contenido |
|---|---|
| `profiles` | Perfiles de la cuenta (máx 5), PIN, settings JSON |
| `historial` | Progreso al segundo exacto, temporada, episodio |
| `guardados` | `tmdb_id`, poster, tipo |
| `likes` | `tmdb_id`, poster, tipo |
| `tv_link` | Código de 6 dígitos, expira 15 min |

### 2.3 Sistema de archivos

| Ruta | Contenido |
|---|---|
| `Documents/<addon_id>/` | `manifest.json`, `index.json`, `config.json`, `logo.png` |
| `Documents/tvchanel_addons/<addon>/` | Addons de canales |
| `getApplicationDocumentsDirectory()` + subcarpeta | Descargas de vídeo |

> Los subtítulos **no** van a disco: se cachean en `SharedPreferences`
> (`os_sub_<id>_<subId>`).

---

## 3. Las 15 cachés

Todas en `SharedPreferences`:

| Caché | Key / prefijo | TTL | Ubicación |
|---|---|---|---|
| **Progreso player (rápida)** | `cachePlayerRapido_<id>[_T.._C..]` | — | `player_page.dart:320` |
| **Progreso player (completa)** | `cachePlayer_<id>[_T.._C..]` | — | `player_page.dart:311` |
| **Servidor óptimo (Caché A)** | `srv_opt_<id>_<S>_<E>_<lang>` | **60 min** | `player_controller.dart:112` |
| **Lista servidores válidos (Caché B)** | `srv_valid_<...>` | 12 h (`FuentesCache.ttl()`) | `player_controller.dart:114` |
| **Lista servidores global** | `serv_cache_<id>_<tipo>_<S>_<E>` | 12 h configurable | `source_aggregator.dart:355` |
| **Último enlace** | `ultimo_enlace_<...>` | 12 h | `source_aggregator.dart:354` |
| **m3u8 por embed** | `m3u8_cache_v2_<id>_<tipo>_T<S>_C<E>` (timestamp **por entrada**) | **1 h** | `servers_modal.dart:89` |
| **Preloader de servidores** | `serv_cache_<id>_<tipo>_<T>_<C>` | **2 días** | `server_preloader.dart:6` |
| **Método de extracción por host** | `extractor_host_methods` (map `host → método`) | — | `extractor.dart:780` |
| **Verificación de URL** | En memoria (`_VerifyGate._cache`) | Sesión | `source_aggregator.dart:581` |
| **Catálogo de modelos IA** | En memoria | **6 h** | `ai_client.dart:38` |
| **Logos de TMDB** | En memoria de la instancia del servicio | Sesión | `tmdb_home_api.dart:76` |
| **"Para ti"** | `for_you_recs_v1` | **20 h** | `for_you_generator.dart:78` |
| **Secciones diarias** | `daily_home_sections_v1` | **24 h** | `daily_sections_generator.dart` |
| **Top 10 regional** | `regional_top10_ai_v2` + `regional_top10_ts_v2` | **24 h** | `regional_top10.dart` |
| **Eventos de IA** | `ai_usage_events_v1` | 7 días / 500 máx | `ai_usage_tracker.dart:70` |
| **Home** | Estático en memoria (`_cachedData`) | Sesión | `home_page.dart:40` |
| **Favoritos TV** | `tvchanel_favorites_v1` | — | `home_tvchanel_tv.dart` |
| **Listas M3U** | `tvchanel_m3u_playlists` | — | `tvchanel_repository.dart:12` |
| **Subtítulos OpenSubtitles** | `os_sub_<id>_<subId>` | — | `subtitle_selector.dart` |
| **Prefs de fuentes** | `servidores_cache_ttl_hours` | Default 12 h | `source_aggregator.dart:356-365` |
| **Addons** | `addons_v1`, `source_packages_v1`, `addon_settings_v1` | — | `addons/services/storage_service.dart` |
| **Stremio** | `stremio_catalog_addons_v1`, `stremio_collections_v1`, `stream_badge_packs_v1` | — | `stremio/*.dart` |
| **Perfil de gusto** | `user_taste_profile_v1`, `user_likes_v1` | — | `user_taste_profile.dart` |
| **Historial local (invitado)** | `local_historial`, `local_guardados`, `local_likes` | Top 100 | `supabase_data.dart` |
| **Perfiles locales (cache)** | `local_profiles_cache` | — | `supabase_config.dart:13` |
| **Sesión de perfil** | `current_profile_id/name/avatar/backdrop`, `is_guest_mode` | — | `supabase_config.dart:9-14` |
| **Recordatorios de episodios** | `episode_reminders_v1` | — | `episode_reminder_service.dart` |
| **Memoria de modelos IA** | `kilo_model_memory_v1` | — | `model_memory.dart` |

---

## 4. Invalidación

| Acción | Método |
|---|---|
| Borrar caché del player | `ServerLoader.clearCache(CacheType.optimal \| validServers \| all)` (`player_controller.dart:305-316`) — itera `prefs.getKeys()` y borra por prefijo |
| Borrar caché de fuentes | `FuentesCache.clearFor(...)` (`source_aggregator.dart:518+`) |
| Borrar preloader | `ServidoresPreloader.clearCache()` (`server_preloader.dart:95`) |
| Borrar m3u8 cache | `TvM3u8Cache.clear()` (`servers_modal.dart:173`) |
| Borrar por prefijo (TV) | `_clearByPrefix(prefixes, msg)` (`tv_cache_tab.dart:51`) |
| "Borrar todo" | `settings_page.dart:664` |

UI: `features/settings/presentation/cache/cache_section.dart` (115 líneas) y
`tv_cache_tab.dart` (158 líneas).

---

## 5. Writing serialization: por qué importa

Cuando llegan varios `FuenteEvent` a la vez y todos escriben en la misma clave
de `SharedPreferences`, se produce una carrera (lectura-modificación-escritura).
`TvM3u8Cache` lo resuelve serializando con una cadena de futures:

```dart
// servers_modal.dart:69-81
static Future<void> _writeChain = Future.value();
static void _enqueueWrite(Future<void> Function() op) {
  _writeChain = _writeChain.then((_) => op());
}
```

---

## 6. `core/constants/storage_keys.dart` — no se usa

```dart
// storage_keys.dart (6 líneas)
class StorageKeys {
  static const String appMode            = 'app_mode';
  static const String disclaimerAccepted = 'disclaimer_accepted';
  static const String favorites          = 'favorites';
  static const String watchHistory       = 'watch_history';
}
```

**No se importa en ningún sitio.** Las keys reales son **strings hardcodeados
dispersos por 70+ ficheros**. Además, `app_mode` y `disclaimer_accepted` están
**triplicados**: `main.dart:25-26`, `app_config.dart:4-5` y aquí.

---

## 7. Gestión de cookies

**No existe.** No hay cookie jar, ni `CookieInterceptor` de Dio, ni parsing de
`Set-Cookie`.

Consecuencia: los CDNs que exigen cookies (o que las emiten para validar) **fallan
siempre**. Es la limitación más probable que explica streams que "se verifican"
pero luego dan 403 al reproducir.

---

## 8. `lib/core/network/` — 100 % muerto

| Fichero | Contenido |
|---|---|
| `http_client.dart` (6) | `class AppHttpClient { AppHttpClient._(); }` + comentario *"implementations still use Dio / http directly inside scrapers and extractors"* |
| `http_interceptor.dart` (2) | `class HttpInterceptor {}` |
| `http_headers.dart` (7) | `HttpHeadersHelper.defaultHeaders()` — el único con contenido real, y no se usa |
| `network_checker.dart` (11) | `hasConnection()` con `InternetAddress.lookup('google.com')` — **el shell móvil reimplementa esto en línea** (`mobile_shell.dart:637-646` con `example.com`) |
| `request_cache.dart` (7) | `Map` estático, sin uso |

**No hay capa de red centralizada.** Cada scraper/extractor hace su propio
`http.get` con sus propios headers y timeouts.

`dio` (`pubspec.yaml:18`) **solo se usa en `downloads/download_manager.dart`**.
El resto de la app usa `http`.

---

## 9. Headers reales que sí importan

| Contexto | Headers | Ubicación |
|---|---|---|
| **Player** | UA Chrome 124 + `Referer`/`Origin`/`Range` por host + canonicalización a mayúsculas | `player_page.dart:668-728` |
| **IPTV** | UA parametrizado (`okhttp/4.12.0`, `VLC/3.0.16`, Chrome Android, `Lavf/58.76.100`) + `Connection: keep-alive` + `Accept: application/vnd.apple.mpegurl` para `.m3u8` | `player_tvchanel.dart:67-90` |
| **M3U remoto** | UA Chrome + `Accept: */*` | `tvchanel_repository.dart:77-84` |
| **GitHub addons** | `Accept: application/vnd.github+json` + `User-Agent: LolApp1-Addons/1.0` | `community_service.dart:51-61` |
| **GitHub raw** | `User-Agent: LolApp-TvChanel/1.0` | `tvchanel_repository.dart:274-279` |
| **IA Kilo** | `Content-Type: application/json; charset=utf-8` + `Accept: text/event-stream` — y explícitamente **NUNCA** `Authorization` | `ai_client.dart:95-97` |
| **Custom API** | `Accept: application/json` + `Referer: https://modlyo.com/` | `custom_api.dart:227-233` |
| **Scrapers** | UA Chrome 120-154 según el fichero + `Accept-Language: es-MX` o `es-ES` | `base_home_scraper.dart:62-79` |

---

## 10. Límites y riesgos

| Riesgo | Detalle |
|---|---|
| **Límite de tamaño** | `SharedPreferences` en Android ≈ 1 MB. Con 325 vídeos descargados en caché + addons grandes, se puede agotar |
| **Podar `rawCode`** | `StorageService` corta el JS a 80.000 caracteres → hay que re-descargarlo en cada arranque |
| **Sin cifrado** | Todo en XML plano: progreso de visualización, historial, PIN de perfil, credenciales de API personalizadas |
| **Sin transacciones** | Los upsert son punto a punto; un crash a mitad puede dejar datos inconsistentes |
| **Colisiones de claves** | ~70 ficheros con strings hardcodeados, sin namespacing central |
| **Sin migración de esquema** | Renombrar una clave = perder todos los datos de esa feature |