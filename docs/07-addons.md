# 07 · Sistema de addons (JS + Stremio)

Subsistema más complejo del proyecto. Hay **dos carriles paralelos** que nunca se
sustituyen:

```
┌────────────────────────────────┐  ┌────────────────────────────────┐
│  CARRIL A — JS / GitHub        │  │  CARRIL B — Stremio / Nuvio   │
│                                │  │                                │
│  AddonManifest con rawCode JS  │  │  manifest.json remoto          │
│  CommonJS: getStreams(...)     │  │  /catalog/{type}/{id}.json     │
│  Ejecutado en QuickJS          │  │  /stream/{type}/{id}.json      │
│  (flutter_js) con red propia   │  │  HTTP puro                     │
│                                │  │                                │
│  JsAddonRuntime (1260 L)       │  │  StremioAddonRepository (231)  │
│  SourceService    (529 L)      │  │  StremioStreamService (225)    │
│  AddonLoader      (952 L)      │  │  StremioCollectionRepo (304)   │
└────────────┬───────────────────┘  └───────────────┬────────────────┘
             │                                      │
             └──────────────┬───────────────────────┘
                            ▼
             AddonSourceAdapter.scrapeAll()   ← cola de workers
                            │                  (concurrencia 1..5)
                            ▼
          Map<String,dynamic>  (vocabulario del modal de servidores)
                            │
                            ▼
                    ServidoresModal
```

---

## 1. Tipos de addon soportados

```dart
// models/addon.dart:3
enum AddonType { catalog, source, function }

// models/addon.dart:30-37
enum FunctionPlacement { player, contentPage, home, search, servers, settings }

// models/addon.dart:76-83
enum FunctionKind { subtitles, rating, posterTags, top10, mainSlider, generic }

// models/addon.dart:154
enum AddonOrigin { builtIn, url, pasted, file, package, github }
```

### Mapeo `AddonType` ↔ `resources` Stremio (`addon_loader.dart:317-338`)

| AddonType | resources generadas |
|---|---|
| `catalog` | `['catalog','meta']` |
| `source` | `['stream']` |
| `function` + `subtitles` | `['subtitles']` |
| `function` + `rating` | `['rating']` |
| `function` + `posterTags` | `['posterTags']` |
| `function` + `top10` | `['top10']` |
| `function` + `mainSlider` | `['mainSlider']` |
| `function` genérico | `['function']` |

### Capabilities y modo

`AddonManifest` (`models/addon.dart`):

```dart
mode        → 'vod' | 'tv' | 'both'   // inferido de extra['mode'] o capabilities
capabilities → Map<String, bool>
isChannelsOnly → heurística de 3 niveles
capabilityLabels → mapa ES de 10 capacidades
nsfw → desde extra['nsfw'], ageRating == 'R' o tag 'nsfw'
```

**`isChannelsOnly`** (`:304-355`) — heurística:
1. `extra['type'|'addonType'|'kind']` ∈ {channels, channel, live, tv, iptv, live-tv, livetv}
2. Intersección de `extra['types']` con *channelish* sin *vodish*
3. `mode == 'tv'`
4. `capabilities` con `channels` y sin VOD

**`capabilityLabels`** (`:408-429`) — 10 etiquetas en español:
`home`, `search`, `discover`, `meta`, `browse`, `episodes`, `resolve`,
`channels`, `download`, `subtitles`.

---

## 2. Formato de los manifests

### 2.1 Plantilla del repo — `assets/addons/example_source.json`

```json
{
  "id": "example.source",
  "name": "Fuente de ejemplo",
  "version": "1.0.0",
  "description": "Manifest JSON de ejemplo (tipo stream). Úsalo como plantilla para crear el tuyo.",
  "resources": ["stream"],
  "types": ["movie", "series"],
  "catalogs": [],
  "idPrefixes": ["tmdb"]
}
```

### 2.2 Formato GitHub propio (documentado en `addon_loader.dart:163-172`)

```jsonc
{
  "id", "name", "version", "description",
  "type": "catalog" | "function",
  "logo", "author",
  "placements": ["player", "contentPage", ...],   // solo function
  "requiredFields": ["tmdbId", "type", "season", "episode"],
  "entry": "index.js" | "main.js"
}
```

### 2.3 Configuración del addon (`config.schema`)

Orden de búsqueda (`addon_loader.dart:372-417`):

1. `manifest['configSchema']`
2. `manifest['config']['fields']`
3. `manifest['settings']`
4. Archivos separados (secundario/secuencial, timeout 6 s):
   `config.schema.json`, `config.json`, `settings.json` → `cfg['fields'] ?? cfg['configSchema'] ?? cfg`

Campos reconocidos por `AddonConfigField.fromJson` (`addon.dart:174-193`):

| Campo JSON | Alias |
|---|---|
| clave | `key` \| `name` \| `id` |
| etiqueta | `label` \| `title` |
| tipo | `type` (string/password/bool/number/select) |
| default | `default` \| `defaultValue` |
| descripción | `description` \| `hint` |
| opciones | `options` (lista de strings o de maps `{value,label,id}`) |

### 2.4 Normalización de `extra` — `buildCatalogExtra()` (`addon_loader.dart:724-865`)

Aplana: capabilities, mode, mediaKinds, genres, homepage, color, minAppVersion,
ageRating, defaultCategory, updateUrl, provider, apiVersion, download, nsfw,
discoverable, tags, hosts, `streamHosts`/`fetchHosts`/`liveStreamHosts`,
permissions, catalogParams — y copia el resto del `extra` anidado **sin pisar**
claves críticas (`types`, `type`).

### 2.5 Paquetes Nuvio (scrapers)

- `SourcePackage` / `PackageScraper` (`addon.dart:610-757`), `filename` obligatorio
- `AddonLoader.fetchPackage()` (`:74-100`) exige `map['scrapers'] is List`:

  ```dart
  if (map['scrapers'] is! List) {
    throw 'El JSON no es un paquete de scrapers (falta "scrapers").';
  }
  ```

- `resolveScraperUrl()` (`:103-112`) quita el `.json` del `manifestUrl` y concatena
  el `filename`

---

## 3. `AddonManager` (412 líneas)

Singleton `ChangeNotifier` con constructor privado (`addon_manager.dart:17-18`).
Inyecta `StorageService`, `AddonLoader`, `SourceService sources` y
`CatalogService catalogs`.

### 3.1 Filtros de colecciones (`:31-47`)

```dart
List<AddonManifest> get sourceAddons =>
    addons.where((a) => a.enabled && a.isSource && !a.isChannelsOnly).toList();
List<AddonManifest> get sourceAddonsIncludingChannels =>
    addons.where((a) => a.enabled && a.isSource).toList();
List<AddonManifest> get catalogAddons =>
    addons.where((a) => a.enabled && a.isCatalog).toList();
List<AddonManifest> get allSources  => addons.where((a) => a.isSource).toList();  // incl. desactivados
List<AddonManifest> get allCatalogs => addons.where((a) => a.isCatalog).toList();
```

### 3.2 `init()` (`:49-70`)

```dart
addons   = await _storage.loadAddons();
packages = await _storage.loadPackages();
await StremioAddonRepository.instance.init();
await StremioCollectionRepository.instance.init();
await StreamBadgeRepository.instance.init();
```

### 3.3 Instalación

**`installFromGitHub({repoOrUrl, type})`** (`:134-166`): delega en
`_loader.fromGitHub`, normaliza `repoClean` (quita `https://github.com/`,
`@branch`, `#…`), fuerza `types:[type]`, `enabled:true` y `githubCanonical`.

**`installCommunity({repo, type})`** (`:178-237`): **3 escalones de fallback**

1. `installFromGitHub`
2. Barrido `branches ∈ {main, master}` × `files`:
   - catalog: `manifest.json`, `addon.json`, `index.js`, `catalog.json`
   - source: `manifest.json`, `addon.json`, `index.js`, `main.js`

   contra `https://raw.githubusercontent.com/$cleaned/$br/$f`
3. Si el JSON descargado **no trae JS real** (`codeEmpty` = null/vacío/empieza
   por `{`) y es source → prueba `index.js`, `main.js`, `addon.js` y acepta el
   que contenga `getStreams`

### 3.4 Gestión

| Método | Líneas | Qué hace |
|---|---|---|
| `uninstall` | 240 | Borra |
| `setEnabled` | 246 | Activa/desactiva |
| `updateAddon` | 255-282 | Recarga por `githubCanonical` o `sourceUrl`, preservando `enabled` y `userConfig` |
| `updateAll` | 284 | Actualiza todos |
| `getStreams` / `getFirstStream` / `getStreamsForAddon` | 293/302/311 | Streams |

### 3.5 Paquetes Nuvio (`:329-410`)

`addNuvioPackage`, `removeNuvioPackage`, `refreshNuvioPackage`,
`installNuvioScraper`, `installAllNuvioScrapers` (devuelve `record ({int ok, int fail})`),
`isNuvioScraperInstalled`.

---

## 4. `AddonLoader` (952 líneas) — descarga y parseo

### 4.1 `fromUrl(url)` (`:13-54`)

Timeout 20 s. Decide por extensión `.json` o body que empieza por `{`:
- **JSON** → rechaza paquetes (`scrapers` es List) con mensaje explícito; si no,
  `fromJsonString(body, sourceUrl, origin: AddonOrigin.url)`. Luego descarga el
  **entry JS** declarado (`map['entry'] ?? map['main'] ?? 'index.js'`)
  resolviendo contra la base del manifestUrl y lo guarda en `rawCode`
- **No JSON** → `fromJsString(body, …)`

### 4.2 `fromGitHub(input, {...})` (`:173-451`)

1. **`GitHubAddress.parse`** (`:885-942`) acepta:
   - `owner/repo`
   - `owner/repo/path@branch`
   - `github.com/owner/repo/tree|blob/<ref>/<path>`
   - `raw.githubusercontent.com/owner/repo/ref/path/file.json`
2. Descarga **en paralelo** 4 candidatos de manifest:
   `manifest.json`, `addon.json`, `catalog.json`, `package.json` (timeout 8 s),
   prefiriendo `manifest.json`/`addon.json` sobre el `package.json` de monorepo
3. Descarga **en paralelo** entry + fallbacks `index.js`, `main.js`, `addon.js`
   (timeout 10 s), prefiriendo el `entry` declarado
4. Tipo: **manda `expectedType`** si es `source`/`catalog`
5. `functionKind` inferido de `manifest['functionKind'|'kind']`, por exports, o
   `defaultPlacement` como fallback
6. `requiredFields` por defecto `['tmdbId','type','title']` para functions

`GitHubAddress.rawUrl(file)` (`:944-948`):

```dart
'https://raw.githubusercontent.com/$owner/$repo/$ref/$path/$file'
```

### 4.3 `_detectExports(code)` (`:665-687`)

Detecta el tipo de addon por *sniffing* del código:

```dart
getStreams   → source
getSubtitles → function
catalog / getCatalog → catalog (solo si NO hay getStreams)
getMeta      → catalog
module.exports → source
```

---

## 5. `JsAddonRuntime` (1260 líneas) — QuickJS ⭐

Usa `package:flutter_js` (QuickJS embebido). **Sí evalúa JS arbitrario** de los
addons, con un **sandbox de red propio**.

### 5.1 Inicialización (`:39-49`)

```dart
_rt = getJavascriptRuntime(xhr: false);      // sin xhr del plugin: fetch propio
_rt!.enableHandlePromises();                  // manejo de promesas
_registerHttpBridge();                        // onMessage 'httpFetch'
_registerTimerBridge();                       // onMessage 'jsTimer'
_installBasePolyfills();                      // const _polyfillsJs (755-1260)
```

### 5.2 Carga de un addon — `loadAddonCode(addonId, code)` (`:316-391`)

1. Cache en memoria por hash: `_loaded: Set<String>` + `_codeHash: Map<String,int>`
2. **Detección de dependencias por *sniffing*** (`:322-339`):

   ```dart
   final needsCrypto  = code.contains('crypto-js') || code.contains('CryptoJS') ||
                        code.contains("require('crypto") || code.contains('require("crypto');
   final needsCheerio = code.contains('cheerio');
   ```

   Si falta la lib → `debugPrint` de aviso, pero **no aborta**.
3. Envuelve el código en una IIFE con `module`/`exports`, ejecuta con `eval`,
   y resuelve el export real (`:344-374`):

   ```dart
   delete globalThis.getStreams;
   eval(__src);
   var __exp = module.exports;
   if (__exp?.default && typeof __exp.default === 'object') __exp = __exp.default;
   if (!__has) { /* fallback a globalThis.getStreams */ }
   globalThis.__addons[$idLit] = __exp;   // o { __loadError: msg }
   ```

4. Verifica `__loadError` y lanza `Exception('Addon JS falló al iniciar: …')`

### 5.3 `getStreams` — `callGetStreams(...)` (`:395-466`)

Construye un script async IIFE que resuelve la función en este orden:

```
addon.getStreams → addon.stream → addon.default.getStreams
                 → addon.default (función) → addon (función)
```

Invocación:

```js
var result = await fn.call(addon, $tmdbLit, $typeLit, $seasonLit, $episodeLit);
```

Vuelve con `rt.evaluateAsync(script)` + `rt.executePendingJob()` +
`rt.handlePromise(...)` con **timeout 30 s**.

### 5.4 Parseo de resultados — `_parseResult` (`:473-556`)

- **Doble `jsonDecode`** (por si el JS devuelve string JSON dentro de JSON)
- Acepta `List`, `{streams:[…]}`, `{url:…}` o `{data:…}`
- URL: `url | file | src | magnet`; si hay `infoHash` sin URL →
  `magnet:?xt=urn:btih:$infoHash`; prefijo `//` → `https:`
- **Headers de proxy Stremio**: `behaviorHints.proxyHeaders.request` se fusiona
  en `headers` (`:524-535`)
- `isTorrent` / `isHls` (`.m3u8`) calculados (`:540-541`)

### 5.5 Otros métodos

| Método | Timeout | Para qué |
|---|---|---|
| `callExtract(addonId, embedUrl)` (`:644-696`) | 15 s | `addon.extract` — convierte embeds en m3u8 directos |
| `callCatalogMethod({addonId, code, method, args, config})` (`:578-641`) | 25 s | API genérica `getHome / search / discover / getMeta` |
| `ensureAddonLoaded(addonId, code)` (`:700-720`) | — | Segundo cargador para catálogos → `globalThis.__nuvioAddons[id]` |
| `evaluateAsyncPublic(script)` (`:723-750`) | 45 s | Puente "Kino": evalúa cualquier script async |

### 5.6 Puentes Dart→JS

#### HTTP — `_httpFetch` (`:105-184`)

```dart
// Redirecciones manuales: followRedirects = false + loop de hasta 6 saltos
// En 303, o en 301/302 con POST, degrada a GET y limpia body

// Detección de charset
if (contentType =~ /charset=(iso-8859-1|latin1|windows-1252)/) {
  body = latin1.decode(bytes);   // en vez de utf8
}
```

Devuelve `{ok, status, body, url, redirected, headers}` en JSON. Timeout 18 s.

#### Timers — `_registerTimerBridge()` (`:58-82`)

Crea un `Timer` real de Dart; al disparar ejecuta
`rt.evaluate('globalThis.__fireTimer($id)')` + `executePendingJob()`.

El lado JS (`_polyfillsJs:819-843`) mantiene la tabla `timers{id}` con
`{fn, ms, args, interval}` y `setInterval` se rearma a sí mismo.

### 5.7 Polyfills de Node (`_polyfillsJs`, `:755-1260`)

**Globals**: `self`, `window`, `global`, `console`, `performance.now`,
`process` (con `env.NODE_ENV='production'`, `browser:true`, `versions.node`,
`nextTick`, `cwd`, `emitWarning`, `on`).

**ES extras**: `String.prototype.replaceAll`, `Array.prototype.at`,
`Object.fromEntries`, `Promise.allSettled`.

**Timers reales** vía puente: `setTimeout`, `setInterval` (con
`Math.max(4, ms)`), `setImmediate`, `clearTimeout/Interval/Immediate`,
`__fireTimer`.

**Otros**: `crypto.getRandomValues`, `btoa`/`atob`, `TextEncoder`/`TextDecoder`,
`Buffer` completo (`from`, `alloc`, `isBuffer`, `concat`, `byteLength`, con
`toString('base64'|'hex'|'latin1'|'utf8')`), `URLSearchParams` con iteradores,
`URL` con parsing por regex, `AbortController`/`AbortSignal`.

#### `fetch` (`:1090-1161`)

```dart
// Normaliza headers (array, _raw, Headers-like)
// URLSearchParams → body + content-type
// JSON delega a sendMessage('httpFetch')
// Soporta signal de aborto
// Construye Response-like con text()/json()/arrayBuffer()/clone()
```

`json()` **rechaza explícitamente** si el body empieza por `<` con un error
descriptivo (`:1110-1113`) — evita que un HTML de error se parsee como JSON.

#### `axios` mínimo (`:1164-1223`)

```js
makeAxios(defs) → get/post/put/patch/delete/head/options/request/create/defaults
baseURL, params, validateStatus, err.isAxiosError, err.code='ERR_BAD_RESPONSE'
// Expuesto como globalThis.__axios (+ __axios.default)
```

#### `require` (`:1226-1255`)

```js
'crypto-js' | 'crypto' → globalThis.CryptoJS   (lanza si no cargó)
/^cheerio/              → globalThis.__cheerio (lazy {load(){}} que falla al usar)
'axios'                 → globalThis.__axios
'url'                   → { URL, URLSearchParams }
'buffer'                → { Buffer }
'querystring'           → { parse, stringify }
otro                    → throw "Module '<x>' is not available"
```

### 5.8 Carga de crypto-js y cheerio desde assets

```dart
// js_runtime.dart:17-24
static const _cryptoAsset  = 'assets/js/crypto-js.bundle.js';
static const _cheerioAsset = 'assets/js/cheerio.bundle.js';
static const _cryptoCdn = [
  'https://cdnjs.cloudflare.com/ajax/libs/crypto-js/3.1.9-1/crypto-js.min.js',
  'https://cdn.jsdelivr.net/npm/crypto-js@3.1.9-1/crypto-js.js',
];
static const _userAgent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/120.0.0.0 Safari/537.36';
```

`_loadCryptoJs()` (`:208-236`): si `globalThis.CryptoJS.AES` ya existe → listo.
Si no, itera `['asset:assets/js/crypto-js.bundle.js', ...cdn]`; el asset se lee
con `rootBundle.loadString()`, exige `src.length >= 1000` y ejecuta
`_libLoaderJs(src, 'CryptoJS')`. **Solo crypto-js tiene fallback CDN.**

`_loadCheerio()` (`:251-272`): **sin fallback CDN**. Lee
`rootBundle.loadString(_cheerioAsset)`; si falla, `debugPrint` y `return`.

#### El truco `_libLoaderJs` (`:276-300`)

Ambos bundles están compilados con `var __libExport = (()=>{…})()`. El loader
los evalúa dentro de una función con `module`/`exports`/`define`/`require`
locales, sin depender de globals:

```js
(function () {
  try {
    var src = <JSON literal del bundle>;
    var m = { exports: {} };
    var fn = new Function('module', 'exports', 'define', 'require',
      src + '\n;return (typeof __libExport !== "undefined") ? __libExport '
      + ': ((module.exports && Object.keys(module.exports).length) '
      + '? module.exports : globalThis.CryptoJS);');
    var lib = fn.call(globalThis, m, m.exports, undefined, undefined);
    // verify: CryptoJS → if (lib && lib.AES) globalThis.CryptoJS = lib
    //         cheerio  → busca el que tenga .load, smoke test
    //                     c.load('<div><a class="x">hi</a></div>')('a.x').text() === 'hi'
    //                     y guarda globalThis.__cheerio
  } catch (e) { return 'err:' + e; }
})();
```

---

## 6. `SourceService` (529 líneas) — resolución de streams

### 6.1 `getStreams` (`:20-56`)

2 etapas:
1. Cada addon JS `enabled && isSource` vía `_resolveAddon`
2. `StremioStreamService.getStreamsForContent`

Dedupe por URL.

### 6.2 `getFirstStream` (`:59-83`)

Modo auto **secuencial**: la primera fuente que responda con ≥1 URL, timeout 20 s
por addon.

### 6.3 `_resolveJsCode(addon)` (`:155-239`) — 4 rutas

1. `rawCode` (si no empieza por `{` y contiene `getStreams` o `module.exports`)
2. IDs hardcodeados `embed69`/`cinecalidad` o `origin == builtIn` → descarga
   `sourceUrl` ?? `cinecalidadJsUrl`:
   ```dart
   static const cinecalidadJsUrl = 'https://raw.githubusercontent.com/'
     'KennethJYS/Nuvio-Providers-Latino/refs/heads/main/providers/cinecalidad.js';
   ```
3. `sourceUrl` directo
4. `githubCanonical` parseando `@branch` y probando candidatos:
   `<sub>/index.js`, `<sub>/plugin.js`, `index.js`, `plugin.js`

### 6.4 Detección de stubs (`:115-140`)

Para dar un error legible al usuario:

```dart
// isStub: <2000 bytes y contiene 'stub'/'sustituye este archivo'
//         o (getStreams sin cuevana/scrape/fetch)
// looksEmpty: <1500 bytes sin scrape/extract/__NEXT_DATA__
```

### 6.5 `resolveTmdb(content)` (`:284-384`) ⭐ — el punto más delicado

**Prioridad de 6 niveles**:

1. `extra.url_personalizada | urlPersonalizada | episodeUrl | kinoRef`
2. Candidatos que contengan URL HTTP embebida: `content.id`, `extra.tmdbId`,
   `extra.tmdb_id`, `extra.lacartoonsUrl`, `extra.jkanimeUrl`
3. `TmdbService.parseId(content.id)` (formato `tmdb:movie:123` / `tmdb:series:123:1:5`)
4. `extra.tmdbId`
5. `extra.jkanimeUrl | lacartoonsUrl | sourceUrl`
6. `content.id` si empieza por `jkanime:` / `lacartoons:` / `http` o contiene esos dominios

### 6.6 `_unwrapStreamKey(raw)` (`:243-273`)

Extrae URL + `S1E1` de claves compuestas tipo `kino:pkg:https://site/serie/5:s1e1`:

```dart
RegExp(r'[/:.]s(\d+)e(\d+)$')
```

Con recursión para `kino:` anidados.

### 6.7 `_runJsGetStreams` (`:393-425`)

```dart
// log de diagnóstico
// SIEMPRE recarga (loadAddonCode) "para evitar quedar con stub viejo"
await JsAddonRuntime.instance.callGetStreams(
  addonId: addon.id, code: code,
  tmdbId: tmdbId, type: type, season: season, episode: episode, addonName: addon.name);
// y aplica _applyExtractor
```

### 6.8 `_looksPlayable` (`:427-434`)

```dart
bool _looksPlayable(String u) {
  final l = u.toLowerCase();
  return l.contains('.m3u8') || l.contains('.mp4') || l.contains('.mkv')
      || l.contains('playlist') || l.contains('/hls/');
}
```

Lo que no es playable pasa por `callExtract` (`:436-482`).

### 6.9 `_resolveStremioStyle` (`:484-516`)

Para `rawCode` que empieza por `{`:

```dart
GET {base}/stream/{type}/{id}.json
// y parsea data['streams']
```

---

## 7. `CatalogService` (441 líneas)

**Principio declarado en el docstring** (`:10-15`):

> *"La app NO llama a TMDB directamente"* — los datos vienen de (1) JS del addon,
> (2) `json_api`/`static_json` en `extra`, (3) addons Stremio de catálogo.

### 7.1 `_config(addon)` (`:22-43`)

Prioridad documentada: `extra` → defaults del schema (`putIfAbsent`) →
`userConfig` (que sobrescribe).

> **Comentario clave (`:20-21`)**: *"La API key debe venir del ADDON (extra o
> hardcode en index.js), no de la app."*

### 7.2 Métodos

| Método | Qué hace |
|---|---|
| `getHomeRows(catalogs)` (`:45-85`) | Agrega 3 fuentes: catálogos JS/`json_api` → `StremioAddonRepository.buildHomeSections()` → `StremioCollectionRepository.buildHomeSectionsFromCollections()`. Cada uno con `try/catch` + `debugPrint` para no romper el home |
| `search(query, catalogs)` (`:87-103`) | Dedupe por `item.id`; marca cada resultado con `extra.catalogId` / `extra.catalogName` |
| `discover({addon, category, genreId, page})` (`:105-134`) | Llama método JS `'discover'` con args redundantes para compat (`genreId`, `genero`, `genre`); fallback `_jsonApiList` |
| `getDetails(item, catalogs)` (`:136-161`) | Itera catálogos llamando `'getMeta'`; el primero que devuelva algo gana |

### 7.3 Parsing tolerante

- `_parseRows` (`:228-258`) acepta `rows`, `catalogs` o `items`
- `_parseItems` (`:260-280`) acepta `items` / `results` / `data`
- **`_mapLoose` (`:294-421`) es el parser de emergencia**: normaliza
  temporadas/episodios desde `seasonNumber|season_number|number`,
  `episodeNumber|episode_number|number`, y **extrae URLs de capítulo** de
  `url|link|extra.url|extra.jkanimeUrl` hacia `extra.url_personalizada` /
  `jkanimeUrl` / `tmdbId` (`:340-350`) — el puente con fuentes JKAnime/LaCartoons.
  También fuerza `type = series` si hay temporadas (`:389-393`) y corrige
  títulos "jkanime"/"jk anime" (`:394-404`)
- `_jsonApiList` (`:423-440`): base desde `configValue('baseUrl') ?? extra['baseUrl']
  ?? extra['base']`, path desde `extra['paths'][action] ?? '/$action'`

---

## 8. `StorageService` (77 líneas)

3 claves `SharedPreferences`:

```dart
static const _addonsKey   = 'addons_v1';
static const _packagesKey = 'source_packages_v1';
static const _settingsKey = 'addon_settings_v1';
```

⚠️ `saveAddons()` (`:27-38`) **poda `rawCode` si supera 80.000 caracteres**:

```dart
if (code.length > 80000) j['rawCode'] = null;   // no reventar el límite de prefs
```

El JS se re-descarga luego desde `sourceUrl`/`githubCanonical` vía
`SourceService._resolveJsCode`.

---

## 9. `CommunityService` (181 líneas) — GitHub Search

```dart
// community_service.dart:33-38
static const topicSource     = 'lol-addon-fuentes';
static const topicCatalog    = 'lol-addon-catalogos';
static const topicSourceAlt  = ['lol-addon-source', 'lol-addon-fuente'];
static const topicCatalogAlt = ['lol-addon-catalog', 'lol-addon-catalogo'];
// :160-165
static const topicTvChanel      = 'lol-tvchanel';
static const topicTvChanelAlt   = ['lol_tvchanel','lol-addon-tvchanel','lol-addon-tv-channels'];
```

Búsqueda:

```
GET https://api.github.com/search/repositories
    ?q=topic:<t>+fork:false&sort=stars&order=desc&per_page=30
Accept: application/vnd.github+json
User-Agent: LolApp1-Addons/1.0
timeout: 15 s
```

Seed offline: `loladdons/allpeliculasse` (`:41-49`).
`fetchAll()` (`:134-143`) corre sources y catalogs en `Future.wait` y los
mergea con los seeds (`:145-156`).
`fetchTvChanelAddons()` (`:170-180`) usa type `'tvchanel'`.

---

## 10. Carril Stremio

### 10.1 `StremioUrlBuilder` (104 líneas)

```dart
// stremio_url_builder.dart:3-13
String stremioTransportBaseUrl(String manifestUrl) {
  final noQuery = manifestUrl.split('?').first;
  if (noQuery.endsWith('/manifest.json'))
    return noQuery.substring(0, noQuery.length - '/manifest.json'.length);
  if (noQuery.endsWith('manifest.json'))
    return noQuery.substring(0, noQuery.length - 'manifest.json'.length);
  return noQuery.endsWith('/') ? noQuery.substring(0, noQuery.length - 1) : noQuery;
}

// :41-59
String buildStremioResourceUrl({required String manifestUrl, required String resource,
                                required String type, required String id,
                                String? extraPathSegment}) {
  // {base}/{resource}/{type}/{encodedId}.json
  // {base}/{resource}/{type}/{encodedId}/{extraPathSegment}.json
  // El query string del manifestUrl original SE PRESERVA y se anexa (:50-52)
}
```

`buildCatalogUrl` (`:61-87`): extras en orden fijo `search=`, `genre=`, `skip=`.
`normalizeManifestUrl` (`:89-104`): añade `https://` si falta y `manifest.json`
si no termina en `.json`.

### 10.2 `StremioManifestParser` (163 líneas)

`StremioManifestParser.parse` (`:7-34`): exige `id` y `name` no vacíos →
`FormatException('Manifest inválido: faltan id o name')`.

- `_parseResources` (`:36-68`): acepta `String` (usa defaults de raíz) u objeto
  `{name, types, idPrefixes}` con fallback a los tipos/prefixes raíz
- `_parseCatalogs` (`:70-110`): acepta la forma moderna
  `extra: [{name,isRequired,options,optionsLimit}]` **y** la antigua
  `extraSupported`/`extraRequired`
- `_parseHints` (`:112-121`): `configurable`, `configurationRequired`, `adult`, `p2p`
- `_resolveLogo` (`:123-129`): resuelve logo relativo contra la base del transport

`StremioCatalogParser.parse` (`:134-162`): dedupe por `type:id`; heurística de
paginación `if (metas.length >= 20) nextSkip = currentSkip + metas.length;`

### 10.3 `StremioStreamService` (225 líneas)

**`streamAddons()`** (`:23-31`): addons activos cuyo manifest declara
`resource == 'stream'` **o** que no declaran `resources` (heurística para addons
antiguos).

**`resolveVideoId(content, {season, episode})`** (`:36-72`):

| Prioridad | Origen | Normalización |
|---|---|---|
| 1 | `extra['stremioId']` | + `:s:e` si es serie y no trae ya `:` |
| 2 | `extra['imdb'] \| extra['imdb_id']` | → `tt…` + `:s:e` |
| 3 | `extra['tmdbId' \| 'tmdb_id']` | → `tmdb:{id}` + `:s:e` |
| 4 | Fallback: `content.id` | normaliza `tmdb:…`, elimina `tmdb:` duplicado |

`resolveType` (`:74-86`): `extra['stremioType']` → si no, `series|episode →
'series'`, `live → 'channel'`, `movie → 'movie'`.

`getStreamsForContent` (`:88-118`): `Future.wait` sobre todos los addons de
stream, dedupe por `'${url}|${infoHash}'`.

**`parseStreamsPayload`** (`:149-216`) — cascada de resolución:

```
url → magnet → infoHash→"magnet:?xt=urn:btih:" → ytId→"https://youtube.com/watch?v="
     → externalUrl
```

Fusiona `behaviorHints.proxyHeaders.request` y `headers`. Título =
`'$name · $description'`.

`_guessQuality` (`:218-224`): `4K, 2160P, 1080P, 720P, 480P, 360P, HD, SD`.

### 10.4 `StremioAddonRepository` (231 líneas)

Singleton `ChangeNotifier`, prefs key **`'stremio_catalog_addons_v1'`** (`:15`).

`addOrRefresh(url)` (`:54-89`): guarda el addon **aunque falle** (con
`errorMessage`) y hace `rethrow` — así la UI muestra el error sin perder el item.

`allCatalogTargets()` (`:108-123`): aplana addons×catalogs a `CatalogTarget`.

`buildHomeSections({maxPerSection=20, maxSections=30})` (`:144-176`):
`Future.wait` sobre targets; cada catálogo → una `StremioHomeSection` con
`title = catalogName`, `subtitle = addonName`; orden estable por `addonName`.

`_toContentItem(StremioMeta)` (`:178-216`): mapea type
(`series|tv|show|anime → series`, `channel|tv_channel → live`), **extrae `tmdbId`
del id** (`id.split(':').last` si contiene `:`, si no `int.tryParse(id)`) y lo
guarda en `extra['tmdbId']` junto con `stremioId`, `stremioType`, `logo`,
`source: 'stremio_catalog'`.

### 10.5 `StremioCollectionRepository` (304 líneas)

Prefs key **`'stremio_collections_v1'`** (`:18`).

Formato (docstring `:8-12`):

```json
{ "id", "title", "sources": [ { "provider":"addon", "addonId", "type", "catalogId", "genre" } ] }
```

- `importJson(raw)` (`:71-101`): acepta objeto, array, o `{collections:[…]}`
- `createFromCatalog(target)` (`:104-122`): id sintético `col_{hash}_{type}_{catalogId}`
- `resolveItems(collection, {maxPerSource=40})` (`:125-163`): por cada `source`
  con `provider=='addon'` y `manifestUrl`, consulta `StremioAddonRepository.fetchPage(...)`
  con `genre`, dedupe `type:id`
- `buildHomeSectionsFromCollections({maxPerSection=20})` (`:166-193`): solo
  colecciones con `showOnHome`
- `StremioCollectionSource.fromJson` (`:282-303`): si falta `manifestUrl` pero
  hay `addonId`, lo **resuelve buscando** en `StremioAddonRepository.instance.addons`
  por `manifest.id == addonId || manifestUrl.contains(addonId)`

### 10.6 `StreamBadgeRepository` (340 líneas) — "corrientes"

Prefs key **`'stream_badge_packs_v1'`** (`:35`).

Documenta **2 formatos** (`:6-30`):

**1) Elite / Nuvio**

```json
{ "filters": [ { "id", "name", "pattern": "(?i)\\bremux\\b", "imageURL", "isEnabled" } ],
  "groups": [ ... ] }
```

**2) Simple**

```json
{ "id", "name", "badges": [ { "match", "logo", "label" } ] }
```

`StreamBadgePack.fromRaw` (`:199-253`): si hay `filters`, filtra solo
`type == 'filter'` (`:224-227`), deriva el nombre del `sourceUrl` si falta, y
normaliza el id a kebab-case.

`StreamBadgeRule.matches(text)` (`:275-298`): traduce el flag inline `(?i)` a
`RegExp(..., caseSensitive:false)` y `(?-i)` a caseSensitive:true; ante regex
inválida hace fallback a `contains`.

`matchAllBadges(text)` (`:130-148`): recorre packs habilitados × badges, dedupe
por `logo ?? label`.

---

## 11. `AddonSourceAdapter` (330 líneas) — puente con el modal

### 11.1 `scrapeAll({...})` (`:33-138`)

Devuelve un `Stream<Map<String,dynamic>>` **progresivo**:

1. **Excluye** `isChannelsOnly` (`:48-50`) — los addons de TV en vivo no aparecen
   en el modal VOD
2. Construye un `ContentItem` sintético `tmdb:movie:N` / `tmdb:series:N` con
   `extra.tmdbId` (+ `season`/`episode`)
3. **Cola con N workers**: `limit` leído de prefs
   `servidores_concurrent_checks` (default 3, clamp 1..5, `_readMaxConcurrent`
   `:19-26`). Cada addon con **timeout 25 s**. Los resultados se emiten **en
   cuanto llegan**, sin esperar al más lento (`pump()` recursivo `:79-111`)
4. Al final añade los streams Stremio vía
   `mgr.sources.getStreams(sourceAddons: const [])` (`:114-129`)

### 11.2 `_toModalMap(StreamItem, AddonManifest)` (`:198-241`)

Traduce al vocabulario del modal:

```dart
{
  'servidor_url': ..., 'url': ...,
  'calidad': 'Digital',           // default
  'idioma' / 'lang': ...,
  'servidor_nombre' / 'nombre': ...,
  'fuente': ..., 'addon_id': ..., 'addon_name': ..., 'provider': ...,
  'is_hls': ..., 'headers': ..., 'infoHash': ...,
  'badge_logo': ..., 'badge_label': ..., 'badges': [...],
  // flags estilo extractores:
  'es_<addonId>': true, 'en_<addonId>': true, 'lat_<addonId>': true,
}
```

`_normalizeLang` (`:286-315`): `es_MX` / `es_ES` / `en_US` / `ja_JA`.
`_guessLang` (`:317-329`): infiere del título/provider.

**Consumido por** `source_aggregator.dart:1025`, que añade `map['es_addon']=true`,
normaliza idioma, limita a `maxPorFuente*4` y lanza verificación con el extractor
(`:1053-1069`).

---

## 12. `TmdbService` de addons (509 líneas)

Cliente TMDB **opcional**, alimentado por el propio addon:

```dart
// tmdb_service.dart:8
static const defaultKey = '';   // Placeholder; el addon debe traer apiKey en config
static const _base = 'https://api.themoviedb.org/3';
static const _img  = 'https://image.tmdb.org/t/p';
```

Endpoints: `movie/popular`, `movie/top_rated`, `tv/popular`, `tv/top_rated`,
`trending/all/week`, `genre/{movie|tv}/list`, `discover/{movie|tv}?with_genres=`,
`search/multi`, `{movie|tv}/{id}?append_to_response=…`, `tv/{id}/season/{n}`.

Tamaños de imagen: `w500` poster, `w1280` backdrop, `w300` season/still,
`w185` profile, `w342` similar, `w500` logo, `w300` collection.

`parseId(contentId)` (`:477-494`) parsea `tmdb:movie:123` y
`tmdb:series:123:1:5` → `ParsedTmdbId{type,id,season,episode}`.

---

## 13. UI de addons

| Fichero | Líneas | Rol |
|---|---|---|
| `screens/addons_hub_page.dart` | 986 | Hub raíz: 3 tiles (Addons actuales / Comunidad / Extensiones Nuvio) + switch `torrent_mkv_enabled`. Contiene `AddonsActualesPage` (`:375-524`) y `AddonsComunidadPage` (`:527-986`) |
| `screens/nuvio_packages_page.dart` | 1435 | Pegar manifest URL → intenta primero **Stremio catálogo**; si no tiene catálogos cae a **paquete Nuvio** |
| `screens/tv_nuvio_packages_page.dart` | 1435 | Catálogo precargado de **12 repos Nuvio oficiales** |
| `screens/addons_onboarding_page.dart` | 621 | Onboarding de primer arranque (`addons_onboarding_done_v1`) |
| `screens/nuvio_extensions_hub_page.dart` | 229 | Hub de las 4 extensiones: Corriente, Complementos, Plugins, Colecciones |
| `screens/tv_addons_comunidad_page.dart` | 602 | Comunidad TV con `CommunityService` |
| `screens/tv_addons_actuales_page.dart` | 445 | Addons instalados con foco D-pad |
| `screens/stremio_complementos_page.dart` | 185 | Instalar addon Stremio por URL de manifest |
| `screens/stremio_collections_page.dart` | 382 | Importar colección por JSON o crearla desde un catálogo |
| `screens/stream_corriente_page.dart` | 216 | Instalar packs de badges desde URL o pegar JSON |
| `widgets/custom_api_config_card.dart` | 315 | Tarjeta de "Mis APIs" |

### Catálogo precargado de Nuvio (`tv_nuvio_packages_page.dart:21-106`)

12 repos con su `manifestUrl` y `langs`:
`D3adlyRocket`, `yoruix`, `phisher98`, `michat88`, `Abinanthankv`, `nvmindl`,
`saimuelbr`, `realbestia1`, `Gowaru`, `mooncrown04`,
**`KennethJYS/Nuvio-Providers-Latino`**, `iamtoxiic`.

Entrada en Ajustes:
- Móvil: `settings_page.dart:1059-1067` → `AddonsHubPage(isTv: false)`
- TV: `tv_settings.dart:52,196` → `TvAddonsTab`

---

## 14. Resumen de tipos de addon (end-to-end)

| Tipo | Transporte | Recurso | Dónde se consume |
|---|---|---|---|
| **catalog** | Stremio `/catalog/{type}/{id}[/skip\|search\|genre].json` | `catalog` | `StremioAddonRepository.buildHomeSections` → Home |
| **meta** | Stremio `/meta/{type}/{id}.json` | `meta` | El URL builder lo soporta; no hay cliente meta dedicado |
| **stream** | Stremio `/stream/{type}/{id}.json` | `stream` | `StremioStreamService` → `SourceService.getStreams` → modal |
| **subtitles** | Tipos declarados en manifest | `subtitles` | El player usa **OpenSubtitles v3 propio**: `https://opensubtitles-v3.strem.io/subtitles/{movie\|series}/{imdb}[:s:e].json` |
| **source (JS)** | GitHub raw o URL, CommonJS `getStreams` | `stream` | `SourceService` → `JsAddonRuntime` |
| **function** | JS `getHome/search/discover/getMeta/getSubtitles/extract` | `subtitles\|rating\|posterTags\|top10\|mainSlider\|function` | `CatalogJsBridge` → `JsAddonRuntime.callCatalogMethod` |
| **badges** | JSON remoto (`filters` Elite o `badges` simple) | — | `StreamBadgeRepository` → `AddonSourceAdapter` |
| **canales TV** | Addon local `manifest.json`+`index.json`+`logo.png` | — | módulo `features/tvchanel/` |

---

## 15. Problemas del subsistema

1. **Doble carril sin converger**: un addon JS y uno Stremio con el mismo nombre
   se gestionan por caminos completamente distintos, sin normalización común.
2. **`rawCode` limitado a 80 KB** → los addons grandes deben re-descargar JS
   en cada arranque.
3. **cheerio sin fallback CDN**: si el asset falta, se rompe para siempre.
4. **Búsqueda secuencial en `getFirstStream`** → hasta 20 s por addon.
5. **El runtime ejecuta JS arbitrario de terceros** con red habilitada. Es el
   diseño, pero implica que un addon malicioso puede enviar cualquier petición.
6. **Sin validación de schemas**: los addons se parsean con `as Map<String,dynamic>`
   y `?.` por todas partes.