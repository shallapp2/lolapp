# 13 · Features de UI

Inventario de todas las pantallas y flujos de usuario.

---

## 1. Patrón común

Todas las features usan:

- `StatefulWidget` + `setState` (sin providers)
- **Caché estática a nivel de clase** (`static Map<String,dynamic>? _cachedData`
  en `home_page.dart:40-42`) para no recargar al volver con tab
- `SharedPreferences` directo para todo
- `AutomaticKeepAliveClientMixin` cuando vive dentro de un `IndexedStack`
- Variante `tv_*` / `*_tv` con foco D-pad
- `debugPrint` para logs (el `Logger` de `core/utils/logger.dart` no se usa)

---

## 2. Mapa de las 17 features

| Feature | Ficheros | Descripción |
|---|---|---|
| `player/` | 28 | Reproductor — ver [10-reproductor.md](10-reproductor.md) |
| `settings/` | 28 | 13 sub-secciones de configuración |
| `discover/` | 13 | Catálogo por fuentes |
| `tvchanel/` | 11 | IPTV — ver [11-tv-iptv.md](11-tv-iptv.md) |
| `addons/` | 11 | Hub de addons, comunidad, packs Nuvio |
| `content/` | 9 | Ficha de película/serie |
| `profile/` | 9 | Selector estilo Netflix + CRUD |
| `downloads/` | 8 | Descargas HLS segmentadas |
| `home/` | 7 | Home VOD + secciones carousel |
| `servers/` | 7 | Modal de servidores + precargadores |
| `foryou/` | 5 | Motor "Para ti" + onboarding de gustos |
| `search/` | 2 | Buscador global TMDB |
| `auth/` | 2 | Login / Registro |
| `favorites/` | 1 | `BibliotecaUnificadaPage` |
| `history/` | 1 | Versión TV de la biblioteca |
| `lolbot/` | 1 | Chat de IA (solo móvil) |

---

## 3. Home

| Fichero | Líneas | Rol |
|---|---|---|
| `features/home/presentation/home_page.dart` | 1669 | `HomePage` móvil |
| `features/home/presentation/tv_home_page.dart` | 1970 | Home TV |
| `movie_section.dart` / `series_section.dart` | 692 / 692 | Secciones TMDB móvil |
| `tv_movie_section.dart` / `tv_series_section.dart` | 691 / 691 | Secciones TV |

**Servicio**: `TmdbHomeMobileService` (837 L) / `TmdbHomeService` (841 L)
→ ver [08-tmdb.md §3](08-tmdb.md)

### 3.1 Flujo de carga (`home_page.dart`)

```dart
// initState (:64-87)
if (_hasLoadedOnce && _cachedData != null) {
  // reusa caché y refresca historial en silencio
} else {
  _fetchAll();
}

// _fetchAll (:102-147) — en paralelo
_fetchHomeApi()      // 11 requests a TMDB
_loadHistorial()     // progreso desde prefs + Supabase

// _loadStremioRows (:149-230) — después
StremioAddonRepository.instance.buildHomeSections()          // catálogos
StremioCollectionRepository.instance.buildHomeSectionsFromCollections()  // Stremio/Nuvio
// + DailySectionsGenerator + RegionalTop10Service (TV: _loadAlgoSections :141-206)
```

### 3.2 Componentes de `home_page.dart`

| Widget | Líneas |
|---|---|
| `_HeroSlide` | 773 |
| `_SectionHeader` | 966 |
| `_ContinueCard` | 1000 (con barra de progreso) |
| `_TabbedSection` | 1188 |
| `_PosterCard` | 1302 |
| `_EpisodeSection` | 1455 |
| `_EpisodeCard` | 1503 |

### 3.3 Navegación

Abre `PageContenido` (detalle), `PlayerScreen` directo (`:278`),
`ContentOptionsModal`, `AddonsOnboardingPage`, `LolbotPage`, `ForYouSection`.

### 3.4 Tema

```dart
// home_page.dart:19
const kAccentColor = Colors.purpleAccent;   // ⚠️ NO es el rojo del resto de la app
// fondo negro, card #1a1a2e
```

---

## 4. Discover (Descubrir / Servicios)

| Fichero | Líneas | Rol |
|---|---|---|
| `presentation/discover_page.dart` | 1041 | `EscrubirPage` |
| `presentation/tv_discover_page.dart` | 1099 | Versión TV |
| `presentation/source_search.dart` | 1320 | Buscador por fuente |
| `presentation/source_discovery_page.dart` | 419 | "Derivar" contenido desde una URL |
| `presentation/tv_source_discovery_page.dart` | 419 | Idem TV |
| `domain/extractor.dart` | 1327 | ⚠️ UI en carpeta `domain/` |
| `domain/derivar.dart` | 1418 | Idem |
| `domain/mobile/player_screen.dart` | 2060 | Player de Descubrir |
| `domain/mobile/extractor.dart` | 979 | Extractor móvil |
| `domain/mobile/derivar.dart` | 677 | Derivación móvil |
| `domain/mobile/pag.dart` | 1165 | `ServiciosPage` (la pantalla principal) |
| `domain/mobile/servidores_modal.dart` | 938 | Modal de servidores |
| `domain/mobile/hls_quality_widget.dart` | 690 | Selector de calidad |

**Servicio**: `TmdbDiscoverApi` (265 L)

**Componentes**: `_PosterTile` (`:766`, con precarga de imagen al enfocar/long-press),
`_FilterPill` (`:959`)

⚠️ **Ocho de los trece ficheros de esta feature están en carpetas `domain/`** y
son **código de UI** (`StatelessWidget`, `build()`, `Colors`, etc.). Contradice
la arquitectura declarada.

**"Derivar contenido"** (`source_discovery_page.dart`): búsqueda inversa por
título para identificar a qué película/serie corresponde una URL de servidor.

---

## 5. Buscar

| Fichero | Líneas |
|---|---|
| `features/search/presentation/search_page.dart` | 1382 |
| `features/search/presentation/tv_search_page.dart` | 1106 |

- `_doSearch()` (`:123`)
- `_loadDiscover({reset})` (`:250`) con **paginación incremental** + sección
  "Descubrir" cuando no hay resultados
- Componentes: `_FilterChipButton` (`:1193`), `_DiscoverPosterCard` (`:1247`)
- TV: expone `BuscarPageState` al shell para poder inyectar foco desde el menú
  (`tv_shell.dart:81`, vía `getAKeyFocusNode()`)

---

## 6. Ficha de contenido

| Fichero | Líneas | Rol |
|---|---|---|
| `content_page.dart` | 3037 | `PageContenido` + `GuardadosBus` + `GuardadosCache` |
| `tv_content_page.dart` | 3943 | Versión TV |
| `content_options_modal.dart` | 615 | Modal de opciones (guardar, ver después, likes) |
| `tv_content_options_modal.dart` | 659 | Versión TV |
| `content_info.dart` | 530 | Ficha de datos extendida |
| `cast_modal.dart` | 342 | Reparto |
| `seasons_episodes.dart` | 275 | Selector de temporadas/episodios |
| `widgets/upcoming_episodes_modal.dart` | 581 | Próximos episodios |
| `widgets/tmdb_upcoming_service.dart` | 206 | Servicio `next_episode_to_air` |

**Servicio**: `TmdbContent` (854 L) + OMDb → ver [08-tmdb.md §7](08-tmdb.md)

### 6.1 Buses globales

```dart
// content_page.dart:33
class GuardadosBus { static final ValueNotifier<int> version = ...; static void bump() ...; }
// content_page.dart:39
class GuardadosCache { /* caché en memoria de guardados */ }
```

`GuardadosBus.bump()` lo llama el player al guardar progreso.

### 6.2 Flujo

| Método | Líneas | Qué hace |
|---|---|---|
| `_fetchContent` | 139 | Carga la ficha |
| `_loadSavedState` | 179 | ¿Está guardado? |
| `_loadProgress` | 184 | Barra de progreso |
| `_loadEpisodeProgress` | 233 | Progreso por episodio |
| `_loadDownloadedState` | 271 | ¿Descargado? |
| `_toggleSaved` | 322 | Guardar/quitar |
| `_openEpisodeModal` | 485 | Modal de capítulos |
| `_openServidores` | 520 | → `ServidoresModal` |
| `_startDownload` | 604 | Iniciar descarga |
| `_autoDownloadAndOpenExtractor` | 630 | Descarga + extractor |
| `_openYoutube` | 816 | Trailer |
| `_openLetterboxd` | 830 | Ficha externa |
| `_checkAiring` | 867 | Estado de emisión |

---

## 7. "Para ti" (foryou)

| Fichero | Líneas |
|---|---|
| `for_you_section.dart` | 412 |
| `taste_onboarding_page.dart` | 403 |
| `taste_onboarding_page_tv.dart` | 668 |
| `profile_recap_page.dart` | 330 |
| `like_button.dart` | 85 |

Ver [12-ia.md](12-ia.md) para el motor completo.

---

## 8. Servidores (el modal más complejo)

| Fichero | Líneas | Rol |
|---|---|---|
| `servers_modal.dart` | 1923 | `ServidoresModal` móvil |
| `tv_servers_modal.dart` | 2160 | Versión TV |
| `tv_servers_modal_fuentes.dart` | 1242 | TV para `tv_discover_player` |
| `server_preloader.dart` | 276 | Precargador |
| `server_preloader_service.dart` | 214 | Servicio |
| `tv_server_preloader.dart` | 275 | Precargador TV |
| `tv_server_preloader_service.dart` | 214 | Servicio TV |

### 8.1 `ServidoresModal` — estructura

- **Pestañas por fuente** (`_TabKey` `:193`), reordenables según el contenido
  (`:384-431`)
- `TvM3u8Cache` (`:58-191`): caché de m3u8 **1 h con timestamp POR ENTRADA**,
  escritura **serializada** con cadena de futures (`:69-81`) para evitar carreras
- Flujo: `_startFlow` (`:549`) → `_addServerToBuckets` (`:681`) →
  `_ingestCached` (`:725`) → `_hydrateM3u8FromCache` (`:750`) →
  `_runManual` (`:778`) → `_persistCache` (`:903`)

### 8.2 `_openServer(servidor)` (`:940-1085`) — 3 destinos

```dart
1. widget.forDownload  → ExtractorDownloadPage           (:1001-1022)
2. m3u8 fresco o URL directa → PlayerScreen              (:1028-1060)
   // playHeaders del servidor — el comentario del código:
   // "Headers del addon/stream (Referer/Origin/Cookie) — sin esto CDN → 403"
3. Si no → WebPlayerView (WebView)                        (:1062-1084)
```

Si `reutilizarUltimoEnlace` → guarda `FuentesCache.saveLastLink()` antes de
navegar (`:970-984`). `_schedulePreload()` tras cada navegación (`:894`).

### 8.3 Filtro torrent/MKV (`:663-680`)

Detecta `.torrent`, `magnet:`, `torrent:`, `infoHash`, `fmt=='mkv'`. Si
`torrent_mkv_enabled == false` los descarta (`:682`). **Nunca se reproducen**.

### 8.4 Badges (`:19-47`)

`_applyCorrienteBadges` → `StreamBadgeRepository.matchAllBadges` sobre
nombre + calidad + provider.

---

## 9. Addons

Ver [07-addons.md §13](07-addons.md) para el inventario de pantallas.

Entrada en Ajustes:
- Móvil: `settings_page.dart:1059-1067` → `AddonsHubPage(isTv: false)`
- TV: `tv_settings.dart:52, 196` → `TvAddonsTab`

---

## 10. Auth y Perfiles

Ver [09-supabase.md §11](09-supabase.md).

---

## 11. Descargas

| Fichero | Rol |
|---|---|
| `download_manager.dart:123` | `DownloadManager.instance` — `ChangeNotifier` |
| `downloads_page.dart` / `downloads_page_tv.dart` | Páginas de descargas (móvil/TV) |
| `local_player_*.dart` | Reproducción local |
| `extractor_download_page*.dart` | Descarga desde extractor |
| `download_helper.dart` | Notificaciones |

**Características**:
- Descargas HLS **segmentadas** con reintentos por segmento (backoff 1s/2s/4s)
- `FlutterForegroundTask` con canal `downloads_channel`, `allowWakeLock/WifiLock`
- Configuración: automática, concurrencia (`max_concurrent_downloads`), calidad
  preferida, carpeta
- `idm_download_enabled`: delega la descarga a IDM externo

---

## 12. Biblioteca (favorites + history)

| Fichero | Rol |
|---|---|
| `features/favorites/presentation/favorites_page.dart:40` | `BibliotecaUnificadaPage` — tabs `Guardados / Historial / Descargas` (`_Tab` en `:31`) |
| `features/history/presentation/history_page.dart:92` | `GuardadosPage` — versión TV |

Filtros de películas/series, modo selección múltiple y borrado.

---

## 13. Configuración (28 ficheros, 13 sub-secciones)

### 13.1 Raíz

| Fichero | Líneas | Rol |
|---|---|---|
| `settings_page.dart` | 1321 | `ConfigPage` móvil con lista de `_buildSectionCard` |
| `tv_settings.dart` | 356 | Config TV |
| `tv_settings_page.dart` | 329 | Implementación TV con 9 tabs |
| `config_shared.dart` | 48 | `kAccentColor = #E50914`, `kCardColor = #1C1C1E`, enums `IdiomaPred`, `SubSize`, `SubHeight`, `FuenteSeleccion` |
| `tv_config_shared.dart` | 1146 | `MenuPositionPref` (ChangeNotifier) + 8 widgets de card con foco: `SourceToggleCard`, `FocusActionCard`, `IdiomaSelectorCard`, `SubSizeSelectorCard`, `MenuPositionCard`, `VerticalOffsetCard` |

### 13.2 Las 13 sub-secciones

| Subcarpeta | Fichero principal | Contenido |
|---|---|---|
| `sources/` | `sources_section.dart` (1048) | **El más denso.** Master switch `native_sources_enabled`, toggle por fuente, `verificar_servidores`, `un_servidor_por_idioma`, `mostrar_servidores_player`, `reutilizar_ultimo_enlace`, TTL de caché, concurrencia 1-5, `seleccionar_servidores` auto/manual, idioma de audio, APIs personalizadas |
| `updates/` | `tv_updates_tab.dart` (1703), `update_notification_service.dart` | Auto-actualización con selector de ABI |
| `actualizaciones/` | `actualizaciones_page.dart` (1008) | Equivalente móvil, `open_filex` |
| `player/` | `player_section.dart` (609) | Mini-player, subtítulos al iniciar, `SubSize`, `SubHeight`, descarga IDM, estilo de subtítulo (fondo/contorno/color) |
| `notifications/` | `notifications_section.dart` (696) | `EpisodeReminderService`, canal + hora (11:00 por defecto) |
| `downloads/` | `downloads_section.dart` (304) | Automático, concurrencia, calidad, carpeta |
| `content/` | `content_section.dart` (332), `tmdb_api_options.dart` (173) | TMDB API key, idioma metadata, secciones home, filtros |
| `cache/` | `cache_section.dart` (115) | Borrar historial / guardados / caché servidores / todo |
| `account/` | `account_settings_page.dart` (301) | Cambiar password, generar/vincular código TV |
| `ai/` | `ai_usage_section.dart` (303) | Cuota de IA (200/h) |
| `appearance/` | `appearance_section.dart` (108) | Cambiar modo móvil ↔ TV, posición del menú |
| `addons/` | `tv_addons_tab.dart` (177) | Acceso al hub de addons |
| `tvchanel/` | `tv_chanel_tab.dart` (223) | Listas M3U8 y addons de canales |
| `supabase/` | `supabase_section.dart` (96) | Estado de sesión y perfiles |

### 13.3 Sección de Cache (`:115` móvil, `:158` TV)

4 acciones:
1. Borrar historial
2. Borrar guardados
3. **Borrar caché de servidores**
4. Borrar todo

TV: `_clearByPrefix(prefixes, msg)` (`:51`) — borrado por prefijo de key.

### 13.4 Sección de Notificaciones (`:696`)

`EpisodeReminderService` (singleton):
- Canal `episode_premiere_channel`
- Hora por defecto **11:00** (`:74`)
- `timezone` + `tzdata`
- Persistencia `episode_reminders_v1`
- `_startTestCountdown` (`:145`) + `_fireTestNotification` (`:193`) para probar
- `_cancelOne` (`:231`) / `_cancelAll` (`:243`)

Permisos: `SCHEDULE_EXACT_ALARM`, `USE_EXACT_ALARM`, `RECEIVE_BOOT_COMPLETED`.

### 13.5 Sección de Actualizaciones

`VersionService` (`core/constants/versiones.dart`) → `https://www.modlyo.com/lol/versiones.php?tipo=lol`
con `currentVersionName = "1.0.3"` y `currentVersionCode = 7`.

- Selector de ABI (arm64 / armeabi / x86 / universal) para elegir el APK
- `_downloadAndInstallApk` (`actualizaciones_page.dart:444`) vía `open_filex` +
  permiso `REQUEST_INSTALL_PACKAGES`
- `qr_flutter` en la versión TV
- `UpdateNotificationService` se lanza en `main.dart:42` (fire-and-forget)

---

## 14. Móvil vs TV: tabla de equivalencias

| Feature | Móvil | TV |
|---|---|---|
| Home | `home_page.dart` (1669) | `tv_home_page.dart` (1970) |
| Contenido | `content_page.dart` (3037) | `tv_content_page.dart` (3943) |
| Opciones | `content_options_modal.dart` (615) | `tv_content_options_modal.dart` (659) |
| Servidores | `servers_modal.dart` (1923) | `tv_servers_modal.dart` (2160) + `tv_servers_modal_fuentes.dart` (1242) |
| Player | `player_page.dart` (2930) | `tv_player_page.dart` (4505) + `tv_discover_player.dart` (2943) + `tv_player_webview.dart` (1821) |
| Subtítulos | `subtitle_selector.dart` (1758) | `tv_subtitle_selector.dart` (2055) |
| Calidad | `quality_selector.dart` (690) | `tv_quality_selector.dart` (690) |
| Buscar | `search_page.dart` (1382) | `tv_search_page.dart` (1106) |
| Config | `settings_page.dart` (1321) | `tv_settings.dart` (356) + `tv_settings_page.dart` (329) |
| Perfiles | `profile_selection_page.dart` (666) | `tv_profile_selection_page.dart` (1012) + `tv_picker_page.dart` (939) |
| Editar perfil | `edit_profile_page.dart` (290) | `tv_edit_profile_page.dart` (1560) |
| Gestionar perfiles | `manage_profiles_page.dart` (159) | `tv_manage_profiles_page.dart` (338) |
| Login | `login_register_page.dart` (565) | `tv_login_register_page.dart` (961) |
| Descargas | `downloads_page.dart` | `downloads_page_tv.dart` |
| Biblioteca | `favorites_page.dart` | `history_page.dart` |
| TV en vivo | `home_tvchanel.dart` (292) | `home_tvchanel_tv.dart` (1101) |
| Player TV | `player_tvchanel.dart` (490) | `player_tvchaneltv.dart` (1021) |
| M3U8 | `m3u8_page.dart` (225) | `m3u8_page_tv.dart` (611) |
| Addons canal | `addon_chanel.dart` (217) | `addon_chanel_tv.dart` (897) |
| Addons | `addons_hub_page.dart` | `tv_addons_actuales_page.dart` / `tv_addons_comunidad_page.dart` |
| Onboarding gustos | `taste_onboarding_page.dart` (403) | `taste_onboarding_page_tv.dart` (668) |
| Discover | `domain/mobile/pag.dart` (1165) | `tv_discover_page.dart` (1099) |
| Lolbot | `lolbot_page.dart` (358) | ❌ **No existe** |
| Player Descubrir | `domain/mobile/player_screen.dart` (2060) | `tv_discover_player.dart` (2943) |
| Extractor Descubrir | `domain/mobile/extractor.dart` (979) | `features/player/data/extractor.dart` (1325) |

**Total de líneas duplicadas entre variantes móvil/TV**: >25.000.

---

## 15. Resumen de hallazgos por pantalla

| Pantalla | Líneas | Problema |
|---|---|---|
| `tv_player_page.dart` | 4505 | Clase gigante; imposible de mantener |
| `tv_content_page.dart` | 3943 | Ídem |
| `player_page.dart` | 2930 | Ídem |
| `tv_discover_player.dart` | 2943 | 3ª copia de la lógica de player |
| `tv_servers_modal.dart` | 2160 | Copia del modal |
| `tv_subtitle_selector.dart` | 2055 | Copia |
| `domain/mobile/player_screen.dart` | 2060 | 4ª copia |
| `mobile_shell.dart` | 2023 | Shell |
| `tv_shell.dart` | 2600 | Shell |
| `servers_modal.dart` | 1923 | UI + caché + verificación en un archivo |
| `tv_player_webview.dart` | 1821 | 5ª copia |
| `subtitle_selector.dart` | 1758 | UI + API + caché + SRT→VTT |
| `content_page.dart` | 3037 | UI + llamadas TMDB + `GuardadosBus` |
| `discovery_page.dart` / `domain/mobile/pag.dart` | 1165 / 1041 | UI dentro de `domain/` |

**Ningún fichero supera las 600 líneas con una responsabilidad clara.** El
límite Rajiv/Dalmia (400-600 líneas) se incumple en ~20 ficheros.