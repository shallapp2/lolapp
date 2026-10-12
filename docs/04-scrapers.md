# 04 · Scrapers y catálogo

Subsistema que obtiene **listados, búsquedas y fichas técnicas** haciendo
scraping de sitios web de cine y series. Es el camino *legacy/alternativo* al
sistema de addons; se usa sobre todo en la sección **Descubrir** y en
**"derivar contenido desde una URL"**.

> **Nota importante**: hay **dos pipelines paralelos**. Este documento describe
> el de **catálogo** (`scrapers/`). El de **reproducción** está en
> [05-extractores.md](05-extractores.md) y [06-fuentes-y-agregadores.md](06-fuentes-y-agregadores.md).

---

## 1. Visión general

```
TMDB id / título
   │
   ├─► Listado por tipo/género/página  ─► home/*_scraper.dart ─┐
   │      (SeriesKao, TioPlus, PelisPlus, Cuevana)              │
   │                                                            ├─► ScraperResult
   ├─► Búsqueda por texto ────────────► base/buscador.dart ────┘   {ok, section, url,
   │      (5 sitios, con heurísticas)                                currentPage, totalPages,
   │                                                                 hasNext, items[]}
   └─► Ficha técnica (TMDb + servidores + temporadas)
          └─► detail/detalle_scraper.dart ─► DetalleContenido
                                                 ├─ List<DetalleServidor>
                                                 └─ List<DetalleTemporada> → episodios
```

---

## 2. `lib/data/scrapers/base/` — La abstracción

### 2.1 `base_home_scraper.dart` (79 líneas)

DTOs y el único `fetchHtml()` compartido:

```dart
class ScraperItem {
  final String titulo;
  final String tipo;       // movie | tv | anime | dorama | episode
  final String url;
  final String poster;
  final double? rating;
  final int? year;
  final int? tmdbId;
  final String? sinopsis;
  final List<String> generos;
  // ...
  Map<String, dynamic> toMap() => {
    'titulo': titulo, 'tipo': tipo, 'url': url, 'poster': poster,
    'rating': rating, 'year': year, 'tmdb_id': tmdbId, ... };
}

class ScraperResult {
  final bool ok; final String? error;
  final String section; final String url;
  final int currentPage, totalPages; final bool hasNext;
  final List<ScraperItem> items;
}
```

`fetchHtml()` (`:62-79`):

```dart
Future<String?> fetchHtml(String url, {Duration timeout = const Duration(seconds: 25)}) async {
  final res = await http.get(Uri.parse(url), headers: {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/120 Safari/537.36',
    'Accept-Language': 'es-MX,es;q=0.9',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
  }).timeout(timeout);
  if (res.statusCode >= 200 && res.statusCode < 400) return res.body;
  return null;   // silencioso
}
```

### 2.2 `base_detail_scraper.dart` (92 líneas)

**Duplica** `ScraperItem` y `ScraperResult` (mismos campos, sin `toMap()`) con un
`fetchHtml()` **mejorado** basado en `dart:io HttpClient`, pensado para sitios con
SSL raro:

```dart
final client = HttpClient();
client.userAgent = 'Mozilla/5.0 ... Chrome/122.0.0.0 Safari/537.36';
client.connectionTimeout = timeout;
client.badCertificateCallback = (cert, host, port) => true;  // ⚠️ acepta CUALQUIER cert
request.followRedirects = true;
request.maxRedirects = 5;
```

> ⚠️ `base_detail_scraper.dart:112-113` debe ser forzado por el scraper que lo
> importe, porque declara sus propias clases homónimas. Todos los
> `*_detail_scraper.dart` importan este fichero.

### 2.3 `scraper_context.dart` (93 líneas) — la clase `Fuente`

La abstracción clave. **No hay herencia de clases**: es un *descriptor
funcional* con closures inyectados.

```dart
class Fuente {
  final String id;                  // 'serieskao', 'tioplus', ...
  final String label;               // nombre visible
  final List<String> tipos;         // movie, tv, anime, dorama...
  final List<String> generos;       // slugs; vacío = sin filtro
  final bool supportsPopulares;
  final bool hasListing;            // aparece en el selector de servicio
  final bool hasSearch;             // aparece en el filtro de búsqueda

  /// Carga de listado. Solo se llama si [hasListing] es true.
  final Future<ScraperResult> Function({
    String? tipo, String? genero, bool populares, int page,
  })? fetch;

  /// Búsqueda por texto. Solo se llama si [hasSearch] es true.
  final Future<List<BuscadorItem>> Function(String q)? search;
}
```

Helpers estáticos (`:57-92`): `tipoLabel()` y `generoLabel()` (title-case).

### 2.4 `registry.dart` (172 líneas) — **PUNTO DE EXTENSIÓN** ⭐

```dart
// registry.dart:1-7
// UNICO lugar donde se registran las fuentes.
// Para agregar una fuente nueva:
//   1. Implementa el scraper (fetch) y/o el parser de búsqueda (en buscador.dart).
//   2. Añade una entrada Fuente aquí.
//   3. pag.dart la usa automáticamente — no hace falta tocarlo.
```

Las 5 fuentes registradas:

| id | Label | Tipos | Listing | Búsqueda | `fetch` delega en | `search` delega en |
|---|---|---|---|---|---|---|
| `serieskao` | SeriesKao | movie, tv, anime, dorama | ✅ | ✅ | `SeriesKaoScraper.fetch` | `BuscadorScraper.searchSeriesKao` |
| `tioplus` | TioPlus | movie, tv, anime, dorama | ✅ | ✅ | `TioPlusScraper.fetch` | `searchTioPlus` |
| `cuevana` | Cuevana | movie, tv, tendencias, episodios | ✅ | ✅ | `CuevanaScraper.fetch` | `searchCuevana` |
| `pelisplus` | PelisPlus | movie, tv, anime, dorama | ✅ | ✅ | `PelisPlusScraper.fetch` | `searchPelisPlus` |
| `cinehax` | CineHax | movie, tv | ❌ | ✅ | — (solo búsqueda) | `searchCineHax` |

Ejemplo de entrada completa:

```dart
Fuente(
  id: 'serieskao',
  label: 'SeriesKao',
  tipos: SeriesKaoScraper.tiposDisponibles(),
  generos: SeriesKaoScraper.generos,
  supportsPopulares: true,
  hasListing: true,
  hasSearch: true,
  fetch: ({String? tipo, String? genero, bool populares = false, int page = 1}) {
    return SeriesKaoScraper.fetch(
      tipo: genero == null || genero.isEmpty ? tipo : null,   // género tiene prioridad
      genero: genero == null || genero.isEmpty ? null : genero,
      populares: populares, page: page);
  },
  search: BuscadorScraper.searchSeriesKao,
),
```

Helpers y API pública:

```dart
List<Fuente> get fuentesConListado  => fuentesRegistry.where((f) => f.hasListing).toList();
List<Fuente> get fuentesConBusqueda => fuentesRegistry.where((f) => f.hasSearch).toList();
Fuente? fuenteById(String id) => fuentesRegistry.firstWhere((f) => f.id == id, orElse: ...);
```

```dart
// registry.dart:129-172 — búsqueda SECUENCIAL (no paralela)
Future<BuscadorResult> buscarEnFuentes({required String q, String tipo = 'todas'}) async {
  final aBuscar = tipo == 'todas' ? fuentesConBusqueda
                                      : fuentesConBusqueda.where((f) => f.id == tipo).toList();
  for (final fuente in aBuscar) {
    List<BuscadorItem> items = [];
    try { if (fuente.search != null) items = await fuente.search!(query); }
    catch (_) { items = []; }        // un fallo de una fuente no cancela las demás
    resultados[fuente.id] = items;
  }
  return BuscadorResult(ok: true, query: q, tipo: tipo, total: total, resultados: resultados);
}
```

⚠️ **Debilidad**: cada búsqueda va **en serie** → 5 fuentes × ~1 s = latencia
acumulada. No hay `Future.wait`.

**Consumidores**: `features/discover/domain/mobile/pag.dart:56,95,214,315,335`,
`features/discover/presentation/source_search.dart:83,89,221,260`,
`features/discover/presentation/tv_discover_page.dart:65,288`.

### 2.5 `base/buscador.dart` (468 líneas)

Clase `BuscadorScraper` con **un método estático por sitio**. Todos usan
`fetchHtml(...)` + `RegExp`. **No usa cheerio ni selectors CSS: es 100 % regex
sobre HTML crudo.**

```dart
// buscador.dart:49-58
static Future<List<BuscadorItem>> searchSeriesKao(String q) async {
  final html = await fetchHtml('https://serieskao.top/search?s=${Uri.encodeComponent(q)}');
  final arts = RegExp(r'<article class="card">([\s\S]*?)</article>').allMatches(html);
  ...
}
```

### 2.6 `base/detalle_scraper.dart` (56 líneas)

Dispatcher de ficha técnica por id de servicio:

```dart
switch (s) {
  case 'serieskao': return await DetalleSeriesKao.fetch(url: url, titulo: titulo, tipo: tipo);
  case 'tioplus':   return await DetalleTioPlus.fetch(...);
  case 'cuevana':   return await DetalleCuevana.fetch(...);
  case 'pelisplus': return await DetallePelisPlus.fetch(...);
  case 'cinehax':   return await DetalleCineHax.fetch(...);
  default: return DetalleContenido(ok: false, error: 'Servicio no soportado: "$servicio"', ...);
}
```

### 2.7 `data/models/scraper/detalle_model.dart` (91 líneas)

```dart
class DetalleServidor { final String nombre, url; final String? idioma, calidad; }

class DetalleCapitulo {
  final int temporada, numero; final String titulo, url;
  final String? imagen, airDate; final double? rating; final int? duracion;
}

class DetalleTemporada { final int numero; final String nombre;
                        final List<DetalleCapitulo> episodios; }

class DetalleContenido {
  final bool ok; final String? error;
  final String servicio, titulo, tipo;
  final String? anio, sinopsis, poster, backdrop, logo;
  final double? rating; final int? tmdbId; final String? imdbId;
  final List<String> generos;
  final List<DetalleServidor> servidores;
  final List<DetalleTemporada> temporadas;
  final Map<String, dynamic>? extra;   // status, studios, slug, anime_id...
}
```

---

## 3. Los 5 scrapers de listado (`home/`)

Todos comparten el mismo esqueleto:
`static const base` → `tiposDisponibles()` → `fetch()` (mapea tipo→path, compone
URL, llama `fetchHtml`, parsea) → `_parseItems` (regex) → `_parsePagination` →
`_decode` (entidades HTML).

### 3.1 TioPlus — `home/tioplus_scraper.dart` (211 líneas)

```dart
static const String base = 'https://tioplus.app';
static List<String> tiposDisponibles() => ['movie','tv','anime','dorama'];

// mapeo tipo → path
case 'movie':  basePath = '/peliculas'; break;
case 'anime':  basePath = '/animes';   break;
case 'dorama': basePath = '/doramas';  break;

// composición con placeholders 'null' en la ruta
path = '$basePath/filter/$yearSeg/$genSeg/$page';   // year/gen = 'null' si no hay
```

### 3.2 SeriesKao — `home/serieskao_scraper.dart` (198 líneas)

```dart
static const String base = 'https://serieskao.top';
static const List<String> generos = [
  'accion','animacion','aventura','belica','ciencia-ficcion','comedia','crimen',
  'documental','drama','fantasia','familia','guerra','historia','romance',
  'suspense','terror','western','misterio',
];

// género → '/generos/$genero'  (valida contra la whitelist)
if (genero != null && genero.isNotEmpty) {
  if (!generos.contains(genero)) {
    return ScraperResult(ok: false, error: 'Género no válido: $genero');
  }
  path = '/generos/$genero';
}
// populares → sufijo en la ruta
path = populares ? '/peliculas/populares' : '/peliculas';
```

**Paginación tolerante a encoding roto** (`:145-182`) — 3 *fallbacks*:

```dart
// principal: "Página 3 de 659" (tolera "PÃ¡gina")
final info = RegExp(r'pagination__info[^>]*>\s*P[^0-9]{0,4}gina\s+(\d+)\s+de\s+(\d+)',
                    caseSensitive: false).firstMatch(html);
// fallback 1: <button class="pagination__btn--active">N</button>
// fallback 2: max([?&]page=(\d+)) entre los enlaces
// fallback 3: si existe aria-label="Siguiente" con href → total = current+1
```

### 3.3 Cuevana — `home/cuevana_scraper.dart` (173 líneas) ⭐ el más robusto

```dart
static const String base = 'https://wv3.cuevana3.eu';
static List<String> tiposDisponibles() => ['movie','tv','tendencias','episodios'];
```

**Parsea `__NEXT_DATA__` de Next.js** en vez de HTML:

```dart
final m = RegExp(r'<script id="__NEXT_DATA__"[^>]*>([\s\S]*?)</script>').firstMatch(html);
final next = jsonDecode(m.group(1)!) as Map<String, dynamic>;
final pageProps = next['props']?['pageProps'] as Map<String, dynamic>;
```

**Paginación EN RUTA, no en query** (`?page=N` se ignora en el sitio → bucle
infinicio):

```dart
if (page > 1) url = '$base$path/page/$page';
```

Lectura defensiva de los nombres de clave (`:70-96`):

```dart
_readInt(pp['pages'],       (v) => total = v);
_readInt(pp['totalPages'],  (v) => total = v);
_readInt(pp['total_pages'], (v) => total = v);
_readInt(pp['page'],        (v) => current = v);
_readInt(pp['currentPage'], (v) => current = v);
_readInt(pp['current_page'],(v) => current = v);
```

**Heurística anti-bucle** (`:150`):

```dart
final hasNext = items.isNotEmpty && current < total;   // lista vacía ⇒ no hay más
```

### 3.4 PelisPlus — `home/pelisplus_scraper.dart` (169 líneas)

```dart
static const String base = 'https://www.pelisplushd.la';
static const Map<String,String> subPorTipo = {
  'movie':'peliculas', 'tv':'series', 'anime':'animes',
};
// /generos/{genero}/{sub}?page=N
```

### 3.5 CineHax — `home/cinehax_scraper.dart` (21 líneas)

Solo constructor de URL, sin `fetch` (por eso `hasListing: false`):

```dart
const base = 'https://cinehax.com/ver/';
if (tipo == 'serie' || tipo == 'tv') {
  final s = (season  ?? 1).clamp(1, 999);
  final e = (episode ?? 1).clamp(1, 999);
  return '$base?tipo=serie&id=$tmdbId&season=$s&episode=$e';
}
return '$base?tipo=pelicula&id=$tmdbId';
```

---

## 4. Los 5 scrapers de detalle (`detail/`)

**Patrón universal**:
`fetchHtml(url)` → regex de metadatos → **`data-*` de servidores** →
`seasonsJson`/pills → **resolución de TMDB** → construcción de `DetalleContenido`.

> **Clave TMDB hardcodeada en los 5 scrapers**:
> `a2d9bbed370d9f678e34006f8750a5a5`

### 4.1 TioPlus — `detail/tioplus_detail_scraper.dart` (245 líneas)

```dart
static const _tmdbKey = 'a2d9bbed370d9f678e34006f8750a5a5';

// el tipo se deduce de la URL
final isMovie = !url.contains('/serie/') && !url.contains('/anime/');
final endpoint = isMovie ? 'movie' : 'tv';
```

**Título + año separados por regex** (`:36-46`):

```dart
final h1 = RegExp(r'<h1 class="slugh1">([^<]+)</h1>').firstMatch(html);
final m  = RegExp(r'^(.+?)\s*\((\d{4})\)$').firstMatch(t);
```

**Backdrop desde CSS inline** (`:62-66`):

```dart
final bg = RegExp(
  r'background-image:\s*url\("?(https://image\.tmdb\.org/t/p/w1280/[^"\)]+)"?\)',
  caseSensitive: false,
).firstMatch(html);
```

**⚠️ SERVIDORES = tokens base64 en `data-server`, NO son URLs todavía**
(`:98-109`):

```dart
final srv = RegExp(r'data-server="([^"]+)"').allMatches(html);
for (...) {
  servidores.add(DetalleServidor(nombre: 'Opción $i', url: token));  // ← token base64
}
```

La resolución real ocurre en `providers/tioplus_extractor.dart:347-388`:
`btoa(data-server)` → `/player/<b64>` → `window.location.href`.

**Temporadas desde JSON embebido en `<script>`** con re-parseo si falla
(`:113-157`):

```dart
final jsonM = RegExp(r'const seasonsJson = (\{.*?\});\s*</script>', dotAll: true)
                 .firstMatch(html);
var jsonStr = jsonM.group(1)!;
Map? seasons = jsonDecode(jsonStr);
if (seasons == null) {   // fallback: des-escapar
  jsonStr = jsonStr.replaceAll(r'\/', '/').replaceAll('\n', '').replaceAll('\r', '');
  seasons = jsonDecode(jsonStr);
}
final epUrl = '${url.replaceAll(RegExp(r'/$'), '')}/season/$t/episode/$num';
```

**Resolución TMDB por búsqueda, con reintento por título original**
(`:161-177`):

```dart
await searchTmdb(tituloFinal);
if (tmdbId == null && tituloOriginal != null) await searchTmdb(tituloOriginal!);
```

Logo con preferencia de idioma `es` (`:195-203`).

### 4.2 Cuevana — `detail/cuevana_detail_scraper.dart` (202 líneas)

`__NEXT_DATA__` → `pageProps` → `thisMovie` / `thisSerie`.

**Servidores desde el mapa `videos` indexado por idioma** (`:80-99`) — aquí **sí
son URLs de embed**:

```dart
const langs = {
  'latino': 'Español Latino', 'spanish': 'Español',
  'english': 'Subtitulado',   'japanese': 'Japonés',
};
langs.forEach((key, label) {
  for (final v in (videos[key] as List? ?? [])) {
    servidores.add(DetalleServidor(
      nombre:  v['cyberlocker']?.toString() ?? 'Servidor',
      url:     v['result']?.toString() ?? '',       // ← URL de embed
      idioma:  label,
      calidad: v['quality']?.toString(),
    ));
  }
});
```

Capítulos desde `thisSerie.seasons[].episodes[]` (`:118-139`):
`url: '$_base/episodio/$slug-temporada-$num-episodio-$epNum'`

### 4.3 SeriesKao — `detail/serieskao_detail_scraper.dart` (216 líneas)

**JSON-LD embebido** (no `<meta>`) (`:50-68`):

```dart
final y2   = RegExp(r'"datePublished":(\d{4}|"\d{4}")').firstMatch(html);
final desc = RegExp(r'"description":"([^"]+)"').firstMatch(html);
final rat  = RegExp(r'"ratingValue":"([\d.]+)"').firstMatch(html);
```

**IMDb desde `data-url`** (`:70-74`):

```dart
final imdbM = RegExp(r'data-url="/vidurl/(tt\d+)/"').firstMatch(html);
// → find external_ids en TMDB
```

Episodios por bloque `id="season-N"` + `<a class="episode-item">` (`:78-108`).
Servidores = `data-url="https?://..."`, filtrando YouTube (`:112-121`).

### 4.4 PelisPlus — `detail/pelisplus_detail_scraper.dart` (334 líneas) — el más extenso

**`_fetch()` propio con `Referer` obligatorio** (`:11-30`) — el `fetchHtml` base
no lo manda y el sitio lo requiere:

```dart
static const _ua = 'Mozilla/5.0 ... Chrome/120.0.0.0 Safari/537.36';
http.get(Uri.parse(url), headers: {
  'User-Agent': _ua,
  'Accept-Language': 'es-MX,es;q=0.9',
  'Referer': _base,
});
```

**Año con doble fallback** (`:93-103`):

```dart
final y1 = RegExp(r'<title>[^:]*:\s*[^(]+\((\d{4})\)', caseSensitive: false).firstMatch(html);
final y2 = RegExp(r'Fecha de estreno:</span>\s*(\d{4})').firstMatch(html);
```

**Servidores con `data-url` + `data-name`** (`:134-148`) — `data-name` es el
**IDIOMA**:

```dart
final srv = RegExp(r'data-url="(https?://[^"]+)"\s+data-name="([^"]+)"').allMatches(html);
servidores.add(DetalleServidor(nombre: 'Opción $i', url: u, idioma: m.group(2)));
```

**Temporadas por pills verticales** (`:150-178`):

```dart
final blocks = RegExp(r'<div[^>]+id="pills-vertical-(\d+)"[^>]*>(.*?)</div>',
                      dotAll: true).allMatches(html);
final m2 = RegExp(r'/temporada/(\d+)/capitulo/(\d+)').firstMatch(href);
```

Búsqueda TMDB con y sin año (dos intentos) (`:180-208`), y **enriquecimiento
desde TMDB** (`:267-303`): `still_path`, `air_date`, nombre oficial.

### 4.5 CineHax — `detail/cinehax_detail_scraper.dart` (151 líneas)

**No scrapea metadata del sitio**: lee el *query string* (`:13-24`).

```dart
final query  = uri?.queryParameters ?? {};
final tipoKao = query['tipo'] ?? 'pelicula';
final tmdbId  = int.tryParse(query['id'] ?? '');
if (tmdbId == null || tmdbId <= 0) {
  return DetalleContenido(ok: false, error: 'TMDB ID inválido', ...);
}
```

**Todos los metadatos vienen de TMDB** con `append_to_response=images,external_ids,credits` (`:55-57`).

Servidores de `data-url` + `<iframe src>`, filtrando YouTube (`:32-45`).
Capítulos desde TMDB, con `url = 'https://cinehax.com/ver/?tipo=$tipo&id=$id&season=$s&episode=$n'` (`:128-150`).

---

## 5. `data/scrapers/servers/` — ⚠️ VACÍO

```
cinehax_server_scraper.dart       0 bytes
cuevana_server_scraper.dart       0 bytes
pelisplus_server_scraper.dart     0 bytes
serieskao_server_scraper.dart     0 bytes
tioplus_server_scraper.dart       0 bytes
```

Los 5 ficheros tienen **0 bytes**. La lógica de servidores por sitio se resolvió
migrando a `lib/data/extractors/providers/` ([05-extractores.md](05-extractores.md)).

---

## 6. Todas las URLs base

### A) Catálogo (`lib/data/scrapers/`)

| Fuente | Base URL | Scrapers |
|---|---|---|
| SeriesKao | `https://serieskao.top` | `home/serieskao_scraper.dart:4`, `detail/serieskao_detail_scraper.dart` |
| TioPlus | `https://tioplus.app` | `home/tioplus_scraper.dart:4`, `detail/tioplus_detail_scraper.dart` |
| Cuevana | `https://wv3.cuevana3.eu` | `home/cuevana_scraper.dart:5`, `detail/cuevana_detail_scraper.dart:7` |
| PelisPlus | `https://www.pelisplushd.la` | `home/pelisplus_scraper.dart:4`, `detail/pelisplus_detail_scraper.dart:7` |
| CineHax | `https://cinehax.com` | `home/cinehax_scraper.dart:11`, `detail/cinehax_detail_scraper.dart` |
| AnimeJK (solo buscador, **no registrado**) | `https://jkanime.net` | `base/buscador.dart:367` |

### B) Reproducción (`lib/core/constants/sources.dart`, 443 líneas)

14 `SourceDefinition` con `id`, `label`, `prefsKey`, `badgeText`, `flagKey`,
`badgeColor`, `icon`, `scrape`, `forceIdioma`, `maxResults`.

| # | id | Label | prefsKey | forceIdioma | maxResults |
|---|---|---|---|---|---|
| 1 | `embed69` | Embed69 | `embed69_enabled` | — | 12 |
| 2 | `poseidon` | Poseidon | `poseidon_enabled` | — | 12 |
| 3 | `cuevana` | Cuevana | `cuevana_enabled` | — | 16 |
| 4 | `unlimplay` | Unlimplay | `unlimplay_enabled` | — | 12 |
| 5 | `cinesrc` | CineSRC | `cinesrc_enabled` | — | 12 |
| 6 | `cinecalidad` | Cinecalidad | `cinecalidad_enabled` | `es_MX` | 12 |
| 7 | `tioplus` | TioPlus | `tioplus_enabled` | `es_MX` | 12 |
| 8 | `fuegocine` | FuegoCine | `fuegocine_enabled` | `es_MX` | 12 |
| 9 | `hackstore` | HackStore | `hackstore_enabled` | — | 15 |
| 10 | `pelisplus` | PelisPlusHD | `pelisplus_enabled` | — | 15 |
| 11 | `pelispedia` | Pelispedia | `pelispedia_enabled` | — | 12 |
| 12 | `seriesmetro` | SeriesMetro | `seriesmetro_enabled` | — | 12 |
| 13 | `smartpelis` | SmartPelis | `smartpelis_enabled` | — | 12 |
| 14 | `customapi` | Mis APIs | `custom_api_enabled` | — | **64** |

```dart
// sources.dart:18-49
typedef SourceScraper = Stream<Map<String, dynamic>> Function({
  required int tmdbId, required bool isMovie, required int season, required int episode,
});

class SourceDefinition {
  final String id, label, prefsKey, badgeText, flagKey;
  final Color badgeColor; final IconData icon;
  final SourceScraper scrape; final String? forceIdioma; final int maxResults;
}
```

> ⚠️ `kRegisteredSources` se consume **solo por la UI de Ajustes**
> (`sources_section.dart:126`, `tv_sources_tab.dart`, `playback_setup_modal.dart`).
> **No participa del pipeline de reproducción**, que usa el enum `FuenteId` de
> `source_aggregator.dart`.

---

## 7. Patrones de parsing que conviene conocer

### 7.1 `__NEXT_DATA__` (Next.js)
```
<script id="__NEXT_DATA__" type="application/json">{...}</script>
→ jsonDecode → props.pageProps
```
Sitios: Cuevana (home + detail).

### 7.2 JSON-LD (schema.org)
```html
<script type="application/ld+json">
  { "@type": "Movie", "datePublished": "2023", "description": "...", "ratingValue": "8.1" }
</script>
```
Sitios: SeriesKao.

### 7.3 Constantes JS en `<script>`
```html
<script>const seasonsJson = {...};</script>
```
Sitios: TioPlus.

### 7.4 Atributos `data-*`
```html
<li data-server="BASE64_TOKEN">        <!-- TioPlus -->
<li data-url="https://..." data-name="Latino">   <!-- PelisPlus -->
<div data-url="/vidurl/tt1234567/">   <!-- SeriesKao -->
```
Típico: los servidores vienen como tokens ofuscados en atributos HTML.

### 7.5 Parser con fallback en dos etapas

```dart
// buscador.dart:377-381
final matches = arts.isEmpty
  ? RegExp(r'class="anime__item"[^>]*>([\s\S]*?)(?=class="anime__item"|class="page_|$)').allMatches(html)
  : arts;
```

### 7.6 Detección de tipo por badge
```dart
// buscador.dart:205-217
final span = RegExp(r'<span class="typeItem[^"]*">([^<]+)</span>').firstMatch(block);
if (span != null) {
  final t = span.group(1)!.toLowerCase();
  if (t.contains('anime')) tipo = 'anime';
  else if (t.contains('serie')) tipo = 'tv';
  else if (t.contains('pel'))   tipo = 'movie';
}
```

### 7.7 Normalización de entidades HTML
```dart
// buscador.dart:460-467
static String _norm(String s) => s
  .replaceAll('&amp;', '&').replaceAll('&quot;', '"')
  .replaceAll('&#039;', "'").replaceAll(RegExp(r'\s+'), ' ').trim();
```

---

## 8. Consumo desde la UI

| Pantalla | Fichero | Uso |
|---|---|---|
| Servicios / Descubrir (móvil) | `features/discover/domain/mobile/pag.dart` | Listados por servicio/tipo/género, búsqueda |
| Servicios TV | `features/discover/presentation/tv_discover_page.dart` | Igual, con D-pad |
| Buscador por fuente | `features/discover/presentation/source_search.dart` | `buscarEnFuentes()` |
| Derivar contenido | `features/discover/presentation/source_discovery_page.dart` | Búsqueda inversa por título desde una URL |
| Derivados | `features/discover/domain/{extractor,derivar}.dart` (1327 + 1418 L) | Resolución desde un sitio concreto |
| Modal de servidores por sitio | `data/aggregators/main_fuentes_servidores.dart` | Mapea servicio → fuente interna |

---

## 9. Fragilidades conocidas

1. **Todo es `RegExp` sobre HTML crudo.** Cualquier cambio de clase CSS en el
   sitio rompe el scraper. No hay tests que lo detecten.
2. **Búsqueda secuencial** en `buscarEnFuentes()` — 5 fuentes en serie.
3. **5 ficheros de 0 bytes** en `servers/` olvidados en el repo.
4. **Sin User-Agent rotativo**: cada scraper tiene su propio UA hardcodeado
   (Chrome 120 / 122 / 124 / 154).
5. **Sin caché HTTP** de HTML.
6. **Key de TMDB hardcodeada** en los 5 ficheros de detalle.
7. **`badCertificateCallback => true`** en `base_detail_scraper.dart:58`
   desactiva por completo la validación TLS.