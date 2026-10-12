# 15 · Configuración (preferencias)

Todas las preferencias están en `SharedPreferences`, leídas directamente desde
cada feature con strings hardcodeados (no hay un `StorageKeys` central en uso).

---

## 1. Estado de la sesión / arranque

| Pref | Tipo | Default | Efecto |
|---|---|---|---|
| `app_mode` | `String` | — | `'mobile'` \| `'tv'`. Determina qué shell y qué orientación se cargan. Selector en el primer arranque; cambiable desde Ajustes |
| `disclaimer_accepted` | `bool` | `false` | Muestra el aviso legal. **Rechazar cierra la app** |
| `taste_onboarding_done_v1` | `bool` | `false` | Onboarding de gustos (perfil de la IA) |
| `addons_onboarding_done_v1` | `bool` | `false` | Onboarding de addons. `AddonsOnboardingPage.shouldShow()` / `markDone()` |
| `menu_position` | `String` | `'side'` | `'side'` (rail) \| `'top'` (barra superior) en el shell TV. `MenuPositionPref.version` notifica |
| `seleccionar_servidores` | `String` | `'auto'` | `'auto'` \| `'manual'`. Su presencia marca que el setup inicial se completó (`hasPlaybackSetup()`) |

---

## 2. Fuentes de servidores

| Pref | Tipo | Default | Efecto |
|---|---|---|---|
| **`native_sources_enabled`** | `bool` | **`false`** | ⚠️ **MASTER SWITCH**. Si es `false`, **ninguna** de las 13 fuentes nativas se consulta; solo addons |
| `verificar_servidores` | `bool` | `true` | Resolver el m3u8 antes de mostrar el servidor (abre WebView) |
| `un_servidor_por_idioma` | `bool` | `true` | Solo 1 servidor por idioma y fuente |
| `mostrar_servidores_player` | `bool` | `true` | Mostrar el botón "servidores" en el player |
| `reutilizar_ultimo_enlace` | `bool` | `false` | Reutilizar el último servidor que funcionó antes de scrapear |
| `servidores_cache_ttl_hours` | `int` | `12` | TTL de `FuentesCache` |
| `servidores_concurrent_checks` | `int` | `2` | Verificaciones HLS simultáneas (1-5). Sincroniza `_VerifyGate.setMaxConcurrent` y `AddonSourceAdapter._readMaxConcurrent` |
| `idioma_audio_predeterminado` | `String` | `'latino'` | `'latino'` \| `'castellano'` \| `'subtitulado'` |
| `idioma_predeterminado_enabled` | `bool` | — | Activar el filtro de idioma |
| `idioma_predeterminado` | `String` | — | Código de idioma preferido |
| `seleccion_fuente` | `String` | `'auto'` | `FuenteSeleccion`: `manual` \| `autoPrimera` \| `autoIdioma` |
| `capitulos_precarga` | `int` | `2` | Episodios a precargar (0-10) |
| `torrent_mkv_enabled` | `bool` | `false` | Mostrar torrents/MKV en el modal. ⚠️ **No se reproducen** (`libtorrent_flutter` sin uso) |
| `native_sources_*` / `custom_api_sources` | — | — | Fuentes personalizadas del usuario (JSON list) |

### 2.1 Toggles por fuente (13)

| Pref | Fuente |
|---|---|
| `embed69_enabled` | Embed69 |
| `poseidon_enabled` | Poseidon |
| `cuevana_enabled` | Cuevana |
| `unlimplay_enabled` | Unlimplay |
| `cinecalidad_enabled` | Cinecalidad |
| `tioplus_enabled` | TioPlus |
| `fuegocine_enabled` | FuegoCine |
| `pelispedia_enabled` | Pelispedia |
| `seriesmetro_enabled` | SeriesMetro |
| `smartpelis_enabled` | SmartPelis |
| `cinesrc_enabled` | CineSRC |
| `custom_api_enabled` | Mis APIs |
| `pelisplus_enabled` | PelisPlusHD |

⚠️ **Inconsistencia**: `kRegisteredSources` en `core/constants/sources.dart`
define 14 fuentes con estos `prefsKey`, pero `FuentesConfig.fuentesActivas`
(`source_aggregator.dart:285-303`) solo consulta **12** — `hackstore_enabled` y
`pelisplus_enabled` **no aparecen** en la lista. Esos dos providers existen
(`HackStoreService`, `PelisPlusService`) pero **no se invocan desde el
agregador**.

---

## 3. Contenido / TMDB

| Pref | Tipo | Default | Efecto |
|---|---|---|---|
| `tmdb_use_custom_api` | `bool` | `false` | Usar tu propia API key |
| `tmdb_custom_api_key` | `String` | `''` | La key |
| `tmdb_api_language` | `String` | `'es-MX'` | `'es-MX'` \| `'es-ES'` \| `'en-US'` (whitelist en `tmdb_apis.dart:70-81`) |
| `tmdb_enrichment` | `bool` | — | Buscar logos y metadatos extra |
| `allow_adult_content` | `bool` | — | Permitir `adult: true` |
| `show_unreleased` | `bool` | — | Mostrar contenido nowna estreado |
| `show_unreleased_episodes` | `bool` | — | ⚠️ `tmdb_content.dart:64` lo fuerza a `false` siempre |
| `show_season_specials` | `bool` | `false` | Mostrar season 0 (specials) |
| `regional_peru` | `bool` | — | Excluir 27 idiomas regionales (indio, asiático, ruso). **No** excluye es/en/fr/de/it/pt |
| `regional_filter` | `bool` | — | Filtro regional en búsqueda |
| `disable_non_latin_titles` | `bool` | — | Excluir títulos con > 35 % de caracteres no latinos |
| `spanish_latino` | `bool` | — | Incluir audio latino |
| `spanish_castellano` | `bool` | — | Incluir audio castellano |
| `english` | `bool` | — | Incluir subtitulado en inglés |
| `home_sessions` | `bool` | — | Sección de home por "sesión" |
| `home_featured_movies` | `bool` | — | Sección "Destacadas - Películas" |
| `home_featured_series` | `bool` | — | Sección "Destacadas - Series" |
| `home_popular_movies` | `bool` | — | Sección "Populares - Películas" |
| `home_popular_series` | `bool` | — | Sección "Populares - Series" |
| `home_trending_movies` | `bool` | — | Sección "Tendencia - Películas" |
| `home_trending_series` | `bool` | — | Sección "Tendencia - Series" |
| `home_year_movies` | `bool` | — | Sección "Por año - Películas" |
| `home_year_series` | `bool` | — | Sección "Por año - Series" |
| `home_latest` | `bool` | — | Sección "Últimos estrenos" |
| `download_language` | `String` | — | Idioma preferido de las descargas |

---

## 4. Reproductor

| Pref | Tipo | Default | Rango | Efecto |
|---|---|---|---|---|
| `mini_player_enabled` | `bool` | `true` | — | Permitir minimizar el player |
| `subtitulos_inicio` | `bool` | — | — | Activar subtítulos automáticamente |
| `subtitulo_predeterminado` | `String` | `'es'` | — | Idioma de subtítulo por defecto |
| `subtitulo_offset_sec` | `int` | `0` | −30 … +30 | Desfase temporal |
| `subtitulo_font_size` | `int` | `18` móvil / `22` TV | 12 … 36 | Tamaño de fuente |
| `subtitulos_negrita` | `bool` | `true` móvil / `false` TV | — | Negrita |
| `subtitulo_vertical_offset` | `int` | `0` | −40 … +160 | Desplazamiento vertical |
| `player_vertical_offset` | `int` | `0` | ±80 | Desplazamiento vertical (TV) |
| `subtitulo_tamano` | `String` | — | `pequeno`/`mediano`/`grande` | Tamaño (TV) |
| `subtitulo_altura` | `String` | — | baja/media/alta | Altura del panel |
| `subtitulo_fondo` | `bool` | — | — | Fondo del subtítulo |
| `subtitulo_contorno` | `bool` | — | — | Contorno |
| `subtitulo_color` | `String` | hex | — | Color del texto |
| `idm_download_enabled` | `bool` | `false` | — | Descargar con IDM externo |
| `player_sub_id_<id>` | `String` | — | — | Id del subtítulo seleccionado por contenido |
| `player_sub_url_<id>` | `String` | — | — | URL del subtítulo seleccionado |
| `player_sub_lang_<id>` | `String` | — | — | Idioma |

⚠️ **No hay control de velocidad de reproducción** pese a que
`PlayerConstants.speeds = [0.5, 0.75, 1.0, 1.25, 1.5, 2.0]` existe
(core/constants/player_constants.dart, sin usar).

---

## 5. Descargas

| Pref | Tipo | Default | Efecto |
|---|---|---|---|
| `enable_downloads` | `bool` | — | Habilitar descargas |
| `auto_direct_download` | `bool` | `false` | Descargar automáticamente al elegir servidor |
| `max_concurrent_downloads` | `int` | — | Descargas simultáneas |
| `show_download_button_main` | `bool` | — | Mostrar el botón en la pantalla principal |
| `download_language` | `String` | — | Idioma preferido |
| — calidad preferida | `String` | — | En `downloads_section.dart` |

---

## 6. Notificaciones y actualizaciones

| Pref | Tipo | Default | Efecto |
|---|---|---|---|
| `recibir_actualizaciones` | `bool` | `true` | Avisar de nuevas versiones |
| `recibir_parches` | `bool` | `true` | Avisar de parches |
| `episode_reminders_v1` | `String` (JSON) | — | Recordatorios de estreno |
| — hora del recordatorio | `String` | `'11:00'` | Hora de la notificación diaria |

La versión actual está **hardcodeada** en `core/constants/versiones.dart:9-10`:

```dart
static const String currentVersionName = "1.0.3";
static const int    currentVersionCode = 7;
static const String apiUrl = "https://www.modlyo.com/lol/versiones.php?tipo=lol";
```

---

## 7. AI / Algoritmo

| Pref | Tipo | Default | Efecto |
|---|---|---|---|
| `taste_onboarding_done_v1` | `bool` | `false` | Onboarding de gustos completado |
| `user_taste_profile_v1` | `String` (JSON) | `{}` | `{genres: {k: w}, keywords: {k: w}}` |
| `user_likes_v1` | `String` (JSON) | `{}` | `{tmdbId: {title, type, genres, keywords, likedAt}}` |
| `for_you_recs_v1` | `String` (JSON) | — | Cache de "Para ti" (gate de 20 h) |
| `for_you_marks_v1` | `String` (JSON) | — | Marcas `{t: ms, f: bool}` |
| `daily_home_sections_v1` | `String` (JSON) | — | Cache de secciones diarias |
| `daily_ai_gate_ms_v1` | `int` | `0` | Gate de 24 h |
| `daily_ai_gate_day_v1` | `int` | — | Key legada (compatibilidad) |
| `regional_top10_ai_v2` | `String` (JSON) | — | Cache del Top 10 |
| `regional_top10_ts_v2` | `int` | `0` | Timestamp del Top 10 |
| `kilo_model_memory_v1` | `String` (JSON) | `{}` | `{modelId: {e, f, h}}` |
| `ai_usage_events_v1` | `String` (JSON) | — | Eventos de uso (7 días / 500 máx) |

---

## 8. Addons

| Pref | Tipo | Efecto |
|---|---|---|
| `addons_v1` | `String` (JSON) | Addons instalados. `rawCode` **podado a 80.000 caracteres** |
| `source_packages_v1` | `String` (JSON) | Paquetes Nuvio |
| `addon_settings_v1` | `String` (JSON) | Configuración por addon |
| `stremio_catalog_addons_v1` | `String` (JSON) | Catálogos Stremio |
| `stremio_collections_v1` | `String` (JSON) | Colecciones importadas |
| `stream_badge_packs_v1` | `String` (JSON) | Packs de badges ("corrientes") |
| `extractor_host_methods` | `String` (JSON) | `{'<host>': {'method', 'servidor', 'ts'}}` |
| `custom_api_enabled` | `bool` | Activar "Mis APIs" |
| `custom_api_sources` | `String` (JSON list) | URLs o códigos de las APIs |

---

## 9. TV en vivo (TVChanel)

| Pref | Tipo | Efecto |
|---|---|---|
| `tvchanel_m3u_playlists` | `String` (JSON) | Listas M3U: `{id, name, url, addedAt, enabled}` |
| `tvchanel_favorites_v1` | `String` (JSON `Set<String>`) | IDs de canales favoritos (**solo TV**) |

---

## 10. Cachés del player

| Pref / patrón | TTL | Efecto |
|---|---|---|
| `cachePlayer_<tmdb>[_T.._C..]` | — | Progreso completo |
| `cachePlayerRapido_<tmdb>[_T.._C..]` | — | Progreso rápido (lectura de 1er nivel) |
| `srv_opt_<id>_<S>_<E>_<lang>` | 60 min | Servidor óptimo por idioma |
| `srv_valid_<...>` | 12 h | Lista de servidores válidos |
| `serv_cache_<id>_<tipo>_<S>_<E>` | 12 h configurable | Lista global de servidores |
| `ultimo_enlace_<...>` | 12 h | Último servidor que funcionó |
| `m3u8_cache_v2_<id>_<tipo>_T<S>_C<E>` | 1 h | m3u8 por embed |
| `os_sub_<id>_<subId>` | — | Subtítulo OpenSubtitles cacheado |

---

## 11. Supabase / sesión

| Pref | Tipo | Efecto |
|---|---|---|
| `current_profile_id` | `String` | UUID del perfil activo |
| `current_profile_name` | `String` | Nombre |
| `current_profile_avatar` | `String` | URL del avatar |
| `current_profile_backdrop` | `String` | URL del backdrop |
| `is_guest_mode` | `bool` | Modo invitado |
| `local_profiles_cache` | `String` (JSON) | Caché de perfiles sin conexión |
| `local_historial` | `String` (JSON) | Historial local (**top 100**) |
| `local_guardados` | `String` (JSON) | Guardados locales |
| `local_likes` | `String` (JSON) | Likes locales |

---

## 12. Cómo cambiar una preferencia correctamente

1. **Localiza el string literal** con grep:

   ```bash
   grep -rn "'mi_preferencia'" lib/
   ```

2. **Añádelo a `StorageKeys`** (`core/constants/storage_keys.dart`) — que
   actualmente **no se usa**, pero es el sitio correcto.
3. **Lee y escribe solo desde el servicio** de esa feature, nunca desde la UI
   directamente.
4. Si el valor afecta a una lista en memoria (ej. `FuentesConfig`,
   `MenuPositionPref`), **notifica** para que la UI se refresque.
5. Si es un valor de red que se lee en hot paths, considera cachearlo en
   memoria (ver `TmdbApis.getApiKey()`).

---

## 13. Inconsistencias detectadas

| Pref | Problema |
|---|---|
| `app_mode` | Declarado en `main.dart:25`, `app_config.dart:4` y `storage_keys.dart:2` |
| `disclaimer_accepted` | Igual, triplicado |
| `result.dart` / `success.dart` | Mismo código literal duplicado en `core/result/` |
| `hackstore_enabled`, `pelisplus_enabled` | Existen en `kRegisteredSources` pero **no** en `FuentesConfig.fuentesActivas` |
| `show_unreleased_episodes` | Se declara pero `tmdb_content.dart:64` lo fuerza a `false` |
| `torrent_mkv_enabled` | Filtra servidores que nunca se pueden reproducir |
| `PlayerConstants.speeds` | Constante de velocidades sin ningún control en la UI |
| `seleccion_fuente` | `FuenteSeleccion` existe en dos sitios con posibles valores distintos |