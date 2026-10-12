# 18 · Glosario y acrónimos

## A — Servicios y plataformas

| Término | Significado |
|---|---|
| **TMDB** | *The Movie Database*. Base de datos de películas/series. API en `api.themoviedb.org/3`, imágenes en `image.tmdb.org/t/p/{size}` |
| **OMDb** | *Open Movie Database*. API para ratings IMDB, Metascore, runtime, director, actores |
| **Stremio** | Ecosistema de addons de vídeo. Protocolo HTTP con `manifest.json` + `/{catalog\|meta\|stream}/{type}/{id}.json` |
| **Nuvio** | Otro cliente de addons, con su propio formato de paquetes de "scrapers" (`scrapers: []` en el manifest) |
| **Supabase** | Backend-as-a-Service usado para cuentas, perfiles y sincronización. PostgreSQL + Auth |
| **Kilo AI** | Proveedor de LLM **gratuito y sin API key** (`api.kilo.ai/api/gateway`). Base de todas las funciones de IA |
| **GitHub API** | Se usa para descubrir addons de la comunidad por *topics* |
| **modlyo.com** | API propia del autor: versiones de la app, fuentes PHP del usuario, subtítulos VTT, servidores de FuegoCine |
| **IntroDB / SkipDB** | APIs con marcas de tiempo de intro/outro/recap |
| **OpenSubtitles (Stremio)** | `opensubtitles-v3.strem.io`, addon de subtítulos |

## B — Formatos de vídeo

| Término | Significado |
|---|---|
| **HLS** | *HTTP Live Streaming*. Formato de streaming adaptable basado en playlists `.m3u8`. Consta de una **master playlist** (lista de variantes) y **media playlists** (lista de segmentos) |
| **master.m3u8** | Playlist que referencia las variantes de calidad (`#EXT-X-STREAM-INF:RESOLUTION=1920x1080,BANDWIDTH=...`) |
| **DASH** | *Dynamic Adaptive Streaming over HTTP*. Alternativa a HLS con `.mpd`. La app lo menciona en los User-Agents de IPTV |
| **Segmento (.ts / .m4s)** | Fragmento de vídeo de unos pocos segundos. HLS descarga muchos; DASH usa fragmentos fMP4 |
| **ABR / bitrate switching** | Cambio automático de calidad según el ancho de banda |
| **CDN** | *Content Delivery Network*. Los `.m3u8` suelen servirse desde CDN con protección anti-hotlink |
| **Referer / Origin** | Headers HTTP obligatorios en muchos CDN. Si faltan → **403** |
| **Embed** | URL de una página HTML intermedia (`hglink.to/e/abc`) que contiene el vídeo |
| **Cyberlocker** | Sitio de hosting tipo "megaupload" (`voe`, `filemoon`, `streamwish`…) |
| **magnet / infoHash** | Enlace torrent. La app los detecta y **filtra**, pero no los reproduce |
| **MPD / M3U8 / MP4** | Contenedores/listas: DASH manifest, HLS playlist, MP4 progresivo |

## C — Técnicas de scraping

| Término | Significado |
|---|---|
| **Scraping** | Descarga y parseo de HTML para extraer datos |
| **Selector CSS** | Regla para localizar elementos (`div.card`). La app **no los usa**: todo es `RegExp` |
| **RegExp** | *Regular Expression*. Patrón de texto. Es la herramienta principal de parsing del proyecto |
| **`__NEXT_DATA__`** | Script JSON embebido por Next.js con los datos de la página. Se parsea con `jsonDecode` en Cuevana |
| **JSON-LD** | `schema.org` embebido. Se usa en SeriesKao |
| **Dotall (`(?s)`)** | Flag de regex para que `.` también matchee saltos de línea |
| **Ofuscación** | Código difícil de leer. Ej: `P.A.C.K.E.R` (empaquetador JS), ROT13, Caesar, base64 en VOE |
| **P.A.C.K.E.R** | Empaquetador JS genérico. El desempacador propio (`_unpackEval`, radix variable) traduce los índices a literales |
| **PoW (Proof-of-Work)** | Computación costosa que demuestra trabajo. En Embed69: buscar un nonce tal que `sha256(challenge+nonce)` empiece por N ceros |
| **AES-256-CBC** | Cifrado en bloque. La clave son los **bytes binarios** del SHA-256 (no hex); el IV son los primeros 16 bytes del ciphertext |
| **ROT13** | Cada letra se sustituye por la que está 13 posiciones después. Se usa como primera capa en VOE |
| **Base64** | Codificación binaria→texto. Se usa en 4 sitios distintos: VOE, Cinecalidad (`zopass`), TioPlus (`data-server`), Embed69 |
| **`data-*` attributes** | Atributos HTML personalized. Típico: `data-server`, `data-url`, `data-name` |
| **User-Agent** | Cadena que identifica el cliente. La app usa 5+ UAs distintos según el contexto |

## D — Flutter / Android

| Término | Significado |
|---|---|
| **D-pad** | Botones direccionales del mando de TV. En Flutter se gestionan con `FocusNode` + `onKeyEvent` + `LogicalKeyboardKey` |
| **`FocusNode` / `Focus` / `FocusScope`** | Sistema de foco de Flutter. `requestFocus()`, `canRequestFocus`, `autofocus` |
| **`FocusTraversalGroup`** | Grupo que define el orden de tabulación (`OrderedTraversalPolicy`) |
| **`ensureVisible`** | Desplaza un `Scrollable` para centrar el elemento enfocado. Imprescindible en TV |
| **`IndexedStack`** | Apila widgets pero solo renderiza el activo. Conserva el estado de las pestañas |
| **`AutomaticKeepAliveClientMixin`** | Conserva el estado de un widget dentro de un `IndexedStack` o `PageView` |
| **`ChangeNotifier`** | Patrón de observabilidad de Flutter. `addListener`/`removeListener`/`notifyListeners` |
| **`ValueNotifier<T>`** | `ChangeNotifier` con un valor. Se usa como "event bus" informal (`DownloadNavBus`, `GuardadosBus`) |
| **`Provider`** | Paquete de DI + estado (declarado, **no usado**) |
| **ExoPlayer** | Reproductor de vídeo nativo de Android. Envuelto por `video_player` |
| **MediaCodec** | Codec de硬件 de Android. El error clásico "MediaCodec Decoder init failed" aparece al crear 2 decoders |
| **`MethodChannel`** | Canal de comunicación Dart ↔ Kotlin/Platform |
| **Platform View** | Vista nativa incrustada en el árbol de widgets Flutter (`webview_flutter` lo usa) |
| **`SystemUiMode.immersiveSticky`** | Modo pantalla completa que no se cierra con un toque |
| **`SystemChrome.setPreferredOrientations`** | Bloquea la orientación (landscape en TV, portrait en móvil) |
| **`PopScope`** | Controla el botón "atrás" del sistema. `canPop: false` + diálogo de confirmación |
| **`LEADBACK_LAUNCHER`** | Intent-filter que hace que la app aparezca en el launcher de Android TV |
| **`android.software.leanback`** | `uses-feature` que declara soporte de TV |
| **Foreground Service** | Servicio que sigue corriendo con la app en segundo plano (descargas, Chromecast) |
| **APK / ABI** | Paquete Android / arquitectura de CPU (arm64-v8a, armeabi-v7a, x86_64) |
| **App Bundle (.aab)** | Formato de publicación de Play Store |

## E — Streaming y red

| Término | Significado |
|---|---|
| **Espejo / mirror** | Dominio alternativo del mismo sitio. Las listas de espejos (`_doodMirrors` con 30 entradas) permiten sobrevivir a bloqueos |
| **Caché TTL** | *Time To Live*. Tiempo que un dato cacheado se considera válido |
| **Caché "first-win"** | Estrategia: se acepta el primer servidor que funciona y se cancela el resto |
| **Semáforo / gate** | Control de concurrencia. `_VerifyGate` limita los WebView simultáneos |
| **Primer salto / segundo salto** | En un embeds: URL → player.php → m3u8 |
| **Hotlink protection** | Protección del CDN que exige `Referer`/`Origin` correctos |
| **Rate limit** | Límite de peticiones por IP. Mitigado rotando 3 API keys de TMDB y 3 de OMDb |
| **SSE** | *Server-Sent Events*. Stream de texto con formato `data: {...}\n\n`. La app decodifica con buffer de bytes para no romper UTF-8 |
| **PKCE** | *Proof Key for Code Exchange*. Flujo de OAuth que `supabase_flutter` usa por defecto |
| **QuickJS** | Motor JavaScript embebido. `package:flutter_js` lo usa para ejecutar los addons |
| **Polyfill** | Implementación de una API de JS estándar que el runtime no trae (`fetch`, `Buffer`, `URL`, `TextEncoder`…) |

## F — Dominio del proyecto

| Término | Significado |
|---|---|
| **TMDB id** | Identificador numérico de una película/serie en TMDB. El identificador interno de la app |
| **ID convergente** | `imdbId` (`tt1234567`) — el estándar de la industria. La app lo obtiene de `external_ids` |
| **`S1E1`** | Temporada 1, Episodio 1. Formato de la clave de subtítulos |
| **Add-on / Addon** | Extensión que aporta fuentes (streams), catálogos (contenido) o funciones (subtítulos, badges) |
| **Fuente** | Proveedor de servidores de streaming. Hay dos tipos: *nativas* (hardcodeadas) y *addons* (instalables) |
| **Servidor** | Un enlace de reproducción concreto, con idioma y calidad |
| **Extractor** | Componente que convierte una URL de embed en una URL de reproducción |
| **Provider (extractor)** | Implementación concreta para un sitio: `voe`, `doodstream`, `cinesrc`… |
| **Corriente / badges** | Etiquetas/logos que se añaden a los servidores según el pack de reglas instalado |
| **Escrubir / Derivar** | Buscar una película/serie dentro de un catálogo, o al revés (URL → título) |
| **`es_MX` / `es_ES` / `en_US` / `ja_JA`** | Códigos de idioma canónicos: Latino, Castellano, Subtitulado, Japonés |
| **Latino / Castellano** | Audio en español latinoamericano vs. español de España |
| **Voseo** | Variante del español rioplatense ("vos"). En este proyecto, un servidor "VOSE" se marca como subtitulado |
| **`espeak`/`zopass`/`rcp`** | Tokens/paths ofuscados propios de cada sitio |
| **`_v1` en una clave** | Versión del esquema de datos. Permite migrar sin romper |
| **Perfil** | Subcuenta de un usuario (máx 5 por cuenta), con avatar, PIN y ajustes propios |
| **Modo invitado** | Uso sin cuenta, con todo en local |
| **Pairing TV** | Vincular la TV con el móvil mediante un código de 6 dígitos |

## G — Siglas de claves

| Clave | Significado |
|---|---|
| `tmdb_id` | ID de TMDB |
| `imdb_id` | ID de IMDb |
| `progress_seconds` | Segundo exacto de reproducción |
| `profile_id` | UUID del perfil en Supabase |
| `account_id` | UUID de la cuenta en `auth.users` |
| `season` / `episode` | Temporada / Episodio (**0 = centinela** para películas) |
| `tipo` | `'movie'` \| `'tv'` (normalizado; `'serie'` se convierte a `'tv'`) |
| `es_<fuenteId>` | Flag booleano: este servidor viene de esa fuente |
| `es_addon` | Flag: viene de un addon |
| `resolved_m3u8` | URL de reproducción ya resuelta |
| `verificado` | Se comprobó que la URL devuelve vídeo |
| `webview_only` | No tiene resolver nativo; hay que abrirlo en WebView |
| `prioridad` | 999 = CineSRC (último recurso) |
| `is_hls` | La URL es `.m3u8` |
| `isTorrent` | La URL es magnet/torrent |

---

## Índice de los 14 providers

| # | Provider | Archivo | Resuelve a |
|---|---|---|---|
| 1 | **VOE** | `hls_extractor.dart:250-312` | ✅ Nativo (ROT13+Caesar+base64) |
| 2 | **DoodStream** | `hls_extractor.dart:315-357` | ✅ Nativo (`/pass_md5/`) |
| 3 | **StreamWish** | `hls_extractor.dart:360-446` | ✅ Nativo (8 espejos + `/dl`) |
| 4 | **VidHide** | `hls_extractor.dart:449-514` | ✅ Nativo (P.A.C.K.E.R) |
| 5 | **GoodStream** | `hls_extractor.dart` | ✅ Nativo |
| 6 | **LuluStream** | `hls_extractor.dart` | ✅ Nativo |
| 7 | **DropCDN** | `hls_extractor.dart:645-692` | ✅ Nativo |
| 8 | **Pixeldrain** | `hls_extractor.dart` | ✅ Nativo |
| 9 | **Buzzheavier** | `hls_extractor.dart:603-642` | ✅ Nativo (header `hx-redirect`) |
| 10 | **OK.ru** | `hls_extractor.dart:706-783` | ✅ Nativo (`flashvars.metadata`) |
| 11 | **FileMoon** | — | ❌ Solo WebView |
| 12 | **Vimeos** | — | ❌ Solo WebView |
| 13 | **VidSrc** | `vidsrc_extractor.dart` | ❌ Solo embed → WebView |

## Índice de los 14 fuentes (nativas)

| # | id | Label | Estado en el agregador |
|---|---|---|---|
| 1 | `embed69` | Embed69 | ✅ Activa |
| 2 | `poseidon` | Poseidon | ✅ Activa |
| 3 | `cuevana` | Cuevana | ✅ Activa |
| 4 | `unlimplay` | Unlimplay | ✅ Activa |
| 5 | `cinecalidad` | Cinecalidad | ✅ Activa |
| 6 | `tioplus` | TioPlus | ✅ Activa |
| 7 | `fuegocine` | FuegoCine | ✅ Activa |
| 8 | `pelispedia` | Pelispedia | ✅ Activa |
| 9 | `seriesmetro` | SeriesMetro | ✅ Activa |
| 10 | `smartpelis` | SmartPelis | ✅ Activa |
| 11 | `customapi` | Mis APIs | ✅ Activa (tope 64) |
| 12 | `cinesrc` | CineSRC | ✅ Activa (último recurso) |
| 13 | `hackstore` | HackStore | ❌ **NO en `fuentesActivas`** |
| 14 | `pelisplus` | PelisPlusHD | ❌ **NO en `fuentesActivas`** |

## Índice de los 5 sitios con scraping de catálogo

| Sitio | Base URL | Listing | Búsqueda |
|---|---|---|---|
| SeriesKao | `https://serieskao.top` | ✅ | ✅ |
| TioPlus | `https://tioplus.app` | ✅ | ✅ |
| Cuevana | `https://wv3.cuevana3.eu` | ✅ | ✅ |
| PelisPlus | `https://www.pelisplushd.la` | ✅ | ✅ |
| CineHax | `https://cinehax.com` | ❌ | ✅ |
| *(AnimeJK)* | `https://jkanime.net` | — | ⚠️ No registrado |

## Índice de las 4 TTL principales

| Caché | TTL |
|---|---|
| `TvM3u8Cache` (m3u8 por embed) | 1 hora |
| `Caché A` (`srv_opt_*`, servidor óptimo) | 1 hora |
| `FuentesCache` (`serv_cache_*`, lista global) | 12 horas (configurable) |
| `ServidoresPreloader` | 2 días |
| `ForYouGate` ("Para ti") | 20 horas (15 min tras fallo) |
| `DailyAiGate` (secciones + Top 10) | 24 horas |
| `catalogTtl` (modelos de IA) | 6 horas |
| `AiUsageTracker` (eventos) | 7 días |