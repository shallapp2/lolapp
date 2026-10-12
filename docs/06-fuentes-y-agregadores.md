# 06 · Fuentes y agregadores

`lib/data/aggregators/` orquesta todas las fuentes (13 nativas + addons),
verifica los servidores y decide cuáles mostrar.

---

## 1. Los 4 agregadores

| Fichero | Líneas | Clase | Estado |
|---|---|---|---|
| `source_aggregator.dart` | 1196 | `MainFuentes` | ✅ **Agregador vigente** — 13 `FuenteId` + addons |
| `language_aggregator.dart` | 975 | `MainFuentes` | ⚠️ **Legacy/TV** — 9 `FuenteId`, `_maxConcurrent = 1` |
| `server_aggregator.dart` | 392 | `ServidoresAggregator` | Agregador simple, sin verificación |
| `main_fuentes_servidores.dart` | 272 | `MainFuentesServidores` | Ruta **por sitio** (`serieskao→embed69`, `cinehax→unlimplay`) |

> ⚠️ **`MainFuentes` está declarada DOS veces** (`source_aggregator.dart:596` y
> `language_aggregator.dart:529`). Ambas declaran `MainFuentes`, `FuenteId`,
> `FuenteSeleccion`, `FuentesConfig`, `FuenteEvent`, `FuentesResult`,
> `FuentesCache` y `_VerifyGate`. No colisionan porque cada consumidor importa
> solo una. **Importar ambas en un mismo library = error de compilación.**

---

## 2. `source_aggregator.dart` — el agregador maestro

### 2.1 `FuentesConfig` (`:185-304`)

Lee **20+ flags** de `SharedPreferences`.

```dart
class FuentesConfig {
  final bool cuevanaEnabled, unlimplayEnabled, cinecalidadEnabled, tioplusEnabled,
            embed69Enabled, poseidonEnabled, fuegocineEnabled, pelispediaEnabled,
            seriesmetroEnabled, smartpelisEnabled, cinesrcEnabled, customApiEnabled;

  final bool verificarServidores;          // 'verificar_servidores'        (default true)
  final bool unServidorPorIdioma;          // 'un_servidor_por_idioma'     (default true)
  final bool mostrarServidoresEnPlayer;
  final bool idiomaPredEnabled;  final String idiomaPredCode;
  final FuenteSeleccion seleccionFuente;   // manual | autoPrimera | autoIdioma
  final bool reutilizarUltimoEnlace;
  final int  capitulosPrecarga;            // 0..10
  final int  concurrentChecks;              // 'servidores_concurrent_checks' 1..5 (default 2)
  final bool nativeSourcesEnabled;          // 'native_sources_enabled' (default false) ← MASTER
}
```

### 2.2 `fuentesActivas` — el master switch (`:285-303`)

```dart
List<FuenteId> get fuentesActivas {
  if (!nativeSourcesEnabled) return [];     // ← si está OFF, NO hay fuentes nativas
  final list = <FuenteId>[];
  if (embed69Enabled)     list.add(FuenteId.embed69);
  if (poseidonEnabled)    list.add(FuenteId.poseidon);
  if (cuevanaEnabled)     list.add(FuenteId.cuevana);
  if (unlimplayEnabled)   list.add(FuenteId.unlimplay);
  if (cinecalidadEnabled) list.add(FuenteId.cinecalidad);
  if (tioplusEnabled)     list.add(FuenteId.tioplus);
  if (fuegocineEnabled)   list.add(FuenteId.fuegocine);
  if (pelispediaEnabled)  list.add(FuenteId.pelispedia);
  if (seriesmetroEnabled) list.add(FuenteId.seriesmetro);
  if (smartpelisEnabled)  list.add(FuenteId.smartpelis);
  if (customApiEnabled)   list.add(FuenteId.customapi);
  if (cinesrcEnabled)     list.add(FuenteId.cinesrc);   // ← SIEMPRE al final
  return list;
}
```

> ⚠️ **Hallazgo clave**: con `native_sources_enabled = false` (**el valor por
> defecto**), las 13 fuentes nativas quedan **desactivadas** y solo corren los
> **addons de Git**. Por eso `fetchProgressive` puede devolver
> `useOnlyAddons = true` (`:858`).

### 2.3 `_VerifyGate` — el semáforo de verificación (`:546-594`)

```dart
class _VerifyGate {
  static int  _inflight = 0;
  static int  _maxConcurrent = 2;                  // sincronizado con Settings
  static final List<Completer<void>> _waiters = [];
  static final Map<String, String>  _memCache = {};  // url → m3u8 ('' = falló)

  static void setMaxConcurrent(int n) { _maxConcurrent = n.clamp(1, 5); }

  static Future<void> acquire() async {
    if (_inflight < _maxConcurrent) { _inflight++; return; }
    final c = Completer<void>();
    _waiters.add(c);
    await c.future;
    _inflight++;
  }

  static void release() {
    _inflight--;
    if (_waiters.isNotEmpty) _waiters.removeAt(0).complete();
  }

  static void putCache(String url, String? m3u8) {
    _memCache[url] = m3u8 ?? '';
    if (_memCache.length > 200) _memCache.removeWhere(...);   // poda
  }
}
```

### 2.4 Normalización de idioma — el "canonificador" común (`:765-798`)

```dart
static String normalizeIdioma(String? raw) {
  if (raw == null || raw.isEmpty) return 'es_MX';
  final l = raw.toLowerCase().trim();
  if (l.contains('castellano') || l.contains('es_es') || l.contains('es-es') || l == 'esp') {
    return 'es_ES';
  }
  if (l.contains('sub') || l.contains('en_us') || l.contains('en-us') || l.startsWith('en') ||
      l.contains('inglés') || l.contains('ingles') || l.contains('english')) {
    return 'en_US';
  }
  if (l.contains('japon')) return 'ja_JA';
  return 'es_MX';   // default: LATINO
}

static String idiomaLabel(String code) => {
  'es_ES': 'Castellano', 'en_US': 'Subtitulado', 'ja_JA': 'Japonés', _: 'Latino',
}[code] ?? 'Latino';
```

### 2.5 `verificarConExtractor()` (`:803-831`)

```dart
static Future<String?> verificarConExtractor(BuildContext context, String url,
    {Duration timeout = _kVerifyTimeout}) async {
  final cached = _VerifyGate.cached(url);
  if (cached != null) return cached.isEmpty ? null : cached;   // '' = cacheado como fallido

  await _VerifyGate.acquire();           // espera turno
  try {
    if (!context.mounted) return null;
    final m3u8 = await ExtractorHlsService.buscarFuente(context, url, timeout: timeout);
    _VerifyGate.putCache(url, m3u8);
    return (m3u8 == null || m3u8.isEmpty) ? null : m3u8;
  } catch (_) {
    _VerifyGate.putCache(url, null);
    return null;
  } finally {
    _VerifyGate.release();
  }
}
```

### 2.6 `fetchProgressive()` — el corazón (`:833-1106`)

```dart
static const Duration _kVerifyTimeout     = Duration(seconds: 5);
static const int      _kMaxVerifyPerFuente = 4;   // máx 4 probes WebView por fuente
```

**Fase 1 — las fuentes prioritarias en paralelo** (`:867-975`):

```dart
Future<void> runFuente(FuenteId fuente) async {
  final idiomasDeEstaFuente = <String>{};
  var count = 0, verifiedAttempts = 0;

  await for (final raw in _scrapeFuente(
      fuente: fuente, tmdbId: ..., isMovie: ..., season: ..., episode: ...)) {
    final max = fuente == FuenteId.customapi ? 64 : maxPorFuente;
    if (count >= max) break;

    final map = Map<String, dynamic>.from(raw);
    final url = map['servidor_url']?.toString() ?? '';
    if (url.isEmpty || seenUrls.contains(url)) continue;   // dedup global
    seenUrls.add(url);

    map['idioma']        = normalizeIdioma(map['idioma']?.toString());
    map[fuente.flagKey]  = true;                             // es_cuevana / es_poseidon...
    map['fuente_id']     = ...;  map['fuente_label'] = ...;
    map['tmdb_id'] = tmdbId;
    map['season']  = isMovie ? 0 : season;
    map['episode'] = isMovie ? 0 : episode;

    if (fuente == FuenteId.cinesrc) {         // embeds WebView
      map['webview_only'] = true;
      map['prioridad']    = 999;
    }

    // 1 servidor por idioma y fuente
    if (unPorIdioma) {
      final idioma = map['idioma'] as String;
      if (idiomasDeEstaFuente.contains(idioma)) continue;
      idiomasDeEstaFuente.add(idioma);
    }

    final debeVerificar = verificar && context != null && context.mounted
        && verifiedAttempts < _kMaxVerifyPerFuente && fuente != FuenteId.cinesrc;

    if (debeVerificar) {
      verifiedAttempts++;
      final m3u8 = await verificarConExtractor(context!, url);
      map['resolved_m3u8'] = m3u8;
      map['verificado']    = (m3u8 != null && m3u8.isNotEmpty);
      controller.add(FuenteEvent(fuente: fuente, servidor: map,
                                 isVerified: map['verificado'] as bool, resolvedM3u8: m3u8));
    } else {
      map['verificado'] = false;
      controller.add(FuenteEvent(fuente: fuente, servidor: map, isVerified: false));
    }
  }
  controller.add(FuenteEvent(fuente: fuente, isDone: true));
}
```

**Fase 2 — CineSRC como último recurso** (`:977-1003`):

```dart
await Future.wait(prioritarias.map(runFuente));   // TODAS en paralelo

if (incluirCinesrc) {
  if (servidoresOtrasFuentes == 0) {
    runFuente(FuenteId.cinesrc);                   // ← solo si NADIE más dio nada
  } else {
    controller.add(const FuenteEvent(fuente: FuenteId.cinesrc, isDone: true));
  }
}
```

**Fase 3 — Addons** (`:1005-1096`):

```dart
await AddonManager.instance.reload();
await for (final raw in AddonSourceAdapter.instance.scrapeAll(
    tmdbId: ..., isMovie: ..., season: ..., episode: ...)) {
  if (addonCount >= maxPorFuente * 4) break;        // tope 4× por fuente
  map['idioma']       = normalizeIdioma(...);
  map['fuente_id']    = map['fuente'] ?? map['addon_id'] ?? 'addon';
  map['fuente_label'] = map['addon_name'] ?? map['provider'] ?? 'Addon';
  map['es_addon']     = true;
  if (unPorIdioma) { final key = '${map['fuente_id']}_$idioma'; ... }
  // verificaciones LANZADAS sin await (el gate las limita) y agrupadas
  verifyFutures.add(() async {
    final m3u8 = await verificarConExtractor(ctx, url); ...
  }());
}
if (verifyFutures.isNotEmpty) await Future.wait(verifyFutures);
```

**Agrupación por idioma** (`:1108-1149`):

```dart
final FuentesResult(
  porFuente: Map<FuenteId, List<Map>>,
  todos: [...],
  porIdioma: Map<String, List<Map>>,   // indexado con normalizeIdioma(...)
);
```

### 2.7 `FuentesCache` — caché persistente de 12 h (`:352-540`)

```dart
static const Duration defaultTtl = Duration(hours: 12);
static const _kLastLinkPrefix = 'ultimo_enlace_';
static const _kServersPrefix = 'serv_cache_';
static const _kTtlPrefsKey    = 'servidores_cache_ttl_hours';

String _serversKey({tmdbId, tipo, season, episode}) =>
    '$_kServersPrefix${tmdbId}_${tipo}_${season}_$episode';
String _lastLinkKey({...}) =>
    '$_kLastLinkPrefix${tmdbId}_${tipo}_${season}_$episode';

// payload: {'ts': DateTime.now().millisecondsSinceEpoch, 'servidores': [...]}
```

Métodos: `saveServers` / `loadServers` / `saveLastLink` / `loadLastLink` /
`clearFor` / `serversCacheAgeMs`. TTL configurable por el usuario.

---

## 3. `language_aggregator.dart` — versión legacy/TV

Estructura idéntica pero con **3 diferencias**:

| Diferencia | Detalle |
|---|---|
| Solo 9 `FuenteId` | Sin `seriesmetro`, `smartpelis`, `cinesrc`, `customapi` |
| `_maxConcurrent = 1` | (`:486`) — la TV no tolera varios WebViews a la vez |
| Política **más permisiva** | Aunque el probe falle, el servidor se emite (`:843-854` vs `source_aggregator.dart:925-949`) |

La política está documentada en el propio código (`:8-13`):

```dart
// - Máx. 1 verificación WebView a la vez (evita OOM / cortes en Android TV)
// - Timeout de probe más corto
// - Si el probe falla/timeout, NO se descarta el servidor: se emite como no verificado
// - Caché en memoria de resultados de verificación por URL
```

---

## 4. `server_aggregator.dart` — agregador simple (sin verificación)

```dart
// :19-78
class ServidoresConfig { /* mismas prefs, sin fuegocine */ 
  static fromPrefs(); static bool anyEnabled(...); }

// :80-92
class ServidorEvent { String source; ServidorMap? server; String? error; bool done; }

// :94
class ServidoresAggregator { ServidoresAggregator._(); }   // ← solo estáticos
```

Helper genérico de registro (`:112-142`):

```dart
void startSource(String source, Stream<dynamic> Function() runner,
                 ServidorMap Function(dynamic) toMap) {
  final ctrl = StreamController<ServidorEvent>();
  controllers[source] = ctrl;
  futures.add(() async {
    await for (final item in runner()) ctrl.add(ServidorEvent(source: source, server: toMap(item)));
    ctrl.add(ServidorEvent(source: source, done: true));
  }());
}
```

Merge de todos los streams con dedup (`:281-301`):

```dart
await for (final event in _StreamGroup.merge(active.map((c) => c.stream))) {
  if (event.server != null) {
    final url = ...;
    if (url.isNotEmpty && seenUrls.contains(url)) continue;
    seenUrls.add(url);
  }
  yield event;
  if (event.done) { finished++; if (finished >= pending) break; }
}
```

`groupBySource` (`:330-370`) clasifica por **flag booleano**:

```dart
static Map<String, List<ServidorMap>> groupBySource(List<ServidorMap> all) {
  final map = {'cuevana':[],'unlimplay':[],'cinecalidad':[],'tioplus':[],'embed69':[],
               'poseidon':[],'cinesrc':[],'pelispedia':[],'seriesmetro':[],
               'smartpelis':[],'customapi':[]};
  for (final s in all) {
    if (s['es_cuevana'] == true) map['cuevana']!.add(s);
    else if (...) ...
  }
}
```

> `_StreamGroup.merge` es un helper propio porque `package:async` no está en
> `pubspec.yaml`.

---

## 5. `main_fuentes_servidores.dart` — agregador por **sitio**

En vez de.source → servers, esta ruta es **sitio → fuente interna**.

```dart
// :32-47  MAPA SERVICIO → FUENTE INTERNA
static String _mapServicio(String servicio) {
  switch (servicio.toLowerCase().trim()) {
    case 'cuevana':   return 'cuevana';
    case 'serieskao': return 'embed69';      // ← serieskao se sirve vía embed69
    case 'pelisplus': return 'pelisplus';
    case 'tioplus':   return 'tioplus';
    case 'cinehax':   return 'unlimplay';    // ← cinehax se sirve vía unlimplay
    default:          return servicio.toLowerCase().trim();
  }
}
```

**Verificación estricta**: descarta el servidor si falla (`:102-120`):

```dart
final esDirecto = map['type'] == 'direct' || urlLower.contains('.mp4') || urlLower.contains('.m3u8');
if (verificar && context != null && context.mounted && !esDirecto) {
  final m3u8 = await ExtractorHlsService.buscarFuente(context, url,
      timeout: Duration(seconds: 10));
  if (m3u8 == null || m3u8.isEmpty) continue;      // ← NO se emite
  map['resolved_m3u8'] = m3u8; map['verificado'] = true;
} else {
  map['verificado'] = esDirecto;
}
```

Usado por: `features/discover/domain/mobile/servidores_modal.dart:229` y
`features/servers/presentation/tv_servers_modal_fuentes.dart:300` (con el mismo
mapeo reimplementado en `:93-98`).

---

## 6. `custom_api.dart` — las APIs PHP del usuario

`lib/data/datasources/remote/sources/custom_api.dart` (326 líneas).
**No confundir con Supabase.**

> **Docstring (`:1-13`)**:
> - Si el usuario pone solo un código (ej: 44029) → se usa la API de Modlyo:
>   `https://modlyo.com/lolfuentes/api.php?idcodigo=44029`
> - Si el usuario pone una URL completa → se usa exactamente esa URL
> - Prefs: `custom_api_enabled` (bool), `custom_api_sources` (JSON list de URLs)

```dart
// :63-64
static const String modlyoBase = 'https://modlyo.com/lolfuentes/api.php?idcodigo=';
```

`addSource` (`:116-160`):
- código suelto → `modlyoBase + código`
- URL completa → se usa tal cual
- sin `http` → asume código
- **valida** que una URL de `modlyo.com/api.php` tenga `idcodigo` (`:146-153`)

`scrapeUrl` (`:191-291`):

```dart
qp['idtmdb']   = '$tmdbId';
qp['tipo']     = isMovie ? 'movie' : 'tv';
if (!isMovie) { qp['temporada'] = '$season'; qp['capitulo'] = '$episode'; }

final res = await http.get(uri, headers: {
  'Accept': 'application/json',
  'Referer': 'https://modlyo.com/',
}).timeout(const Duration(seconds: 60));

// requiere body['status'] == 'ok'
// lee body['fuente']['titulo'] y body['servidores']
```

`_normIdioma` (`:174-188`): castellano → `es_ES`, sub/en → `en_US`,
japan → `ja_JA`, resto → `es_MX`.

`CustomApiServer.toModalMap()` (`:39-55`):

```dart
{
  'servidor_nombre': ..., 'servidor_url': ..., 'calidad': 'HD',
  'idioma': ..., 'estado': 'activo', 'es_customapi': true,
  'custom_codigo': ..., 'custom_source_url': ...,
  'fuente_id': 'custom_$código', 'fuente_label': ...,
}
```

Tope: **64 servidores** (`source_aggregator.dart:881`, `sources.dart:424`).

---

## 7. Cómo se **califican** los servidores (resumen consolidado)

| Criterio | Dónde | Efecto |
|---|---|---|
| **Idioma canónico** | `MainFuentes.normalizeIdioma` | `es_MX` (default) / `es_ES` / `en_US` / `ja_JA` |
| **1 por idioma y fuente** | `un_servidor_por_idioma` (default `true`) | `if (idiomasDeEstaFuente.contains(idioma)) continue;` |
| **Dedup URL global** | `seenUrls` en `fetchProgressive` y `ServidoresAggregator` | La misma URL de 2 fuentes se muestra 1 vez |
| **Verificación HLS** | `verificarConExtractor` con gate 1..5 y máx 4 probes/fuente | `map['verificado']` + `map['resolved_m3u8']` |
| **Tope por fuente** | `maxPorFuente = 16` (customapi = 64, addons = 4×16) | Evita listas infinitas |
| **CineSrc al final** | `prioridad: 999`, `webview_only: true` | Solo se consulta si **todas** las demás fallaron |
| **Orden en la UI** | `servers_modal.dart:1624-1635` | Primero `_esPlayer` (m3u8 fresco <1 h o URL directa), luego WebView |
| **Orden de idiomas** | `servers_modal.dart:471-479` `_langSortKey` | idioma preferido (0) → mismo prefijo `es` (1) → `es_MX,es_ES,en_US,ja_JA` (2..5) → resto (50) |
| **Badges Elite** | `_applyCorrienteBadges` (`servers_modal.dart:19-47`) | `StreamBadgeRepository.matchAllBadges` sobre nombre + calidad + provider |
| **Torrent/MKV** | `_isTorrentOrMkv` (`:663-679`) + `torrent_mkv_enabled` | Filtra `magnet:`, `.torrent`, `.mkv` salvo que se habilite |
| **Calidad** | `calidad` (string: `'HD'`, `'1080p'`, `'Auto'`, `'Digital'`) | **No se usa para ordenar**; solo se muestra (`:1751`). La calidad **real** se obtiene en runtime con `HlsQualityParser` |

### `_langSortKey` en detalle

```dart
// servers_modal.dart:471-479
int _langSortKey(String code) {
  if (code == cfg.idiomaPredCode) return 0;              // idioma preferido
  if (code.startsWith('es'))      return 1;              // cualquier español
  return switch (code) {
    'es_MX' => 2, 'es_ES' => 3, 'en_US' => 4, 'ja_JA' => 5,
    _ => 50,
  };
}
```

### `_isTorrentOrMkv`

```dart
// servers_modal.dart:663-680
bool _isTorrentOrMkv(Map<String, dynamic> m) {
  final u = (m['servidor_url'] ?? '').toString().toLowerCase();
  if (u.startsWith('magnet:')) return true;
  if (u.contains('.torrent'))  return true;
  if (u.contains('torrent:'))  return true;
  if (m['infoHash'] != null)   return true;
  if ((m['fmt'] ?? '').toString().toLowerCase() == 'mkv') return true;
  return false;
}
```

---

## 8. `ServidoresPreloader` — precarga silenciosa

`features/servers/presentation/server_preloader.dart` (276 líneas) +
`server_preloader_service.dart` (214) + variantes TV.

- Clave de caché: `serv_cache_<id>_<tipo>_<T>_<C>`
- TTL: **2 días**
- Se dispara al abrir un modal de servidores y precarga el siguiente contenido
- `clearCache()` público

---

## 9. `TvM3u8Cache` — caché de m3u8 en el modal

`features/servers/presentation/servers_modal.dart:58-191`

```dart
static const ttlMs = 60 * 60 * 1000;   // 1 h con timestamp POR ENTRADA
// key: m3u8_cache_v2_<id>_<tipo>_T<S>_C<E>
```

**Escritura serializada** con cadena de futures (`:69-81`) para evitar carreras
cuando varios `FuenteEvent` llegan a la vez.

---

## 10. Diagrama del flujo completo

```
                     ┌──────────────────────────┐
                     │ PageContenido            │
                     │ _openServidores()        │
                     └────────────┬─────────────┘
                                  ▼
                     ┌──────────────────────────┐
                     │ ServidoresModal          │
                     │ - ingest caché (12 h)    │
                     │ - ingest TvM3u8 (1 h)   │
                     │ - ingest preloader (2 d) │
                     └────────────┬─────────────┘
                                  ▼
              MainFuentes.fetchProgressive(tmdbId, isMovie, s, e)
                                  │
       ┌──────────────────────────┼──────────────────────────────┐
       ▼                          ▼                              ▼
  _scrapeFuente()           AddonSourceAdapter           (await Future.wait
  13 fuentes en              .scrapeAll()                 de las 13)
  paralelo                  cola de 3 workers
  (CineSrc al final)        (prefs 1..5)
       │                          │
       └────────────┬─────────────┘
                    ▼
        ┌───────────────────────┐
        │ Por cada servidor:   │
        │  · normalizeIdioma   │
        │  · dedup URL global  │
        │  · 1 por idioma      │
        │  · tope por fuente   │
        └───────────┬───────────┘
                    ▼
     ┌──────────────────────────────┐
     │ verificarConExtractor()      │
     │  1) _VerifyGate.cached(url) │
     │  2) acquire() ←── semáforo  │  (1..5 concurrentes)
     │  3) buscarFuente():         │
     │     a) NativeResolvers      │  (6 s)
     │     b) WebView 1×1         │  (timeout 5 s)
     │  4) putCache(url, m3u8)    │
     │  5) release()              │
     └──────────────┬───────────────┘
                    ▼
       FuenteEvent{ servidor: map, isVerified, resolvedM3u8 }
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
  _addServerToBuckets     UI: pestañas por fuente,
  (agrupado por idioma)   orden por idioma + verificable
       │
       ▼
  _openServer(servidor)  ──▶ PlayerScreen  (si m3u8 fresco o URL directa)
                         └─▶ WebPlayerView (si es embed sin resolver)
```

---

## 11. Problemas del subsistema

1. **`MainFuentes` duplicada** con 13 vs 9 fuentes → divergencia de comportamiento
   entre móvil y TV no documentada.
2. **`native_sources_enabled = false` por defecto** significa que una
   instalación limpia **no tiene fuentes** hasta que se instalan addons.
3. **Búsqueda secuencial** de fuentes en `registry.dart` (5 fuentes en serie).
4. `maxPorFuente` (16) puede descartar servidores válidos de fuentes pequeñas
   cuando varias fuentes devuelven muchos duplicados.
5. **CineSrc solo se consulta si no hay ni un servidor** → si hay un embed
   malo y nada más, se descarta la oportunidad de tener alternativas.
6. Los "clones" (`pelispedia`/`seriesmetro`/`smartpelis`) y
   (`extractor.dart`/`extractor_mobil.dart`) requieren refactor.