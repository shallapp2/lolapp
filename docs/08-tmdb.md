# 08 · TMDB

Cliente de la API de [The Movie Database](https://themoviedb.org). Aporta
**todos los metadatos** (títulos, sinopsis, imágenes, temporadas) y los
**catálogos** (home, discover, búsqueda, recomendaciones). Es la única fuente de
contenido "real" de la app: los sitios web solo aportan *embeds*.

---

## 1. `core/constants/tmdb_apis.dart` (92 líneas)

### 1.1 Las 3 claves hardcodeadas

```dart
// tmdb_apis.dart:13-19
static const List<String> builtInKeys = [
  'a2d9bbed370d9f678e34006f8750a5a5',
  '439c478a771f35c05022f9feabcca01c',
  '1865f43a0549ca50d341dd9ab8b29f49',
];
```

### 1.2 Rotación aleatoria en cada petición (`:37-48`)

```dart
final keys = builtInKeys.where((k) => k.trim().isNotEmpty).toList();
if (keys.isEmpty) return '';
if (keys.length == 1) return keys.first;
return keys[_rng.nextInt(keys.length)];   // _rng = Random() estático (:35)
```

> Mitiga el rate-limit de TMDB distribuyendo las peticiones entre 3 claves.

### 1.3 Preferencia del usuario

| Pref | Tipo | Default |
|---|---|---|
| `tmdb_use_custom_api` | bool | `false` |
| `tmdb_custom_api_key` | String | `''` |
| `tmdb_api_language` | String | `'es-MX'` (whitelist: `es-MX`, `es-ES`, `en-US`, ver `:70-81`) |

Si `useCustom` y hay key → usa **solo** esa.

### 1.4 Géneros excluidos (`:26-33, 88-91`)

```dart
static const List<int> excludedTvGenreIds    = [10764, 10767, 10763, 10766];
// 10764 Reality · 10767 Talk · 10763 News · 10766 Soap
static const List<int> excludedMovieGenreIds = <int>[];
static String get tvWithoutGenres => excludedTvGenreIds.join(',');  // "10764,10767,10763,10766"
```

---

## 2. Inventario de servicios

| Fichero | Líneas | Clase | Base / uso |
|---|---|---|---|
| `tmdb_home_api.dart` | 841 | `TmdbHomeService` | Home **TV** |
| `tmdb_home_mobile_api.dart` | 837 | `TmdbHomeMobileService` | Home **móvil** (+ `main_slider` hero) |
| `tmdb_content.dart` | 854 | `TmdbContentService` | Ficha completa + temporadas + capítulos + OMDb |
| `tmdb_player_api.dart` | 805 | `TmdbPlayerService` | Datos del reproductor |
| `tmdb_search_api.dart` | 214 | `TmdbSearchService` | Búsqueda simple `/search/multi` |
| `tmdb_search.dart` | 419 | `TmdbSearchService` | ⚠️ **DUPLICADO, no importado por nadie** |
| `tmdb_discover_api.dart` | 265 | `TmdbDiscoverService` | Descubrir por género + orden |
| `tmdb_recommendations_api.dart` | 223 | `TmdbRecommendationsService` | Similares/recomendaciones con fallback de idioma |
| `sedpnFtyH` | 0 B | — | ⚠️ **Archivo vacío, blob git `e69de29…`, 0 referencias** |

Todos usan:

```dart
static const _base = 'https://api.themoviedb.org/3';
static const _imgBase = 'https://image.tmdb.org/t/p';
// con api_key de TmdbApis.getApiKey() y language de TmdbApis.getLanguage()
```

---

## 3. Home

### 3.1 `TmdbHomeService.fetchHome()` (`tmdb_home_api.dart:520-753`)

Lanza **11 futures en paralelo** (`Future.wait` `:617`):

| Clave interna | Endpoint | Sección del home |
|---|---|---|
| `top_movies` | `/movie/top_rated` | `home_featured_movies` |
| `top_tv` | `/tv/top_rated` | `home_featured_series` |
| `popular_movies` | `/movie/popular` | `home_popular_movies` |
| `popular_tv` | `/tv/popular` | `home_popular_series` |
| `trending_movies` | `/trending/movie/week` | `home_trending_movies` |
| `trending_tv` | `/trending/tv/week` | `home_trending_series` |
| `now_playing` | `/movie/now_playing` | `home_latest` |
| `on_the_air` | `/tv/on_the_air` | `home_latest` |
| `recent_discover_movies` | `/discover/movie?sort_by=primary_release_date.desc` | `home_latest` |
| `year_movies` | `/discover/movie?primary_release_year={año}` | `home_year_movies` |
| `year_tv` | `/discover/tv?first_air_date_year={año}` | `home_year_series` |

Títulos resultantes (`section(...)` `:623-646`, `Called` `:650-733`):
`Destacadas - Películas/Series`, `Populares - …`, `Tendencia - …`,
`Últimos estrenos - …`, `Por año - …`, más:

- `continue_watching` — array **vacío** que rellena la página desde prefs (`:735-740`)
- `movie_genre_sliders` — `_buildGenreSliders` (`:755-797`), 7 géneros

### 3.2 `TmdbHomeMobileService.fetchHome()` (`tmdb_home_mobile_api.dart:523-751`)

Mismos endpoints + **`main_slider`** propio (8 ítems, `_mainSliderSize = 8` `:22`).

`_buildMainSlider()` (`:430-519`):

1. **Prioridad de fuentes**: `trending_movies` → `trending_tv` → `popular_movies`
   → `popular_tv` → `now_playing` → `top_movies` → `top_tv` (`:458-464`)
2. Solo ítems **con `backdrop_path`** que pasen los filtros
3. Sort por `popularity` desc (`:469-477`)
4. **Balanceo ~50/50 movie/tv** intercalando por paridad (`:479-495`)
5. Enriquece con logos y reordena **prefiriendo los que tienen logo** (`:503-518`)

### 3.3 Filtros aplicados a todo el home

`_applyFilters` (`tmdb_home_api.dart:172-226`; gemelo en mobile `:128-175`):

| # | Filtro | Pref |
|---|---|---|
| 1 | `poster_path` no vacío | — |
| 2 | **Conocido**: `voteCount >= _minVoteCount` (80 movie / 40 tv) **O** `popularity >= _minPopularity` (8.0) | — |
| 3 | `adult` solo si se permite | `allow_adult_content` |
| 4 | No estrenado → solo si se permite | `show_unreleased` |
| 5 | **Regionalización**: excluye 27 idiomas (indio, asiático, ruso) — **NO** excluye es/en/fr/de/it/pt | `regional_peru` |
| 6 | **Títulos no latinos**: `_isMostlyLatin` con umbral `nonLatin/letters < 0.35` | `disable_non_latin_titles` |
| 7 | `without_genres` en discover de TV | (hardcodeado) |

Prefs leídas por `_loadPrefs` (`:80-102`):

```
tmdb_enrichment, allow_adult_content, show_unreleased, home_sessions,
home_featured_movies/series, home_popular_movies/series,
home_year_movies/series, home_trending_movies/series, home_latest,
regional_peru, spanish_latino, spanish_castellano, english,
disable_non_latin_titles
```

La lista de idiomas excluidos (`:41-71`):

```dart
static const _excludedRegionalLangs = {
  'hi','bn','ta','te','kn','ml','mr','pa','gu','ur',            // indio
  'zh','ja','ko','th','vi','id','ms','tl','my','km','lo',       // asiático
  'ru','uk','bg','sr','hr','be','mk','ka','hy','kk','uz',        // ruso/ex-URSS
  'fa','ar',                                                    // medio Oriente
};
```

---

## 4. Discover — `TmdbDiscoverService` (265 líneas)

```dart
// tmdb_discover_api.dart:158-249
Future<Map<String,dynamic>> discover({
  required String mediaType,   // movie | tv
  String? genero,
  String? sortBy,             // year | title | rating | (default popularity)
  int page = 1,
})
```

- `movieGenreIds` / `tvGenreIds` (`:18-61`): mapas nombre-UI → id TMDB.
  **`'Recientes'` tiene `null`** = sin filtro de género
- `_sortBy` (`:82-95`):
  - `year` → `primary_release_date.desc` | `first_air_date.desc`
  - `title` → `original_title.asc` | `original_name.asc`
  - `rating` → `vote_average.desc`
  - default → `popularity.desc`
- `vote_count.gte`: `50` si `sortBy == 'rating'`, si no `10`
- TV: añade `without_genres = TmdbApis.tvWithoutGenres`
- `Recientes` + `popularity` → cambia a `*_date.desc` y añade `*_date.lte = hoy`
- Respuesta: `{success, data, page, has_next, total_pages}`
  con `has_next = page < total_pages && page < 500`

---

## 5. Búsqueda

**Hay dos implementaciones distintas con el mismo nombre de clase**:

### 5.1 `tmdb_search_api.dart:155-196` ✅ (la usada)

Usada por `search_page.dart:43` y `tv_search_page.dart:28`.

- `/search/multi`, 2 páginas, `limit=40`
- Filtros: `allow_adult`, `show_unreleased`, `regional_filter`

### 5.2 `tmdb_search.dart:276-399` ⚠️ (nadie la importa)

Búsqueda **multi-idioma en paralelo**:

```dart
final langs = [preferredLang, 'es-MX', 'es-ES', 'en-US'];   // :289-294
await Future.wait(...);                                      // :336-348
// página 2 solo del idioma preferido si faltan resultados    // :357-370
```

**Umbrales relajados para queries cortas** (`:297-300`):

```dart
if (q.length <= 3) {
  minVotes = 12;  minPopularity = 1.2;   // en vez de 40/20 y 3.0
}
```

---

## 6. Recomendaciones — `TmdbRecommendationsService` (223 líneas)

### 6.1 `fetchRecommendations({tmdbId, mediaType, collectionId})` (`:18-67`)

1. **Partes de la colección**: `/collection/{id}` (solo movie), etiquetadas
   `source: 'collection'`
2. `/recommendations` **y** `/similar`, cada uno ordenado por `popularity` desc
3. Dedupe que ya excluye el id actual

### 6.2 `fetchItemDetails({tmdbId, mediaType})` (`:71-153`)

Itera `_langPriority = ['es-MX','en-US','es-ES']` contra:

```
/{movie|tv}/{id}?append_to_response=external_ids,images
              &include_image_language={iso},null
```

Rellenando huecos de overview/título/géneros entre idiomas.

### 6.3 `_pickLogo` (`:155-183`)

```
es → en → sin-idioma (iso_639_1 null/vacío) → el primero
```

---

## 7. Ficha de contenido — `TmdbContentService` (854 líneas)

`fetchContent({tmdbId, mediaType})` (`:232-632`):

- **4 requests en `Future.wait` paralelo** (`:297-302`): OMDb, logo, temporadas,
  colección
- Endpoint principal (`:239-248`):

  ```
  /{movie|tv}/{id}?append_to_response=credits,videos,similar,external_ids,
                                images[,content_ratings],keywords
  ```

### 7.1 OMDb (`:11, 96-110`)

```dart
static const _omdbBase = 'https://www.omdbapi.com/';
// 3 claves rotadas
static const _omdbKeys = ['5b0e8a3e', '1289a38c', 'eb1327da'];
// requiere Response == 'True'
```

Extrae: `imdbRating`, `Metascore`, `Ratings`, `Rated`, `Runtime`, `Genre`,
`Director`, `Actors`, `Plot`, `Awards`.

### 7.2 `_fetchAllSeasons` (`:634-765`)

- Filtra `season 0` (specials) según `show_season_specials`
- Filtra capítulos con `air_date` futura (`showUnreleasedEps: false` **siempre**)
- `_titleWithEnglishFallback` (`:162-176`): nombres de temporada/episodio con
  fallback a `en-US` si vienen como "Episodio N" / "Episode N" (`:674-689, 734-742`)

### 7.3 Similares

Filtrado a `vote_average > 7.0` y `.take(14)` (`:401-428`).

### 7.4 ~50 campos de salida

Incluye `imdb_data`, `videos:{es,en}` (solo YouTube trailer/teaser/clip,
`:367-388`), `production_companies`, `production_countries`, `spoken_languages`,
`networks`, `created_by`, `origin_country`, `keywords`, `budget`, `revenue`,
`collection`.

### 7.5 `fetchEpisode({tmdbId, seasonNumber, episodeNumber})` (`:768-853`)

`/tv/{id}/season/{s}/episode/{e}` con fallbacks a `en-US` para nombre y overview,
más `crew` y `guest_stars`.

---

## 8. Datos del reproductor — `TmdbPlayerService` (805 líneas)

`fetchPlayer({tmdbId, mediaType, temporada, capitulo})` (`:151-176`).

### 8.1 Película (`:178-274`)

```
/{movie}/{id}?append_to_response=external_ids,recommendations,belongs_to_collection
```

- Recomendaciones = partes de colección si existe, si no `/recommendations`
- Si quedan <8, se rellena con `/movie/popular` ya estrenados

Devuelve:

```
{idcontenido, tmdb_id, tipo, imdb_id, titulo_contenido, backdrop, poster, logo,
 overview, fecha_salida, calificacion, subtitulo, siguiente,
 recomendaciones, temporadas}
```

### 8.2 Serie (`:276-475`)

1. Si no vienen S/E, `_findFirstEpisode` (`:477-511`) busca el primer capítulo ya
   estrenado
2. `/{type}/{id}/season/{temp}`, filtra no estrenados, localiza `capActual`
3. Resuelve `siguiente` con `_resolveNextEpisode` (`:513-532`) o, si no hay más
   en la temporada, `_resolveNextSeasonFirst` (`:534-572`)
4. En paralelo: `_fetchLogo`, `_buildAllSeasons` (`:574-644`), `_mapRecommendations`

### 8.3 Subtítulos por defecto

URL fija:

```
https://modlyo.com/subtitulo/contenido/{tmdbId}/es_MX.vtt
```

(`tmdb_player_api.dart:268-269` y `:469-470`)

### 8.4 `_isUnreleased` / `_isReleased` (`:81-96`)

```dart
// Compara SOLO el día (sin hora)
static bool _isUnreleased(...) => ...
// ⚠️ SIN fecha se trata como NO estrenado → filtra trailers
static bool _isReleased(...) => ...
```

---

## 9. Imágenes y sus tamaños

Todas bajo `https://image.tmdb.org/t/p/<size><path>`:

| Uso | Tamaño | Ubicación |
|---|---|---|
| Poster home | `w500` | `tmdb_home_api.dart:390`; mobile `:333-334`; search `:176-177` |
| Backdrop home | `w1280` | `tmdb_home_api.dart:391-392`; mobile `:335-336` |
| Logo | `w500` | `tmdb_home_api.dart:342`; mobile `:283`; content `:159`; player `:148` |
| Backdrop aleatorio sin idioma | `w1280` | `tmdb_home_api.dart:512-516` |
| Poster genérico | `w500` (default) | `tmdb_content.dart:90-94` |
| Póster similar/colectiva | `w342` | `tmdb_content.dart:207,421`; discover `:141`; search_api `:91-95` |
| Backdrop en discover/search | `w780` | `tmdb_discover_api.dart:142`; `tmdb_search_api.dart:144` |
| Perfil de actor | `w185` | `tmdb_content.dart:229,489,526` |
| Póster de temporada | `w342` (player) / `w300` (content) | `player:634`; `content:274-275` |
| Still de episodio | `original` (player) / `w300` / `w500` | `player:52,622`; `tmdb_service.dart:331`; `content:841` |
| **`original`** (máxima calidad) | `original` | `tmdb_player_api.dart:52` (default de `_tmdbUrl`) |

### Selección de logo

Implementada **4 veces casi idénticas** (`tmdb_home_api.dart:282-345`,
`tmdb_home_mobile_api.dart:223-286`, `tmdb_content.dart:112-160`,
`tmdb_player_api.dart:98-149`):

```
GET /{movie|tv}/{id}/images?include_image_language={language},null,en,es-MX,es
→ pick por iso_639_1 == language.split('-').first
→ si no, el primero con iso_639_1 null o vacío
→ si no, el primero con iso == 'en'
→ si no, logos.first
```

`tmdb_player_api.dart:_tmdbUrl` (`:52-73`) además **deserializa paths que vienen
como JSON array o string entrecomillado** (`jsonDecode` si empieza por `[`/`"`) —
útil para `still_path` de algunas respuestas.

---

## 10. Caché

**No hay caché HTTP de TMDB.** Lo único cacheado:

| Caché | Dónde | TTL |
|---|---|---|
| `_logoCache: Map<String,String>` **en memoria de la instancia del servicio** (clave `'{m|t}_{tmdbId}'`) | `tmdb_home_api.dart:76`, `tmdb_home_mobile_api.dart:35` | Sesión (dura lo que la pantalla) |
| Rotación de claves | En vez de cache | — |

Se cachea también el **vacío** (`''`) para no reintentar logos inexistentes
(`:298, 311`). Como `TmdbHomeService` es una instancia de campo en cada widget
(`tv_home_page.dart:42`, `home_page.dart:38`), la caché de logos sobrevive
mientras viva la pantalla.

> Consecuencia: cada entrada a Home dispara **11 requests × 3 claves rotadas**
> y **no hay caché entre navegaciones** salvo los carousels ya renderizados.

---

## 11. `data/recommendations/` — el motor de recomendaciones

No es *collaborative filtering*, sino **LLM-augmented**: un modelo propone
títulos en JSON y **todo se verifica contra TMDB antes de mostrarse**.

### 11.1 `user_taste_profile.dart` (142 líneas)

2 claves `SharedPreferences`:

| Clave | Contenido |
|---|---|
| `user_taste_profile_v1` | `{genres: {k: w}, keywords: {k: w}}` |
| `user_likes_v1` | `{tmdbId: {title, type, genres, keywords, likedAt}}` |

```dart
// user_taste_profile.dart:72-87
void recordWatch({required List<String> genres, List<String> keywords = const [],
                  double weight = 1.0}) {
  for (final g in genres)  genreWeights[g.toLowerCase()]   = (genreWeights[g.toLowerCase()]   ?? 0) + weight;
  for (final k in keywords) keywordWeights[k.toLowerCase()] = (keywordWeights[k.toLowerCase()] ?? 0) + weight * 0.5;
}

// :90-109  likeContent() llama recordWatch(..., weight: 3.0)  ← refuerzo fuerte
```

Exposición al prompt (`toPromptSnippet`, `:132-141`): `topGenres(8)` +
`topKeywords(10)` + títulos de los 15 likes más recientes.

### 11.2 `history_signals.dart` (113 líneas)

`loadRawHistory()` (`:27-52`): escanea **todas** las prefs que empiezan por
`cachePlayer_`, parsea, descarta `segundo < 8`, y ordena por `timestamp` desc.

`of(rows)` (`:54-105`) clasifica:

```dart
watched    = f['watched'] || f['terminado']
             || (duracion > 0 && segundo / duracion >= 0.9)     → 'terminado'
abandoned  = duracion > 0 && segundo / duracion < 0.10          → 'abandonado'
otro                                                            → descartado
```

Dedupe por `idcontenido|tmdb_id|id`; tope `cap = 30` (`:102`).
`lines()` (`:108-112`) produce `- {título} ({kind}): {status}`.

### 11.3 `for_you_generator.dart` (309 líneas)

**Gate de 20 h** (`:65-74`):

```dart
class ForYouGate {
  static const windowMs              = 20 * 60 * 60 * 1000;   // 20 horas
  static const windowAfterFailureMs  = 15 * 60 * 1000;       // 15 min tras fallo
  static bool isDue(int lastAttemptMs, bool lastWasModelFailure, int nowMs) {
    if (lastAttemptMs <= 0) return true;
    return nowMs - lastAttemptMs >= (lastWasModelFailure ? windowAfterFailureMs : windowMs);
  }
}
```

Claves: cache `for_you_recs_v1`, marcas `for_you_marks_v1 = {t: ms, f: bool}`.

**Algoritmo** `generateIfDue({force})` (`:241-308`):

1. Guarda `_inProgress` (evita reentrada), lee marcas; si no toca → devuelve cache
2. `HistorySignals.loadRawHistory()` → `of()`. Si vacío y no `force` → cache
   (**no gasta IA**)
3. Escribe marca `(now, false)`
4. Carga `UserTasteProfile` y **enriquece con lo visto**: por cada
   `status == 'terminado'` con géneros → `recordWatch(genres, weight: 1.5)` (`:261-265`)
5. Construye prompt con `HistorySignals.lines()` + `profile.toPromptSnippet()`
6. `ai.complete(prompt, usageKind: 'foryou')`. Si no es `AiText` → marca
   `(now, true)` y devuelve cache
7. `_parseCandidates` (`:147-174`): recorta el primer `[` … último `]`, parsea,
   mapea `{titulo, anio, tipo, porque}`, normaliza `tipo` a `tv`/`movie`
8. Construye el conjunto de **excluidos**: títulos del historial normalizados +
   títulos de los likes (`:285-290`)
9. **`_verify`** (`:196-228`): por cada candidato,
   `GET /search/{movie|tv}?api_key&language=es-MX&query&page=1`, toma
   `results.first`; **descarta** si el título normalizado ya estaba visto, si no
   hay hit, o si ya hay 12; construye
   `posterUrl = https://image.tmdb.org/t/p/w342{poster_path}`; añade el título
   encontrado al set de vistos
10. Guarda y devuelve. Si todo falla → marca `(now, true)`

**Normalización de títulos** `_normalize` (`:230-238`): lowercase, quita acentos
(`á→a, é→e, í→i, ó→o, ú→u, ü→u, ñ→n`), quita todo lo no `[a-z0-9\s]`, colapsa espacios.

**Prompt** (`:134-145`) — noteworthy: **adaptado a Colombia**, con instrucciones
explícitas de terminología:

```
'Eres un recomendador de películas y series para una persona de Colombia. '
'Te digo lo que vio: \'terminado\' le gustó, \'abandonado\' lo dejó (NO propongas nada '
'parecido), \'repetido\' le gustó mucho. Propón 20 títulos que NO estén en la lista. '
'Responde SOLO un arreglo JSON, sin texto alrededor, con objetos '
'{"titulo","anio","tipo","porque"}. "tipo" es "movie" o "tv". '
'"porque" es UNA frase corta en español de Colombia, sin voseo, que explique la '
'relación con lo que vio. Nada de contenido para adultos.'
```

> `howMany = 20` (`:80`), pero la verificación limita a **12 ítems** (`:204`).

### 11.4 `daily_sections_generator.dart` (274 líneas)

Cache `daily_home_sections_v1`. Gate externo: **`DailyAiGate.canGenerateToday()`**
(24 h rodante) de `data/ai/daily_ai_gate.dart` (llave `daily_ai_gate_ms_v1`).

`getSections({force})` (`:106-176`):

1. Cache-first; si no toca el gate → devuelve cache
2. Si `profile.likes.isEmpty && profile.genreWeights.isEmpty && watched.isEmpty`
   → **no gasta IA** (`:117-121`)
3. Prompt (`:188-213`): pide **4-6 secciones estilo Netflix/Spotify** con ejemplos
   de título, JSON array `{id,title,items:[{titulo,tipo,porque}]}`, máximo 8 ítems
   por sección, sin adulto; incluye me gusta (20), perfil de gusto e historial
4. Parseo tolerante (`:216-254`) (`items` acepta `titulo|title`)
5. **Verificación en lotes de 5 en paralelo** contra `/search/{movie|tv}`
   (`:133-141`), tomando máximo 10 ítems por sección y 6 secciones
6. Construye `posterUrl = /t/p/w342{poster}` y `backdropUrl = /t/p/w780{backdrop}`
7. Guarda y llama `DailyAiGate.markGeneratedToday()`

### 11.5 `regional_top10.dart` (334 líneas) — Top 10 LatAm

Claves: `regional_top10_ai_v2` (cache), `regional_top10_ts_v2` (timestamp),
`regional_top10_day_v2` (legacy). TTL propio `_ttlMs = 24 h` (además de `DailyAiGate`).

**Umbrales anti-rareza** (`:78-80`):

```dart
static const minVotesMovie   = 500;
static const minVotesTv      = 200;
static const minPopularity   = 40.0;
```

`load({force})` (`:92-163`): cache válido → return; si el gate permite y hay `ai`,
`_generateWithAi()`; luego **rellena** con `_fetchPopularFiltered` hasta 10 por
categoría si quedan menos.

`_generateWithAi()` (`:165-198`): prompt de *"editor de ranking para Latinoamérica
(México, Colombia, Argentina, España)"*, exactamente 10 películas + 10 series,
*"nada de cine de autor oscuro, cortos, documentales raros"*. Parsea
`{movies:[{titulo,anio}], series:[…]}` recortando del primer `{` al último `}`.

`_fetchPopularFiltered(media)` (`:249-305`):

| media | Endpoints |
|---|---|
| movie | `/movie/popular` + `/movie/now_playing` |
| tv | `/tv/popular` + `/tv/on_the_air` |

Dedupe por id, filtra por umbrales, **ordena por `vote_average` desc**, take 10.

`_tmdbSearch` (`:307-333`): a diferencia de los otros generadores, de los 3
primeros resultados elige el de **más `vote_count`** (no simplemente el primero).

### 11.6 Consumo en la UI

| Fichero | Qué hace |
|---|---|
| `features/foryou/presentation/for_you_section.dart:32-33` | `DailySectionsGenerator(ai: AiClient())` + `RegionalTop10Service(ai: AiClient())`; escucha `HomeAlgorithmBus.version` para invalidar |
| `features/home/presentation/tv_home_page.dart:141-206` | `_loadAlgoSections` convierte `Top10Item`/`HomeSectionItem` al `Map<String,dynamic>` que usa el home TV |
| `features/settings/presentation/ai/ai_usage_section.dart:34` | Instancia `ForYouGenerator` para forzar |
| `features/foryou/presentation/taste_onboarding_page{,_tv}.dart:189-192` | `RegionalTop10Service(ai).load(force:true)` + `DailySectionsGenerator(ai).getSections(force:true)` al terminar el onboarding |

---

## 12. Consumo de cuota

- **TMDB**: 3 claves rotadas aleatoriamente, sin caché → ~40 requests por
  entrada al Home
- **OMDb**: 3 claves rotadas (`5b0e8a3e`, `1289a38c`, `eb1327da`), 1 request
  por ficha de contenido

---

## 13. Problemas

1. **`sedpnFtyH`**: archivo de 0 bytes con blob git canónico vacío (`e69de29…`),
   añadido en el commit `870047d "Migración limpia de código"`. 0 referencias.
   **Borrable.**
2. **`tmdb_search.dart`**: 419 líneas duplicadas y **nunca importadas**.
3. **3 claves API hardcodeadas** (y otra en `lolbot_page.dart:121`, otra en
   `for_you_generator.dart:81`). Riesgo de rate-limit/abuso.
4. **Sin caché HTTP**: se re-piden los mismos datos al volver a la pantalla.
5. La selección de logo está **reimplementada 4 veces**.
6. `TmdbApis.getApiKey()` elige aleatoriamente, pero **no hay *retry* con la
   siguiente clave** si una da 401.
7. `_isReleased` trata "sin fecha" como no estrenado, lo que **filtra trailers**
   de PelisPlus/FuegoCine entre otros.