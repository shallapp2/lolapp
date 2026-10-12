# 05 · Extractores y resolución HLS

Subsistema que convierte una **URL de embed** (página del hosting) en una **URL
de reproducción real** (`.m3u8` / `.mp4`). Es la pieza más crítica del proyecto
y la que más souvent se rompe cuando un sitio cambia.

---

## 1. El algoritmo en 3 niveles

```
┌──────────────────────────────────────────────────────────────────────────┐
│ NIVEL 1 — PROVIDERS (lib/data/extractors/providers/*.dart)               │
│                                                                          │
│  Entrada: (tmdbId, isMovie, season, episode)                            │
│  · Triple fetch a TMDB con 3 idiomas → 3 títulos candidatos             │
│  · Construcción de slug (con y sin artículo inicial, con/sin año)       │
│  · GET a la página del sitio                                            │
│  · Parseo de iframes / data-* / __NEXT_DATA__                           │
│  Salida: List<Servidor> donde servidor_url = EMBED                     │
│          (hglink.to, filemoon.sx, fastream.to, callistanise.com, ...)   │
│                                                                          │
│  ⚠️ Algunos providers ya resuelven 1 salto extra:                       │
│     embed → player.php → URL real                                       │
└──────────────────────────────────────────────────────────────────────────┘
                                    │  servidor_url = embed
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ NIVEL 2 — NativeResolvers (Dart puro, HTTP + regex, ~0.3-2 s)           │
│                                                                          │
│  detectServer(url) por lista de espejos → resolver específico            │
│  Resuelve: voe · doodstream · streamwish · vidhide · filemoon           │
│            goodstream · lulustream · dropcdn · pixeldrain               │
│            buzzheavier · okru                                            │
│  Salida: StreamResult{url(m3u8/mp4), quality, headers, serverName}     │
└──────────────────────────────────────────────────────────────────────────┘
         │ si falla                              │ si falla
         ▼                                       ▼
┌────────────────────────────┐   ┌──────────────────────────────────────────┐
│ NIVEL 3a — WebView 1×1     │   │ NIVEL 3b — ExtractorPage (VISIBLE)       │
│ invisible, en Overlay      │   │ WebView a pantalla + spinner +           │
│ Inyecta MediaDetector.js:  │   │ "¡Stream encontrado! → Reproducir"     │
│  · hook de window.fetch    │   │ Cachea el MÉTODO por host en             │
│  · hook de XMLHttpRequest  │   │ SharedPreferences para la próxima        │
│  · wrap de Hls.js          │   └──────────────────────────────────────────┘
│  · jwplayer / videojs      │
│  · MutationObserver        │
│  · poll de performance     │
└────────────────────────────┘
                                    │
                                    ▼
        PlayerScreen (VideoPlayerController.networkUrl + httpHeaders)
              + HlsQualityParser para el selector de calidad
```

### Servicio principal

```dart
// lib/data/extractors/hls/hls_extractor.dart:858-922
static Future<String?> buscarFuente(BuildContext context, String servidorUrl,
                                    {Duration timeout = const Duration(seconds: 10)}) async {
  try {
    // 1) Resolvers nativos (6 s)
    final native = await NativeResolvers.resolve(servidorUrl, timeout: const Duration(seconds: 6));
    if (native != null && native.url.isNotEmpty) return native.url;
  } catch (_) {}

  // 2) WebView 1×1 invisible en el Overlay
  final overlay = Overlay.maybeOf(context, rootOverlay: true);
  if (overlay == null) return null;
  final completer = Completer<String?>();
  late OverlayEntry entry; var resuelto = false;
  entry = OverlayEntry(builder: (_) => Positioned(
    left: -5, top: -5, width: 1, height: 1,
    child: IgnorePointer(child: Opacity(opacity: 0.0, child: _HiddenProbe(...))),
  ));
  overlay.insert(entry);
  return completer.future;
}
```

---

## 2. `hls/hls_extractor.dart` (1196 líneas)

### 2.1 `StreamResult`

```dart
// hls_extractor.dart:20-34
class StreamResult {
  final String url;
  final String quality;
  final Map<String, String> headers;
  final String serverName;
  final bool verified;
}
```

### 2.2 Detección de host por listas de espejos

```dart
// hls_extractor.dart:83-101
static String detectServer(String url) {
  final s = url.toLowerCase();
  if (_isMirror(s, _voeMirrors))         return 'voe';
  if (_isMirror(s, _streamwishMirrors) || s.contains('filelions')) return 'streamwish';
  if (_isMirror(s, _filemoonMirrors))    return 'filemoon';
  if (_isMirror(s, _vidhideMirrors))     return 'vidhide';
  if (_isMirror(s, _doodMirrors))        return 'doodstream';
  if (_isMirror(s, _goodstreamMirrors))  return 'goodstream';
  if (s.contains('vimeos') || s.contains('vms.sh')) return 'vimeos';
  if (_isMirror(s, _luluMirrors))        return 'lulustream';
  if (_isMirror(s, _dropcdnMirrors))     return 'dropcdn';
  if (s.contains('pixeldrain'))          return 'pixeldrain';
  if (s.contains('buzzheavier') || s.contains('bzh.sh')) return 'buzzheavier';
  if (s.contains('ok.ru') || s.contains('okru'))      return 'okru';
  if (s.contains('vidsrc') || s.contains('moviesapi')) return 'vidsrc';
  return 'unknown';
}
```

`_doodMirrors` tiene **30 entradas** (`:164-196`):
`dood.li`, `dood.la`, `ds2video.com`, `dood.yt`, `dood.to`, `dood.pm`,
`vidply.com`, `do7go.com`, `doply.net`, `dsvplay.com`, …

⚠️ Este mismo patrón está **duplicado 3 veces** en:
- `lib/data/extractors/hls/hls_extractor.dart`
- `lib/features/player/data/native_resolvers.dart` (737 L)
- `lib/features/player/data/extractor.dart` (1325 L)

### 2.3 Dispatch

```dart
// hls_extractor.dart:214-247
static Future<StreamResult?> resolve(String url, {Duration timeout = const Duration(seconds: 8)}) async {
  final server = detectServer(url);
  switch (server) {
    case 'voe':         return await _resolveVoe(url).timeout(timeout);
    case 'doodstream':  return await _resolveDoodstream(url).timeout(timeout);
    case 'okru':        return await _resolveOkRu(url).timeout(timeout);
    // ...
    default: return null;   // filemoon, vimeos, vidsrc NO tienen resolver → WebView
  }
}
```

---

## 3. Los resolvers nativos — detalle criptográfico

### 3.1 VOE — ROT13 + base64 + Caesar(-3) + reverse + base64 ⭐

`hls_extractor.dart:250-312`. Cuatro capas de ofuscación:

```dart
// 1) Si es un stub que redirige, resolver recursivamente
if (html.contains('window.location.href') && html.length < 2000) {
  final m = _reLocationHref.firstMatch(html);
  if (m != null) return _resolveVoe(m.group(1)!);
}

// 2) Extraer <script type="application/json">
// 3) ROT13
var decoded = encText.replaceAllMapped(RegExp(r'[a-zA-Z]'), (m) {
  final c = m.group(0)!; final code = c.codeUnitAt(0);
  final limit = c.toUpperCase() == c ? 90 : 122;      // 65-90 / 97-122
  final shifted = code + 13;
  return String.fromCharCode(limit >= shifted ? shifted : shifted - 26);
});

// 4) Quitar ruido
for (final n in ['@$', '^^', '~@', '%?', '*~', '!!', '#&']) decoded = decoded.replaceAll(n, '');

// 5) base64 → Caesar(-3) → reverse → base64 → JSON
final b64_1      = utf8.decode(base64Decode(_padB64(decoded)));
final shifted    = String.fromCharCodes(b64_1.codeUnits.map((c) => c - 3));
final reversed   = shifted.split('').reversed.join();
final decrypted  = utf8.decode(base64Decode(_padB64(reversed)));
final data = jsonDecode(decrypted);

if (data is Map && data['source'] != null) {
  return StreamResult(url: data['source'].toString(), quality: '1080p', serverName: 'VOE',
                      headers: {'User-Agent': _ua, 'Referer': url});
}
// 6) fallback: primer .m3u8 entrecomillado
```

Helper de padding (`:798-801`):

```dart
final pad = (4 - s.length % 4) % 4;
return s + ('=' * pad);
```

### 3.2 DoodStream — endpoint `/pass_md5/` + token + expiry

`hls_extractor.dart:315-357`

```dart
// Normaliza /d/ o /f/ → /e/
if (!embedUrl.contains('/e/')) embedUrl = embedUrl.replaceAll(RegExp(r'/(d|f)/'), '/e/');

// $.get('/pass_md5/xxx','yyy')
final match = _rePassMd5.firstMatch(res.body);
final passUrl = '${Uri.parse(embedUrl).origin}${match.group(1)}';
final videoBase = (await http.get(passUrl)).body.trim();

// Firma requerida por Dood: 10 chars aleatorios + token + expiry en ms
const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
final rnd = List.generate(10,
  (_) => chars[DateTime.now().microsecondsSinceEpoch % chars.length]).join();
final finalUrl = '$videoBase$rnd?token=${match.group(2)}'
                '&expiry=${DateTime.now().millisecondsSinceEpoch}';
```

### 3.3 StreamWish — barrido de 8 espejos + endpoint `/dl`

`hls_extractor.dart:360-446`

```dart
final rawId = url.split('/').last.replaceAll(RegExp(r'\.html$'), '');
final mirrors = [
  'https://hanerix.com/e/$rawId',   'https://embedwish.com/e/$rawId',
  'https://hglink.to/e/$rawId',      url,
  'https://streamwish.to/e/$rawId',  'https://awish.pro/e/$rawId',
  'https://strwish.com/e/$rawId',    'https://wishfast.top/e/$rawId',
];

for (final mirror in mirrors) {                       // 4 s timeout por espejo
  final hashMatch = RegExp(r'[0-9a-f]{32}', caseSensitive: false).firstMatch(html);
  if (hashMatch != null) {
    final dlUrl = '$mirrorOrigin/dl?op=view&file_code=$rawId&hash=$hash'
                + '&embed=1&referer=&adb=1&hls4=1';
    final dlResp = await http.get(Uri.parse(dlUrl),
      headers: {'X-Requested-With': 'XMLHttpRequest'});
    final m = _reM3u8Any.firstMatch(dlResp.body);
    if (m != null) m3u8Url = m.group(0);
  }
  if (m3u8Url == null) { /* desempaquetar P.A.C.K.E.R */ }
  if (m3u8Url == null) { final f = _reFile.firstMatch(html); ... }
}
```

### 3.4 VidHide — desempaquetado P.A.C.K.E.R en base-36

`hls_extractor.dart:449-514` y desempacador en `:824-851`

```dart
final packedMatch = _rePackerVidHide.firstMatch(html);
if (packedMatch != null) {
  final unpacked = _unpackVidHide(packedMatch.group(0)!);
  final hls = RegExp(r'"hls[24]"\s*:\s*"([^"]+)"').firstMatch(unpacked);
  final label = RegExp(r'\{label\s*:\s*"([^"]+)').firstMatch(unpacked)
             ?? RegExp(r'name\s*:\s*"([^"]+)').firstMatch(unpacked);
  if (label != null) {
    quality = label.group(1)!.toLowerCase().contains('p')
      ? label.group(1)! : '${label.group(1)}p';
  }
}
// Añade referer si falta
if (!finalUrl.contains('referer=')) {
  finalUrl += '${finalUrl.contains('?') ? '&' : '?'}referer=embed69.org';
}
```

El desempacador genérico:

```dart
// hls_extractor.dart:803-822
static String _unpackEval(String payload, int radix, List<String> symtab) {
  const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  int unbase(String str) {
    var r = 0;
    for (var i = 0; i < str.length; i++) {
      final pos = chars.indexOf(str[i]);
      if (pos == -1) return -1;
      r = r * radix + pos;
    }
    return r;
  }
  return payload.replaceAllMapped(RegExp(r'\b([0-9a-zA-Z]+)\b'), (m) {
    final idx = unbase(m.group(1)!);
    if (idx < 0 || idx >= symtab.length) return m.group(0)!;
    final val = symtab[idx];
    return val.isNotEmpty ? val : m.group(0)!;
  });
}
```

### 3.5 OK.ru — `data-options` → HTML-decode → JSON → `flashvars.metadata`

`hls_extractor.dart:706-783`. Único resolver OK.ru del repo.

```dart
final m = RegExp(r'data-options="([^"]*)"').firstMatch(html);
final raw = _htmlDecode(m.group(1)!);   // &quot; &#34; &#39; &apos; &lt; &gt; &amp;
final opt = jsonDecode(raw);

final md = (opt['flashvars'] is Map) ? opt['flashvars']['metadata'] : null;
final meta = md is Map ? md : jsonDecode(md);   // puede venir como String JSON

final hls = meta['hlsMasterPlaylistUrl'] ?? meta['hlsManifestUrl'] ?? meta['ondemandHls'];
if (hls is String && hls.isNotEmpty) {
  return StreamResult(url: hls, quality: 'auto', ...);
}
// MP4 progresivos con mapa de calidad
static const _okruQuality = {
  'mobile':'144p','lowest':'240p','low':'360p','sd':'480p',
  'hd':'720p','full':'1080p','quad':'1440p','ultra':'2160p',
};
```

### 3.6 Buzzheavier — `HEAD` + header `hx-redirect`

`hls_extractor.dart:603-642`

```dart
final head = await http.head(Uri.parse('$cleanUrl/download'),
  headers: {'hx-current-url': cleanUrl, 'hx-request': 'true'});
final hx = head.headers['hx-redirect'];
if (hx != null && hx.isNotEmpty) {
  if (hx.startsWith('/dl/')) finalUrl = 'https://$domain$hx';
  return StreamResult(url: '$finalUrl#.mp4', ...);
}
```

### 3.7 DropCDN

`hls_extractor.dart:645-692`: normaliza a `dr0pstream.com/e/{code}` y selecciona
`master.m3u8`.

---

## 4. El probe WebView (Nivel 3)

### 4.1 `_HiddenProbe` + `_injectPowerfulMediaDetector()`

`hls_extractor.dart:1049-1180`. JS inyectado en un WebView de **1×1 píxel** con
`opacity: 0` e `IgnorePointer`, dentro de un `OverlayEntry`.

```js
// Hook de fetch
if (!window.__mdOrigFetch) window.__mdOrigFetch = window.fetch;
window.fetch = function (...args) {
  const url = typeof args[0] === 'string' ? args[0] : (args[0]?.url || '');
  if (url) sendUrl(url);
  return window.__mdOrigFetch.apply(this, args);
};

// Hook de XMLHttpRequest
XMLHttpRequest.prototype.open = function (method, url) {
  if (!stopped && url) sendUrl(url);
  window.__mdOrigXhrOpen.apply(this, arguments);
};

// Wrap del constructor Hls.js
if (window.Hls && Hls.isSupported()) {
  window.Hls = function (config) {
    const hls = new OriginalHls(config);
    hls.on(Hls.Events.MANIFEST_PARSED, (e, d) =>
      d.levels?.forEach(l => sendUrl(l.url)));
    hls.on(Hls.Events.LEVEL_LOADED, (e, d) =>
      d.details?.fragments?.forEach(f => sendUrl(f.url)));
  };
}

// jwplayer + videojs
window.jwplayer().getPlaylist()...
window.videojs.getAllPlayers()...

// Scan periódico cada 3500 ms
const combinedCheck = () => {
  performance.getEntriesByType('resource').forEach(e => { ... sendUrl(e.name) });
  document.querySelectorAll('video, source, [src], [href], iframe')
    .forEach(el => sendUrl(el.src || el.href || ...));
};

// MutationObserver sobre el DOM
window._mutationObserver = new MutationObserver(...);

// Limpieza
window.__mdCleanup = function () { /* restaura fetch/XHR, desconecta MO, clearInterval */ };
```

### 4.2 Canal de salida JS→Dart

```dart
// hls_extractor.dart:959-968
webView.addJavaScriptChannel('MediaDetector', (message) {
  // message.message = URL absoluta detectada
});
```

### 4.3 Criterio de aceptación

`hls_extractor.dart:998-1007, 1019-1027`:
- Extensión válida: `.m3u8` | `.mp4` | `.ts` | `.m4s`
- **Preferencia** por `master.m3u8` / `playlist.m3u8` / `index.m3u8`
- URL absoluta vía `Uri.resolve`
- Deduplicada en `Set<String>`
- **`_finalizar` en el primer `.m3u8` / `.mp4`** encontrado

> Este mismo JS está **duplicado** en
> `lib/data/extractors/providers/content/extractor_hls.dart:190-324` (fichero
> muerto).

---

## 5. `providers/content/extractor_hls.dart` — variante muerta

341 líneas. **Mismo doc-comment** que `hls_extractor.dart` pero **sin
`NativeResolvers`**: solo el WebView 1×1.

```dart
// content/extractor_hls.dart:1-12
// Extrae la fuente (m3u8/mp4) de un servidor SIN mostrar UI al usuario.
// Usa exactamente la misma lógica de detección que ExtractorPage ...
```

`grep 'providers/content/'` → **0 resultados**. Es código muerto; además
`content/tioplus.dart` es idéntico byte a byte a
`providers/tioplus_extractor.dart`.

---

## 6. Los 13 providers de Nivel 1

### 6.1 `vidsrc_extractor.dart` (305 L) — `VidSrcService`

**Solo encuentra embeds**; el HLS lo resuelve `ExtractorHlsService`.

```dart
// :99-119 — dominios × 2 formatos de ruta, probados en orden
static List<String> _buildEmbedCandidates(int tmdbId, bool isMovie, int s, int e) {
  if (isMovie) return [
    'https://vidsrc.me/embed/movie/$tmdbId', 'https://vidsrc.me/embed/$tmdbId',
    'https://vidsrc.to/embed/movie/$tmdbId', 'https://vidsrc.xyz/embed/movie/$tmdbId',
  ];
  return [
    'https://vidsrc.me/embed/tv/$tmdbId/$s-$e', 'https://vidsrc.me/embed/$tmdbId/$s-$e', ...
  ];
}
```

**Segundo salto** (`:41-57`): `data-hash` → `/rcp/$hash` + `_resolveProrcpFromRcp()`.

**Base dominio dinámico** (`:121-148`) — se reescribe con el host del primer
iframe encontrado:

```dart
static void _applyBaseDom(String src) {
  final u = Uri.parse(src);
  _baseDom = '${u.scheme}://${u.host}';
}
```

Parser con 4 alternativas de comillas (`:150-204`) y headers fijos
(`Referer: https://vidsrc.me/`, `Accept-Language: en-US`).

### 6.2 `cuevana_extractor.dart` (488 L) — `CuevanaService`

```dart
static const _kTmdbKey = 'a2d9bbed370d9f678e34006f8750a5a5';
static const _kBase = 'https://wv3.cuevana3.eu';
```

**Lista blanca de cyberlockers** (`:20-30`) — mismo filtro que el PHP original:

```dart
static const _kAllowed = ['streamwish','vidhide','filelions','vidhidepro',
  'streamwish.to','vidhidepro.com','filelions.com','filelions.to',' '];
```

**Reescritura de dominios** (`:33-37`):

```dart
static const _kDomainMap = {
  'streamwish.to':   'hgplaycdn.com',
  'vidhidepro.com':  'callistanise.com',
  'filelions.to':    'callistanise.com',
};
```

**Triple fetch TMDB** (es-MX, es-ES, en-US) → 3 títulos candidatos + año
(`:119-138`). Episodios con 3 títulos × 2 formatos de slug (`:166-182`):

```
$_kBase/episodio/$slug-temporada-$season-episodio-$episode
$_kBase/episodio/$slug-${tmdb.id}-temporada-$season-episodio-$episode
```

**Validación por contenido, no por HTTP code** (`:234-247`):

```dart
final needle = movie ? '"thisMovie"' : '"thisSerie"';
if (html.contains('__NEXT_DATA__') && html.contains(needle)) {
  return _FoundPage(url: url, html: html);
}
```

**`_resolvePlayer`** (`:306-334`) — sustituye al `player.php` del sitio original:

```dart
final m1 = RegExp(r"var url = '([^']+)'").firstMatch(html);
if (m1 != null) videoUrl = m1.group(1);
// fallback genérico de embeds
final m3 = RegExp(r'''(?:file|src|source)\s*[:=]\s*["'](https?://[^"']+\.m3u8[^"']*)["']''',
                  caseSensitive: false).firstMatch(html);
return _mapDomain(videoUrl!);
```

Anti-rate-limit: `await Future.delayed(Duration(milliseconds: 30))` entre emisiones (`:89`).

### 6.3 `tioplus_extractor.dart` (497 L) — `TioplusService`

**3 escalones de resolución de URL** (`:67-135`):

1. Candidatos por slug:
   `/pelicula/$slug` | `/serie/$slug/season/$s/episode/$e` | `/anime/...`
   + variante **sin artículo inicial** (`el/la/los/las/the/a/an`) (`:214-227`)
2. Fallback: `/api/search/{q}` con prioridad `{'pelicula':0,'serie':1,'anime':2}` (`:308-311`)
3. Si falló serie → fuerza `/anime/$slug/season/$s/episode/$e`

**Validación doble** (`:341-343`) — ambas condiciones necesarias:

```dart
static bool _hasServers(String html) =>
    html.contains('data-server=') && html.contains('subselect');
```

**Resolución del token `data-server`** (`:347-388`):

```dart
final re = RegExp(r'''<li[^>]*data-server=["']([^"']+)["'][^>]*>.*?<span>([^<]+)</span>''',
                  caseSensitive: false, dotAll: true);
final playerPath = base64Encode(utf8.encode(dataServer));       // ← BASE64
final playerUrl  = '$_kTioplusBase/player/$playerPath';
final playerHtml = await _fetchPage(playerUrl,
  extraHeaders: {'Referer': referer, 'Accept': 'text/html'});
var embed = _extractRedirect(playerHtml);    // window.location.href | location.href
embed = _applyHostMap(embed);                // vidhideplus.com → callistanise.com
```

**`_guessIdioma` por el NOMBRE del servidor** (`:417-430`):

```dart
if (n.contains('castellano') || ...) return 'es_ES';
if (n.contains('latino') || n.contains('lat')) return 'es_MX';
if (n.contains('sub') || n.contains('english')) return 'en_US';
return 'es_MX';   // TioPlus es latino por defecto
```

Reescritura de host (`:11-13`):

```dart
const _kHostMap = <String, String>{ 'vidhideplus.com': 'callistanise.com' };
```

### 6.4 `cinecalidad_extractor.dart` (340 L) — `CinecalidadService`

```dart
// :7
static const _kAllowedServers = ['Vimeos', 'Hlswish', 'voe', 'Videoapp'];
// :11
static const _kBase = 'https://www.cinecalidad.am';
```

**Extracción + decodificación base64 del token "zopass"** (`:228-278`):

```dart
final panelRe = RegExp(r'<div id="panel_online".*?<ul class="linklist">(.*?)</ul>',
                       dotAll: true);
for (final m in liRe.allMatches(panelContent)) {
  final serverName = (m.group(2) ?? '').trim().replaceAll(RegExp(r'<span.*'), '');
  if (!_kAllowedServers.contains(serverName)) continue;
  var realUrl = m.group(1) ?? '';
  if (realUrl.startsWith('/zopass/?zopass=')) {
    final uri = Uri.parse(realUrl.startsWith('http') ? realUrl : 'https://dummy$realUrl');
    final zopass = uri.queryParameters['zopass'];
    if (zopass != null && zopass.isNotEmpty) {
      realUrl = utf8.decode(base64Decode(zopass));     // ← BASE64
    }
  }
  result.add(_ExtractedLink(server: serverName, url: realUrl, ...));
}
```

Validación (`:224-226`): `html.contains('panel_online') && html.contains('data-option')`.
Idioma **siempre `es_MX`** (el sitio es latino).

### 6.5 `unlimplay_extractor.dart` (184 L) — `UnlimplayService`

```dart
// :35  whitelist
static const _allowed = ['streamwish', 'vidhide', 'filelions'];

// :40-56  reescritura de host
if (host == 'hglink.to' || host == 'streamwish.to')            newHost = 'vibuxer.com';
else if (host == 'filelions.to' || host == 'minochinos.com') newHost = 'callistanise.com';

// :65-67  embed
https://unlimplay.com/f/embed/movie/$id
https://unlimplay.com/f/embed/tv/$id/$s/$e
```

**Dos fuentes de datos** como JSON embebido (`:87-103`):

```dart
final embedsMatch = RegExp(r'const\s+EMBEDS\s*=\s*(\{.*?\});', dotAll: true).firstMatch(html);
final finalMatch  = RegExp(r'finalizePlayer\s*\(\s*(\{.*?\})\s*\)\s*;', dotAll: true).firstMatch(html);
if (sources.isEmpty) throw Exception('No se encontró EMBEDS ni finalizePlayer');
```

Desambiguación de nombres duplicados (`:109-139`):

```dart
while (bucket.containsKey('$nameKey $n')) n++;
bucket['$nameKey $n'] = url;
```

Orden de idioma (`:141-145`): latino → español → espanol → subtitulado → resto.

### 6.6 `pelisplus_extractor.dart` (464 L) — `PelisPlusService`

```dart
// :4-5  Solo encuentra embeds (NO resuelve HLS/m3u8). ← delega en ExtractorHlsService
static const _base = 'https://www.pelisplushd.la';
static const _tmdbKey = '439c478a771f35c05022f9feabcca01c';
```

**3 títulos × 3 idiomas + variantes** sin `:`, sin puntuación y sin `(AAAA)`
(`:88-131`).

**Búsqueda con *fuzzy match*** (`:137-177`):

```dart
final matchCount = qWords.where(tWords.contains).length;
return (matchCount / qWords.length) >= 0.8;   // ≥ 80 % de palabras deben coincidir
```

**TRES métodos de extracción en cascada** (`:276-349`):

1. `<li data-url="..." data-name="...">`
2. `<span url="...">`
3. `var options = { ... }` → `_parseJsObject` (JSON o parser regex)

`_detectLanguage` + `_isLanguageSupported` (solo latino/castellano) →
`_toCanonicalIdioma`: `es_ES` | `en_US` | `es_MX`.

### 6.7 `poseidon_extractor.dart` (465 L) — clon de Cuevana

```dart
// :13
static const _kBase = 'https://www.poseidonhd2.co';
// :20-26  mapa de dominios MÁS amplio que Cuevana
static const _kDomainMap = {
  'streamwish.to': 'hgplaycdn.com', 'vidhidepro.com': 'callistanise.com',
  'filelions.to': 'callistanise.com', 'voe.sx': 'eugenemakedraw.com',
  'doodstream.com': 'playmogo.com',
};
```

Rutas **con TMDB-ID en el path** (`:175-214`), a diferencia de Cuevana:

```
/pelicula/{tmdbId}/{slug}   /pelicula/{tmdbId}/{slug}-{year}
/serie/{tmdbId}/{slug}/temporada/{s}/episodio/{e}
```

**Diferencia clave**: `_resolvePlayer` **NO aplica whitelist** → genera más
servidores que Cuevana.

### 6.8 `pelispedia` / `seriesmetro` / `smartpelis` — ⭐ clones

Verificado con `diff`: son **el mismo archivo** con solo dos diferencias.

| Provider | `_kBase` |
|---|---|
| Pelispedia | `https://pelispedia.is` (`pelispedia_extractor.dart:14`) |
| SeriesMetro | `https://www3.seriesmetro.net` (`seriesmetro_extractor.dart:13`) |
| SmartPelis | `https://smartpelis.tv` (`smartpelis_extractor.dart:14`) |

Estructura HTML que parsean (documentada en `pelispedia_extractor.dart:192-202`):

```html
<aside id="aa-options">
  <div id="options-0" class="video aa-tb hdd on">
    <iframe src="...?trembed=0&trid=..." data-src="...?trembed=0&trid=...">
  </div>
</aside>
<a href="#options-0">OPCIÓN <span>1</span> <span class="server">-Latino</span></a>
```

**3 métodos** (`:209-309`):

1. Botones `href="#options-N"`: número de opción + idioma

   ```dart
   final numM = RegExp(r'''<span(?![^>]*class=["'][^"']*server)[^>]*>\s*(\d+)\s*</span>''')
                 .firstMatch(inner);
   final langM = RegExp(r'''<span[^>]*class=["'][^"']*server[^"']*["'][^>]*>\s*-?\s*([^<]+?)\s*</span>''')
                 .firstMatch(inner);
   if (language.toUpperCase() == 'VOSE') language = 'Subtitulado';  // voseo = sub
   ```

2. Bloques `<div id="options?-N|video-N|opt-N">` → `iframe` (preferencia `data-src`)
3. Fallback: todos los iframes de `aa-options` en orden

**`_resolveEmbed` — salto extraído en servidor** (`:314-358`):

```dart
final host = Uri.tryParse(url)?.host.toLowerCase() ?? '';
if (host.isNotEmpty && !host.contains('pelispedia')) return url;  // ya es externo
final embedHtml = await _fetch(url);
final inner = _extractIframeSrc(embedHtml);
if (inner != null) {
  if (Uri.parse(inner).host.contains('pelispedia')) {   // un nivel más de iframe
    final deeper = await _fetch(inner);
    final f = _extractIframeSrc(deeper);
    if (f != null && f.isNotEmpty) return f;
  }
  return inner;
}
```

`_languageToCode`: castellano/españa/spain → `es_ES`; inglés/sub/vose → `en_US`;
japon → `ja_JA`; resto → `es_MX`.

> UA `Chrome/154.0.0.0` — el más "moderno" de todos los providers.

### 6.9 `embed69_extractor.dart` (797 L) — el más complejo ⭐

**Proof-of-Work + AES-256-CBC**.

```dart
import 'package:crypto/crypto.dart';       // SHA-256
import 'package:encrypt/encrypt.dart' as enc;  // AES

// :12-20
static const _kBaseUrlSeriesKao = 'https://serieskao.top/vidurl/';
static const _kBaseUrlXupalace   = 'https://xupalace.org/video/';
static const _kTmdbApiKey = 'a2d9bbed370d9f678e34006f8750a5a5';
static const _kOmdbApiKey = '5b0e8a3e';   // fallback cuando TMDB no tiene imdb_id
static const _kOmdbBase   = 'http://www.omdbapi.com/';
```

**IMDb → 5 variantes de ID para series** (`:45-91`):

```
'$imdbId-${s}x$e2'  '$imdbId-${s}x$e'  '$imdbId-$s2x$e2'
'$imdbId-$s-$e2'     '$imdbId-$s-$e'
```

→ prueba SeriesKao y luego Xupalace por cada variante.

**`_getImdbFromTmdb`: 4 escalones** (`:104-135`):

1. `/{type}/{id}/external_ids`
2. `?append_to_response=external_ids`
3. (solo TV) `/tv/{id}`
4. OMDb por título + año (`_getImdbFromOmdbByTitle`, `:184-238`)

`_normalizeImdb` (`:241-251`): `/^tt\d+$/`; dígitos sueltos → `tt$s`;
extrae `(tt\d+)` embebido.

**Proof-of-Work** (`:421-438`):

```dart
static int? _solvePow(String challenge, int difficulty) {
  final prefix = '0' * difficulty;
  for (var nonce = 0; nonce < 1000000; nonce++) {
    final hash = _sha256('$challenge$nonce');
    if (hash.startsWith(prefix)) return nonce;
  }
  return null;
}
static String _sha256(String input) => sha256.convert(utf8.encode(input)).toString();
```

Lectura de constantes (`:390-417`):

```dart
_extractDataLink(html);     // 'let dataLink = [...]'  (3 patrones)
_extractPowConstants(html); // POW_CHALLENGE, POW_SALT, POW_DIFFICULTY
```

**Clave AES = BYTES BINARIOS del SHA-256** (no hex), igual que
`hash(...,'raw',true)` en PHP (`:440-450`):

```dart
static Uint8List _generateAesKey(String challenge, int nonce, String salt) =>
    Uint8List.fromList(sha256.convert(utf8.encode('$challenge$nonce$salt')).bytes);
```

**Descifrado AES-256-CBC real** (`:463-488`) — con un FIX documentado en el
código (*"antes solo hacía base64Decode"*):

```dart
static String? _decryptAES(String encryptedBase64, Uint8List aesKey) {
  final raw = base64Decode(encryptedBase64);
  if (raw.length <= 16) return null;
  final ivBytes     = raw.sublist(0, 16);     // IV = primeros 16 bytes
  final cipherBytes = raw.sublist(16);        // resto = ciphertext (PKCS7)
  final key = enc.Key(aesKey);
  final iv  = enc.IV(Uint8List.fromList(ivBytes));
  final encrypter = enc.Encrypter(enc.AES(key, mode: enc.AESMode.cbc));
  final text = utf8.decode(
    encrypter.decryptBytes(enc.Encrypted(Uint8List.fromList(cipherBytes)), iv: iv),
    allowMalformed: true);
  if (text.contains('http')) return text;     // sanity check
  return null;
}
```

**Post-proceso y filtros** (`:492-544`):

```dart
if (servidoresFiltrados.contains(servidor)) continue;    // ['voe','rapidvideo']
urlReal = urlReal.replaceAll('`', '')
                  .replaceAll('hglink.to', 'vibuxer.com')
                  .replaceAll('filelions.to', 'callistanise.com');
```

**XUPALACE**: parsing por bloques `<li>` independiente del orden de atributos
(`:558-658`):

```dart
final urlRe  = RegExp(r'''go_to_playerVast\(\s*['"]([^'"]+)['"]''');
final langRe = RegExp(r'''data-lang=["'](\d+)["']''');
final mapLang = {'0':'es_MX', '1':'subtitulado', '2':'en_US', '3':'castellano'};
```

**Segunda whitelist: SOLO streamwish y vidhide** (`:678-690`):

```dart
final esStreamwish = servidor == 'streamwish' || url.contains('hglink.to')
                  || url.contains('vibuxer.com');
final esVidhide = servidor == 'vidhide' || url.contains('vidhide')
               || url.contains('filelions') || url.contains('callistanise');
if (!esStreamwish && !esVidhide) continue;
```

Headers completos por origen (`:329-363`):

```dart
headers: {
  'X-Requested-With': 'XMLHttpRequest',
  'Accept-Encoding': 'identity',
  'Connection': 'keep-alive',
  ...(source == 'serieskao'
      ? {'Referer': 'https://serieskao.top/', 'Origin': 'https://serieskao.top'}
      : {'Referer': 'https://xupalace.org/', 'Origin': 'https://xupalace.org'}),
}
```

### 6.10 `hackstore_extractor.dart` (372 L) — API REST de WordPress

```dart
// :12-13
static const _kBase = 'https://hackstore.mx';
static const _tmdbKey = '439c478a771f35c05022f9feabcca01c';
```

Pipeline de 6 pasos (`:28-60`):

1. `_getTmdbTitles` → 3 idiomas + `alternative_titles` + `translations` → hasta 15 slugs
2. `_findContentId` → `GET /wp-api/v1/single/{movies|tvshows}?slug=$slug&postType=...`
3. `_findEpisodeId` → `GET /wp-api/v1/single/episodes/list?_id=$serie&season=$s&page=1&postsPerPage=200`
4. `_getPlayers` → `GET /wp-api/v1/player?postId=$id` → `data.embeds` (máx 15)

`_normalizeSlug` (`:188-208`): quita acentos, no-alnum → espacio → guiones.

**Filtros** (`:294-308`):

```dart
// descarta sub/vose/eng/espana
if (lang.contains('sub') || lang.contains('vose') ||
    lang.contains('eng') || lang.contains('espana')) continue;
// descarta 'la.movie' y reescribe voe.sx
if (rawUrl.isEmpty || rawUrl.contains('la.movie')) continue;
rawUrl = rawUrl.replaceAll('voe.sx', 'eugenemakedraw.com');
```

### 6.11 `cinesrc_extractor.dart` (302 L) — sin HTTP ⭐

**El único provider que no hace ninguna petición**: emite 6 embeds fijos
construidos por plantilla.

```dart
// :32-106
buildEmbedUrl    → 'https://cinesrc.st/embed/{movie|tv}/$id[?s=&e=]?color=%2300ff66&autoplay=true&autonext=true&back=close&prioritize=true'
buildVideoAppUrl → 'https://modlyo.com/embed.php?url=https://videoapp.mov/e/{movie|tv}/...'
buildVidSrcUrl   → 'https://modlyo.com/embed.php?url=https://vidsrc.sh/embed/...?ds_lang=es'
buildVsEmbedUrl  → 'https://vsembed.ru/embed/{movie|tv}?tmdb=$id&color=e600e6&ds_lang=es?auto=1'
buildZxcPrimeUrl → 'https://player.zxcprime.xyz/player/{movie|tv}/...?...&dubLang=esla'
buildVimeusUrl   → 'https://vimeus.com/e/{movie|serie}?tmdb=$id&view_key=KUDU-EYQ76rbZwNu7hun16C4HX0K6qy77hZw4CaveiI&theme=minimal'
```

Orden de emisión (`:108-187`):

```
CineSRC(es_MX) → VideoApp(es_MX) → VidSrc(es_MX)
               → VsEmbed(en_US) → ZxcPrime(en_US) → Vimeus(es_MX)
```

Por eso el agregador lo marca `webview_only = true` y `prioridad = 999`
(`source_aggregator.dart:903-907`).

### 6.12 `fuegocine_extractor.dart` (68 L) — el más simple

```dart
// :6
static const String _apiUrl = 'https://www.modlyo.com/api/servidores.php';

// :21-26
Uri.parse(_apiUrl).replace(queryParameters: {
  'tmdbId': '...', 'type': isMovie ? 'movie' : 'tv',
  if (!isMovie) 'season': '$season', if (!isMovie) 'episode': '$episode',
});

// :34-38  contrato de respuesta
{ success: true, streams: [{ servidor_url, servidor_nombre, calidad, idioma }] }
```

---

## 7. Uso de librerías criptográficas

| Paquete | Dónde | Nota |
|---|---|---|
| **`crypto` (Dart)** + **`encrypt` (AES-CBC)** | `embed69_extractor.dart:5-6` | SHA-256 PoW y AES-256-CBC. Sustituye a crypto-js para el único cifrado real |
| **`crypto-js` (JS asset)** | `js_runtime.dart:17` | Carga **perezosa**: solo si el addon contiene `'crypto-js'`/`'CryptoJS'`/`require('crypto` (`:322-333`). Fallback CDN: cdnjs + jsdelivr. Inyectado como `globalThis.CryptoJS` |
| **`cheerio` (JS asset)** | `js_runtime.dart:18` | Carga perezosa si el addon contiene `'cheerio'` (`:326,334-339`). Expuesto como `globalThis.__cheerio`. **Sin fallback CDN** |
| **`dart_cast`** | Solo `cast_manager.dart:2` y `cast_button.dart:3` | Casting a Chromecast, **fuera** del pipeline de extracción |
| `base64` / `utf8` | `hls_extractor.dart:285-288` (VOE), `cinecalidad:261`, `tioplus:367`, `embed69:465` | 4 usos criptográficos distintos |
| `cheerio` (Dart) / CSS selectors | ❌ **No se usa en absoluto** en `lib/data/**` | Todo el scraping es `RegExp` sobre HTML crudo |

---

## 8. `ExtractorPage` — el flujo manual visible

`lib/features/player/data/extractor.dart:748-1325`.

**Caché de método por host** en `SharedPreferences` (`:780-826`):

```dart
static const String _kHostCacheKey   = 'extractor_host_methods';
static const String _kMethodWebview = 'webview_media_detector';
static const String _kMethodNative  = 'native_resolver';
// {'<host>': {'method':..., 'servidor':..., 'ts':...}}
```

**Modo agresivo si el host ya es conocido** (`:1010-1012`):

```dart
final aggressive  = _hostKnown && _cachedMethod == _kMethodWebview;
final intervalMs  = aggressive ? 1800 : 3500;    // poll más frecuente
```

**Timeout adaptativo** (`:1153-1174`):

```dart
final timeoutSec = _hostKnown ? 7 : 12;
_searchTimer = Timer(Duration(seconds: timeoutSec), () {
  ... isSearching = false; showWebView = true;
});
```

**UI según resultado** (`:1241-1324`):
- No encontró → spinner (texto "Host conocido · resolver nativo")
- Encontró → check verde "¡Stream encontrado!" + botón "Reproducir"
- Falló → WebView a pantalla completa con botón atrás circular (blur)

Navegación al player (`:967-990`):

```dart
Navigator.pushReplacement(context, MaterialPageRoute(
  builder: (_) => PlayerScreen(videoUrl: selectedM3u8Url!, idcontenido: ...)));
```

---

## 9. Duplicación de código en `features/player/data/`

| Fichero | Líneas | Contenido |
|---|---|---|
| `native_resolvers.dart` | 737 | `StreamResult` + `NativeResolvers` (VOE, Dood, StreamWish, VidHide, GoodStream, Lulu, Pixeldrain, Buzzheavier, DropCDN) — **sin OK.ru**, sin WebView |
| `extractor.dart` | 1325 | `NativeResolvers` (idéntico) + `ExtractorPage` (WebView **visible**) |
| `extractor_mobil.dart` | 1325 | **Byte-idéntico a `extractor.dart`** (verificado con `diff`) |

> `hls/hls_extractor.dart` es la **evolución** de `features/player/data/`:
> añade `NativeResolvers` al probe y OK.ru.

---

## 10. Resumen de "qué pasar" cuando algo falla

| Situación | Qué hace la app |
|---|---|
| Provider devuelve 0 embeds | Se salta a la siguiente fuente (`Future.wait` en `fetchProgressive`) |
| CineHax sin metadata | Busca el siguiente capítulo hd |
| Embed con `data-server` en base64 | `/player/<b64>` → `window.location.href` |
| Resolver nativo devuelve `null` | Cae al WebView 1×1 con `MediaDetector.js` |
| WebView 1×1 no encuentra nada en 10 s | Timeout → `_HiddenProbe` se cierra, se marca no verificado |
| Todos los servidores fallan | `_allServersFailed = true` → pantalla de error con "Reintentar búsqueda" (limpia caché) |
| Torrent/MKV detectado | Se **filtra** salvo `torrent_mkv_enabled`; **nunca se reproduce** (`libtorrent_flutter` sin uso) |