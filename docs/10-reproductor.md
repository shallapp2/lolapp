# 10 · Reproductor

El subsistema más grande: **28 ficheros, ~12.200 líneas** solo de player.

---

## 1. Inventario

| Fichero | Líneas | Rol |
|---|---|---|
| `presentation/tv/tv_player_page.dart` | **4505** | `PlayerScreen` TV (el mayor del repo) |
| `presentation/player_page.dart` | 2930 | `PlayerScreen` móvil (el principal) |
| `presentation/tv/tv_discover_player.dart` | 2943 | Player TV para "Descubrir" (usa `ServidoresModalFuentesTv`) |
| `presentation/player_controller.dart` | 776 | `ServerLoader` móvil |
| `presentation/tv/tv_player_controller.dart` | 1028 | `ServerLoader` TV (verificación concurrente) |
| `presentation/tv/tv_player_webview.dart` | 1821 | WebView player TV |
| `presentation/subtitles/subtitle_selector.dart` | 1758 | `OpenSubtitlesModal` + `SubtitleUtils` |
| `presentation/subtitles/tv_subtitle_selector.dart` | 2055 | Versión TV (D-pad) |
| `presentation/quality/quality_selector.dart` | 690 | `HlsQualityParser` + `HlsQualitySelectorModal` |
| `presentation/tv/tv_quality_selector.dart` | 690 | Quality selector TV |
| `data/extractor.dart` | 1325 | `NativeResolvers` + `ExtractorPage` (WebView visible) |
| `data/extractor_mobil.dart` | 1325 | ⚠️ **byte-idéntico** a `extractor.dart` |
| `data/native_resolvers.dart` | 737 | Copia standalone de los resolvers |
| `presentation/web_player_view.dart` | 568 | WebView player móvil |
| `presentation/subtitles/subtitle_widget.dart` | 212 | Render de subtítulos |
| `presentation/tv/tv_subtitle_widget.dart` | 250 | Render de subtítulos TV |
| `widgets/cast_button.dart` | 881 | `CastButton` + diálogos |
| `widgets/cast_manager.dart` | 436 | `CastManager` + `CastTaskHandler` |
| `widgets/cast_screen.dart` | 646 | Control remoto Chromecast |
| `widgets/mini_player_service.dart` | 367 | `MiniPlayerService` |
| `widgets/skip_button.dart` | 784 | `SkipSegment`, `SkipSegmentsData`, `SkipSegmentsLoader` |
| `widgets/next_episode_prompt.dart` | 328 | Tarjeta "siguiente episodio" |
| `widgets/mobile_skip_next_overlay.dart` | 205 | Botón skip móvil |
| `widgets/screensaver_overlay.dart` | 158 | Screensaver (TV) |
| `widgets/because_you_watched_overlay.dart` | 486 | Overlay "porque viste X" |
| `widgets/watch_later_button.dart` | 226 | `VerDespuesList` |
| `presentation/tv/server_loader.dart` | **0** | ⚠️ **Archivo vacío** |
| `presentation/tv/tv_extractor_screen.dart` | **0** | ⚠️ **Archivo vacío** |

---

## 2. ¿Qué motor de video se usa?

| Librería | pubspec | ¿Usado? | Para qué |
|---|---|---|---|
| **`video_player` 2.9.2** | `:34` | ✅ **ÚNICO motor real** | ExoPlayer en Android. Usado en `player_page.dart:6`, `tv_player_page.dart:6`, `tv_discover_player.dart:7`, `player_tvchanel.dart:6`, `mini_player_service.dart:5`, descargas, etc. |
| `webview_flutter` (+`_android`) | `:39-40` | ✅ | Player fallback para embeds: `web_player_view.dart:7`, `tv_player_webview.dart:6`, `extractor.dart` |
| `dart_cast` 0.7.4 | `:48` | ✅ | Chromecast |
| `wakelock_plus` | `:42` | ✅ | `WakelockPlus.enable()` en cada player |
| `cached_network_image` | `:22` | ✅ | Backdrops/posters en el player |
| `media_kit` / `_video` / `_libs_video` | `:23-25` | ❌ | 0 imports |
| `fvp` 0.38.1 | `:35` | ❌ | 0 imports |
| `chewie` 1.8.5 | `:36` | ❌ | 0 imports — toda la UI de controles es **custom** |
| `libtorrent_flutter` 2.0.0 | `:37` | ❌ | 0 imports — torrent/MKV se detecta y **filtra**, nunca se reproduce |

> **Conclusión**: el motor es **`video_player` (ExoPlayer) + `webview_flutter`
> como fallback**. Las otras 4 librerías son peso muerto (~15-25 MB de APK).

---

## 3. Flujo completo: de la fuente al player

### 3.1 Fase A — Origen de la URL (`player_page.dart:918 _initializePlayer`)

Hay **5 caminos**:

```
_initializePlayer()  [player_page.dart:918-1055]
│
├─ (1) URL RECIBIDA EN EL WIDGET → play inmediato
│     if (widget.videoUrl.trim().isNotEmpty)                [:930]
│       → _activeUrl = passed; headers = widget.headers      [:932-933]
│       → unawaited(_warmFallbackServers())   ← en 2º plano   [:937]
│       → _startControllerWithUrl(passed, headers)           [:938]
│
├─ (2) SIN URL → lista de servidores
│     _serverLoader.getServers(...)  → ServerLoader           [:945]
│     → _sortServersByLangThenPlayable(servers)              [:956]
│
├─ (3) PRIMER SERVIDOR "APTO" (m3u8/mp4 ya resuelto) → play  [:961-981]
│     for i in servers: if (_serverIsPlayableNow(s)) → start
│
├─ (4) RESOLUCIÓN PARALELA en lotes (1..5)                    [:984-1012]
│     concurrent = prefs('servidores_concurrent_checks') ?? 2
│     _resolveServersConcurrent(servers, limit)               [:865-916]
│       → por cada lote: Future.wait(batch) → mejor índice bajo
│         → tryResolveServer(s) → MainFuentes.verificarConExtractor
│
└─ (5) ÚLTIMO RECURSO: _serverLoader.resolvePlayable(...)     [:1015-1035]
      → si null → _allServersFailed = true                   [:1038-1043]
```

### 3.2 Fase B — `ServerLoader` (el "cerebro" de resolución)

`player_controller.dart:104-776` (móvil) / `tv_player_controller.dart:105-1028` (TV).

#### Preferencias (`ServerLoaderPrefs`, `player_controller.dart:10-81`)

| Pref | Valores | Default |
|---|---|---|
| `idioma_audio_predeterminado` | `latino` \| `castellano` \| `subtitulado` | — |
| `subtitulo_predeterminado` | `es` \| ... | `es` |
| `seleccionar_servidores` | `auto` \| `manual` | `auto` |
| `idiomaFallbackOrder` | lista ordenada | p.ej. latino → `['es_MX','es_ES','en_US']` (`:70-80`) |

Mapeo de idioma (`:10-81`): `latino → es_MX`, `castellano → es_ES`,
`subtitulado → en_US`.

#### Las 3 cachés

| Caché | Clave | TTL | Líneas |
|---|---|---|---|
| **A — servidor óptimo** | `srv_opt_<id>_<S>_<E>_<lang>` | **60 min** | `:112-114`, `:688-739` |
| **B — lista de servidores válidos** | `srv_valid_<...>` | `FuentesCache.ttl()` (**12 h**) | `:114`, `:741-775` |
| **Global** | `serv_cache_<id>_<tipo>_<S>_<E>` + `ultimo_enlace_<...>` | 12 h configurable | `source_aggregator.dart:352-545` |

`clearCache(CacheType.optimal|validServers|all)` (`:305-316`): itera
`prefs.getKeys()` y borra por prefijo.

#### `resolvePlayable()` (`:195-248`) — orden de intento

1. **Caché A** (`:216-222`) → si hit, retorno inmediato
2. **`_resolveFromCachedLists()`** (`:326-399`) → combina Caché B +
   `tryLoadCachedServers` + `tryReuseLastLink` (si `reutilizarUltimoEnlace`),
   ordena por idioma preferido y resuelve
3. **`_resolveProgressiveFirstWin()`** (`:401-522`) → **streaming first-win**:
   escucha `MainFuentes.fetchProgressive()` y, en cuanto llega un servidor del
   idioma preferido, lo intenta y **cancela el resto** (`finish()` →
   `sub?.cancel()`). **Timeout global 45 s** (`:515-521`). Intentos serializados
   con una cadena `chain` para no abrir N extractores a la vez

#### `tryResolveServer()` (`:594-670`)

1. Si ya tiene `resolved_m3u8` → directo
2. Detecta "directo" por heurística (`:620-626`): `.mp4`, `.m3u8`,
   `hakunaymatata.com`, `/bt/` + `.mp4`, `type=='direct'`, `provider=='netmirror'`
3. Si no → `MainFuentes.verificarConExtractor(context, url, timeout: 8s)` y
   escribe `resolved_m3u8` + `verificado=true`
4. Si falla → `markServerAsInvalid()` (`:250-255`, `Set` en memoria `_invalidUrls`)

#### Precarga del siguiente capítulo

`preloadNext()` (`:264-303`): respeta `capitulos_precarga` (default 2, clamp
0-10) y escribe en Caché A para que el siguiente capítulo arranque sin scrapear.

### 3.3 Fase C — Resolución real desde las fuentes

Ver [06-fuentes-y-agregadores.md](06-fuentes-y-agregadores.md) y
[05-extractores.md](05-extractores.md).

### 3.4 Fase D — Abrir el player

`_startControllerWithUrl()` (`player_page.dart:1085-1149`):

```dart
_controller = VideoPlayerController.networkUrl(Uri.parse(url), httpHeaders: h);  // :1111
await _controller.initialize();                                               // :1115
_controller.addListener(_videoListener);                                      // :1122
_startCacheTimer();                                                           // cada 30 s
final saved = await _getSavedPosition();                                      // :1126
if (saved != null && saved > 5) {
  await _controller.seekTo(Duration(seconds: saved));
}
await _controller.play();                                                     // :1138
// catch (:1140-1148): marca el server como inválido y llama _tryNextServer()
```

### 3.5 Fase E — Headers anti-hotlink ⭐

`_playerHeaders()` (`player_page.dart:668-728`) — **dos pasos críticos**.

**1. Base canónica:**

```
user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/124.0.0.0 Safari/537.36
accept: */*
accept-language: es-ES,es;q=0.9,en;q=0.8
```

+ `widget.headers` + `_activeHeaders` (del servidor/addon).

**2. Referer/Origin por host** (`:686-711`):

| Host | Referer / Origin / extra |
|---|---|
| `net27.cc`, `hakunaymatata.com`, `/bt/` | `referer: https://net27.cc/` + `origin: https://net27.cc` + **`range: bytes=0-`** |
| `voe.sx`, `voe-network` | `https://voe.sx/` |
| `mp4upload` | `https://www.mp4upload.com/` |
| `streamwish`, `streamvid` | `https://streamwish.to/` |
| `filemoon`, `moonmov` | `https://filemoon.sx/` |
| **cualquier otro** | fallback: `referer`/`origin` = `scheme://host` de la URL |

**3. Canonicalización** (`:713-727`): ExoPlayer exige `User-Agent`, `Referer`,
`Origin`, `Accept`, `Accept-Language`, `Range`, `Cookie` en **mayúsculas exactas**.

> TV: `_buildPlayHeaders()` (`tv_player_page.dart:1536-1588`) — misma lógica.
> IPTV en vivo: `_headersFor()` (`player_tvchanel.dart:67-90`) añade
> `Connection: keep-alive` y `Accept: application/vnd.apple.mpegurl` si `.m3u8`.

### 3.6 Fase F — Detección de URL directa

`_isDirectStreamUrl()` (`player_page.dart:731-765`) — heurística amplia:

```dart
// Extensiones
.m3u8 .mp4 .mpd .mkv .webm .ts .m4s .m4a
// Patrones HLS
/playlist  format=m3u8  master.m3u  /hls/  hls=1  /manifest  mime=video  video/mp4
// CDN
googlevideo.com  cloudfront.net  cdn.
// Torrent
/bt/
```

Ordenar por idioma **y** por "playable-now" (`:832-849`).

---

## 4. Gestión de calidad

### 4.1 `HlsQualityParser.parse(masterUrl)` (`quality_selector.dart:47-141`)

1. `http.get` del master m3u8 (timeout 8 s)
2. Parsea líneas `#EXT-X-STREAM-INF` con regex:
   - `RESOLUTION=(\d+)x(\d+)`
   - `BANDWIDTH=(\d+)`
   - `AVERAGE-BANDWIDTH`
3. Resuelve URL relativa con `baseUri.resolve()`
4. Label: `1080p` o `2.5 Mbps` o `kbps`
5. Deduplica por URL
6. **Ordena de mayor a menor** (height, luego bandwidth)
7. **Añade "Auto" (el master) al principio** (`:127-136`)

### 4.2 `HlsQualitySelectorModal` (`quality_selector.dart:253-676`)

- Dialog `maxWidth: 420 / maxHeight: 520`
- `FocusScope autofocus`
- Navegación D-pad completa (`:378-434`):
  - ↑↓ mueven foco + `ensureVisible`
  - ←→ callbacks
  - OK/Enter/Space seleccionar
  - Esc/GoBack cerrar
- Item focusable: blanco si focused, acento si selected
- **Footer con ayuda de teclas** (`:661-667`)

### 4.3 Cambio en caliente (`player_page.dart:1773-1882 _openQualitySelector`)

```dart
1. Toma _activeUrl (master) como referencia                              :1777-1791
2. Guarda savedPos = _controller.value.position                          :1812
3. dispose() del controller viejo                                       :1828-1838
4. Crea controller nuevo con playUrl + httpHeaders: _playerHeaders(master) :1843-1846
5. initialize() → seekTo(savedPos) si > 2 s                             :1857-1859
6. play()  ← NO pierde posición ni estado
7. _activeUrl se mantiene como master para futuros cambios               :1842
```

TV: `tv_quality_selector.dart:253-299` — `.show()` con
`useRootNavigator: true` (necesario para que el dialog tome foco en Android TV,
`:280-281`).

---

## 5. Subtítulos

### 5.1 Fuente A — Subtítulos internos (VTT del backend)

`_loadSubtitles()` (`player_page.dart:1199-1212`) → URL `apiData['subtitulo']`
o fallback:

```
https://modlyo.com/subtitulo/contenido/{tmdbId}/es_MX.vtt
```

Parser `_parseVtt()` (`:1214-1238`) con regex de timestamps:

```dart
RegExp(r'(\d{2}:)?(\d{2}):(\d{2})[.,](\d{3})\s*-->\s*')
```

Quita tags `<...>` (`:1256`).

### 5.2 Fuente B — OpenSubtitles (addon Stremio)

`OpenSubtitlesModal` (`subtitle_selector.dart:9-1702`)

- API: `https://opensubtitles-v3.strem.io/subtitles/series/<imdb>:<S>:<E>.json`
  o `/movie/<imdb>.json` (`:241-250`)
- Agrupa por `lang`, ordena por preferencia `['spa','eng','fre','ita','por','pob']`
  (`:279-288`)
- **60+ idiomas mapeados** (`:124-180`)
- **Sincronización manual** (`_openSyncWindow` `:356+`): ventana independiente con
  captura previa de todos los cues y callbacks, porque el State del modal se
  destruye al cerrar

`SubtitleUtils.downloadAndCache()` (`:1737-1758`): descarga con timeout 15 s,
cachea en `SharedPreferences` con key `os_sub_<id>_<subId>`, convierte
**SRT→VTT** si no empieza por `WEBVTT` (`srtToVtt()` `:1705-1735`).

### 5.3 Preferencias de estilo (`player_page.dart:100-111`)

| Pref | Rango | Default móvil | Default TV |
|---|---|---|---|
| `subtitulo_offset_sec` | −30 … +30 | 0 | 0 |
| `subtitulo_font_size` | 12 … 36 | 18 | **22** |
| `subtitulos_negrita` | bool | `true` | `false` |
| `subtitulo_vertical_offset` | −40 … +160 | 0 | 0 |
| `player_sub_id_<id>` / `player_sub_url_<id>` / `player_sub_lang_<id>` | string | — | — |

Render: `SubtitleWidget` (`subtitle_widget.dart:55`) con `bottomPadding`
dinámico (36 / 120 / 220 px según controles y panel).
TV: `tv_subtitle_widget.dart` (250) con `SubtitlePrefs` que carga desde
`subtitulo_tamano` (pequeño/mediano/grande), `subtitulos_negrita`,
`player_vertical_offset` (±80).

---

## 6. Progreso de reproducción — resume al segundo exacto

### 6.1 Los 3 almacenes

**(1) SharedPreferences (2 claves)**

```dart
// player_page.dart:311-318
_getCacheKey()        → 'cachePlayer_${tmdb}'                       (película)
                     →  'cachePlayer_${tmdb}_T${t}_C${c}'           (serie)
// :320-327
_getCacheKeyRapido()  → 'cachePlayerRapido_${tmdb}'                 (solo {idcontenido,
                     →  'cachePlayerRapido_${tmdb}_T${t}_C${c}'     temporada, capitulo, segundo}
```

`_saveCache()` (`:329-368`) escribe ambos + `GuardadosBus.bump()`.

**Momento de escritura**: timer periódico **30 s** (`:417-422`, solo si
`_isPlaying`), al **98 %** (`:1519-1521`), al terminar (`:1523-1530`), antes de
`_showExitConfirmation` (`:1941`), al cambiar calidad (`:1815`), al minimizar
(`:2014`) y en `dispose()` (`:2069-2074`).

**(2) Supabase `historial` (sync)**

`_saveHistorialToDb()` (`player_page.dart:371-415`):

```dart
await SupabaseData.saveHistorial(
  progressSeconds: pos,          // int
  season: season, episode: episode,
  titulo: ..., backdrop: ..., poster: ...,
  durationSeconds: ...,
);
```

- **Ignora si `pos < 5`** (`:376`)
- Upsert con `onConflict: 'profile_id,tmdb_id,tipo,season,episode'`
  (`supabase_data.dart:215-218`)
- Workaround NULL: se usa **0 como centinela** para películas
- Invitado/sin sesión → `local_historial` en prefs, **top 100** (`:184-197`)

**(3) Mini-player** (`mini_player_service.dart:85-120`): escribe en las **mismas
claves** (`_cacheKey()`/`_cacheKeyRapido()` idénticos, `:56-68`), throttle 5 s,
y **excluye tipo `live`** (`:88`).

### 6.2 Restauración — `_getSavedPosition` (`player_page.dart:425-482`)

3 pasos en cascada:

1. **`cachePlayerRapido_<...>`** → `segundo` si `> 5` (se consulta primero: es la
   lectura más rápida)
2. **`cachePlayer_<...>`** → `segundo` si `> 5`
3. **`SupabaseData.getHistorial()`** → recorre la lista: filtra por `tmdb_id`,
   por `tipo` (tv vs movie, `:453-456`), y para series compara `season`/`episode`
   con tolerancia `sInt != 0` (`:462-473`) → devuelve `progress_seconds ?? segundo`
   si `> 5`

→ El resume es **al segundo exacto**:
`Duration(seconds: saved)` en `player_page.dart:1128`,
`tv_player_page.dart:1628`, `tv_discover_player.dart:865`,
`discover/domain/mobile/player_screen.dart:559`.

---

## 7. Errores y reintentos

| Nivel | Mecanismo | Ubicación |
|---|---|---|
| **Por servidor** | `_tryNextServer()` recorre `_fallbackServers` desde `_fallbackIndex+1`, `markServerAsInvalid` en cada fallo, `playable.url != _activeUrl` para no repetir | `player_page.dart:1152-1196` |
| **Durante init** | `catch` en `_startControllerWithUrl` → `_tryNextServer` | `:1140-1148` |
| **Resolución paralela** | Por lote: si un item falla se marca inválido y se sigue; el lote entero se reintenta | `:865-916` |
| **Progressive** | `Completer` con **timeout 45 s**, `cancelOnError: false`, cadena serializada | `:411-521` |
| **Extractor** | `MainFuentes.verificarConExtractor` timeout **8 s** | `:645-649` |
| **UI de error** | `_buildErrorScreen()` (`:2236-2315`): si `_allServersFailed` → botón "Abrir lista de servidores" + "Reintentar búsqueda" (`clearCache(CacheType.all)` + `_initializePlayer()`, `:2279-2294`); si no → "Probar otro servidor" → `_tryNextServer()` |
| **IPTV en vivo** | **7 intentos de User-Agent** secuenciales con 300 ms entre ellos, timeout 20 s por intento. Espera **400 ms** tras `dispose()` para liberar MediaCodec (`:149`) | `player_tvchanel.dart:123-219` |
| **Descarga** | `_maxSegmentRetries` por segmento con backoff 1s/2s/4s (`:368-419`); `_maxDownloadRetries` con backoff 2s/4s/6s (`:421-479`); distingue cancelación de usuario vía `DioExceptionType.cancel` (`:362-365`) | `download_manager.dart` |

### 7.1 Los 7 User-Agents de IPTV (secuenciales)

```
1. okhttp/4.12.0
2. VLC/3.0.16
3. Chrome Android
4. Lavf/58.76.100
5. okhttp + HLS
6. VLC + HLS
7. okhttp + DASH
```

### 7.2 Mapeo de errores IPTV a lenguaje humano (`:198-214`)

| Error | Mensaje |
|---|---|
| 403 | "geo-bloqueado o con token" |
| 404 | no encontrado |
| timeout | tiempo agotado |
| `MediaCodec/Decoder/CodecException` | "no se pudo decodificar" |
| `SocketException` | "error de red" |

---

## 8. Funciones adicionales

### 8.1 Skip Intro / Outro / Recap

- **IntroDB**: `https://api.introdb.app/segments` (`:538-569`), ventana
  `[start_sec, end_sec]`, botón `MobileSkipNextButton` con `autoHideMs: 15000`
  (`:2144-2167`)
- **TV**: `SkipSegmentsLoader` (`skip_button.dart:40`) con **fallback en cascada**:
  `api.introdb.app` → `api.skipdb.tv`; maneja `intro`, `outro` y `recap`, con
  `start_ms`/`end_ms` alterno

### 8.2 Next Episode Prompt

Se muestra si:

- `progress >= 0.90` **o**
- `remaining <= 3 min` **o**
- `remaining <= 2 s`

(`player_page.dart:1481-1501`). Tarjeta `NextEpisodePrompt` con barra de auto-hide
de 10 s.

**Precarga** al 90 %: `_serverLoader.preloadNext()` (`:1503-1517`).

### 8.3 Mini-player — handoff del controller ⭐

`_minimizeToBar()` (`player_page.dart:1995-2058`):

```dart
// Pone _controllerReady = false para que dispose() NO destruya el controller
mini.attachController(...);   // handoff del MISMO VideoPlayerController
// Si falla, REENGATHA el listener
```

Esto evita el clásico **"MediaCodec Decoder init failed"** al crear un 2º decoder.

Si el mini-player está deshabilitado → snackbar *"Mini-player desactivado en
Configuración → Player"*.

### 8.4 Chromecast

- `CastButton` (881 líneas): descubrimiento de dispositivos (`dart_cast`)
- `CastManager` (436 líneas): mantiene sesión con **Foreground Service** +
  notificación persistente con 3 botones (`Play/Pausa`, `Detener`, `+10s`)
  actualizada cada **3 s** (`:338-344`)
- `CastScreen` (646 líneas): control remoto con skip intro/outro

Permisos necesarios en el manifest: `ACCESS_WIFI_STATE`,
`CHANGE_WIFI_MULTICAST_STATE`, `NEARBY_WIFI_DEVICES` (neverForLocation).

### 8.5 Video fit

**Móvil** (`player_page.dart:34, 171-199, 2108-2115`):

```dart
enum _VideoFitMode { contain, cover, fill, fitWidth, fitHeight }
// Etiquetas: "Original/Expandir/Estirar/Ancho/Alto"
// Implementado con FittedBox(fit: BoxFit)
```

**TV** (`tv_player_page.dart:43-132, 342-357`):

```dart
enum _VideoFitMode { original, ratio16_9, ratio21_9, ratio16_10, ratio4_3 }
// _SegmentedVideoTrackShape personalizado (slider con segmentos)
// Label 2 s con toast
```

### 8.6 Otros

| Función | Detalle |
|---|---|
| Reloj / hora de fin | TV muestra hora actual (`_clockTimer` 30 s, `:610-624`) y `_horaFinEstimada` (`:359-372`) |
| Screensaver | TV, 1 min sin interacción y pausado → `ScreensaverOverlay` (`:635-646`) |
| Because You Watched | Overlay con botones "Producir siguiente" / "Continuar créditos" (`:276-277`) |
| Orientación | Landscape forzado + `immersiveSticky` (`:289-296`); contador estático `_activePlayers` para que el `pushReplacement` no ponga vertical al disposing del player viejo (`:71, 2060-2091`) |

---

## 9. `PlaybackSetupModal` (677 líneas)

`presentation/shared/modals/playback_setup_modal.dart` — modal de
configuración inicial **obligatorio** (TV lo fuerza: `tv_shell.dart:165`).

```dart
// :11-14
static bool hasPlaybackSetup() => prefs.containsKey('seleccionar_servidores');
// :18-26
static Future<void> showPlaybackSetupModal(BuildContext context) {
  // dialog con barrierDismissible: false
}
```

Contenido:
- **Modo**: `Auto` / `Manual`
- **Idioma**: `Latino` / `Castellano` / `Subtitulado`
- Si es Manual: un toggle por cada `kRegisteredSources` de `core/constants/sources.dart`

`_saveAndClose()` (`:153-173`) guarda:

```dart
'seleccionar_servidores'          = 'auto' | 'manual'
'idioma_audio_predeterminado'     = 'latino' | 'castellano' | 'subtitulado'
'verificar_servidores'            = true
'un_servidor_por_idioma'         = false
'mostrar_servidores_player'      = true
// + los flags <fuente>_enabled de cada toggle
```

**100 % D-pad**: `FocusNode` por control con `debugLabel`, `onFocusChange` →
`Scrollable.ensureVisible` para centrar el elemento enfocado (`:100-121`),
`KeyEventResult.handled` para ←/→/↑/↓/Enter.

`_onBack()` (`:175+`): diálogo de aviso *"Debes configurar el modo de
reproducción para ver contenido"*.

---

## 10. `SessionWatchTimer` (core/services/)

```dart
// session_watch_timer.dart:9-45
static const warnAfter = Duration(hours: 4);
```

- Timer de 1 min + check inmediato
- A las 4 h → `AlertDialog` no dismissible *"Descanso recomendado"* con
  `${hours}h ${mins}min usando la app`
- **Continuar** → reinicia otros 4 h (`:81-83`)
- **Salir de la app** → `SystemNavigator.pop()` (`:85`)
- Arranca en `mobile_shell.dart:115` y `tv_shell.dart:166`

⚠️ `shouldContinue` (`:90`) es un getter que **siempre devuelve `true`** —
código muerto.

---

## 11. `PlayerConstants` — código muerto

```dart
// core/constants/player_constants.dart (4 líneas)
class PlayerConstants {
  static const double defaultSpeed = 1.0;
  static const List<double> speeds = [0.5, 0.75, 1.0, 1.25, 1.5, 2.0];
}
```

No se importa en ningún sitio (`grep -rn "PlayerConstants"` → solo su
definición). **No hay control de velocidad de reproducción** en la UI pese a que
la constante existe.

---

## 12. Diagrama del flujo del player

```
Usuario toca contenido
        │
        ▼
  PageContenido (content_page.dart)
        │  _openServidores()  o  PlayerScreen directo
        ▼
  ServidoresModal (servers_modal.dart:242)
        │  fetchProgressive → eventos por servidor
        │  Caché: TvM3u8Cache (1h) / FuentesCache (12h) / Preloader (2d)
        ▼
  _openServer()  [servers_modal.dart:940]
        │
        ├── m3u8 fresco o URL directa ──► PlayerScreen (player_page.dart)
        │                                  │
        │                                  ├─ _initializePlayer() [5 caminos]
        │                                  ├─ ServerLoader → MainFuentes → extractors
        │                                  ├─ _playerHeaders() (UA + Referer/Origin)
        │                                  ├─ VideoPlayerController.networkUrl().initialize()
        │                                  ├─ _getSavedPosition() → seekTo(segundo exacto)
        │                                  ├─ Subtítulos (VTT propio | OpenSubtitles Stremio)
        │                                  ├─ HlsQualitySelectorModal (cambio en caliente)
        │                                  ├─ CastButton → CastManager (FG Service)
        │                                  ├─ Skip Intro (introdb → skipdb)
        │                                  └─ _saveCache() 30s / 98% / fin / salir
        │                                          └─ Supabase.saveHistorial(progress_seconds)
        │
        └── embed sin resolver ──► WebPlayerView (WebView + autoplay)

  Canal IPTV:
  HomeTvChanel(Tv) → PlayerTvChanel(Tv)
        ├─ 7 intentos de UA (okhttp → VLC → Chrome → Lavf → +HLS → +DASH)
        ├─ _ChannelGuide (zapping, 2 filas de foco)
        ├─ Favoritos (tvchanel_favorites_v1)
        └─ _minimize() → MiniPlayerService (tipo: 'live', sin historial)
```

---

## 13. Problemas

1. **4 ficheros de player de 1800-4500 líneas.** La lógica de caché, headers y
   resume está **4 veces duplicada** (móvil / TV / discover / webview).
2. **2 ficheros vacíos**: `presentation/tv/server_loader.dart`,
   `presentation/tv/tv_extractor_screen.dart`.
3. **`extractor_mobil.dart` es byte-idéntico a `extractor.dart`.**
4. **Los torrents/MKV se detectan y filtran pero nunca se reproducen**
   (`libtorrent_flutter` declarado sin usar). El switch `torrent_mkv_enabled` es
   engañoso.
5. **Sin control de velocidad** pese a `PlayerConstants.speeds`.
6. **`SessionWatchTimer.shouldContinue`** siempre `true` → código muerto.
7. La detección de URL directa (`_isDirectStreamUrl`) es una heurística de
   ~30 patrones; los falsos positivos abren el player con una URL que no es
   reproducible.