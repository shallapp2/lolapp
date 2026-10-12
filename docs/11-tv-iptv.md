# 11 · TV, D-pad e IPTV (TVChanel)

Documenta dos temas relacionados pero distintos:
1. **Cómo la app se adapta a Android TV** (D-pad, menús, foco)
2. **El módulo TVChanel** (IPTV: listas M3U, canales, zapping)

---

# PARTE 1 · Android TV y D-pad

## 1.1 Configuración del target

El manifest hace que la **misma APK** funcione en móvil y TV:

```xml
<!-- AndroidManifest.xml:22-27 -->
<uses-feature android:name="android.software.leanback"  android:required="false" />
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />

<!-- :45 -->
android:banner="@drawable/banner"

<!-- :51-53 -->
<meta-data android:name="com.google.android.tv" android:value="true" />

<!-- :77-80 -->
<intent-filter>
  <action android:name="android.intent.category.LEANBACK_LAUNCHER" />
</intent-filter>
```

Además: `usesCleartextTraffic=true` (`:47`), `hardwareAccelerated=true`,
`enableOnBackInvokedCallback=true`.

## 1.2 Cómo se activa el modo TV

**No hay detección automática.** El usuario lo elige en el primer arranque
(`main.dart:363 _buildModeSelector`) y se guarda en
`SharedPreferences['app_mode']`. Se puede cambiar en runtime desde Ajustes →
Apariencia.

## 1.3 La máquina de foco D-pad

Es lo más distintivo y mejor diseñado del proyecto. Cada sección expone su
`FocusNode` raíz al shell mediante un callback, y el shell gestiona la
transferencia menú↔contenido.

### Exposición del nodo

```dart
// tv_shell.dart:51-80
final FocusNode _menuFocusNodes = List.generate(9, (i) => FocusNode(debugLabel: 'tv_menu_$i'));
final FocusNode _contentNodes   = List.generate(9, (i) => FocusNode(debugLabel: 'tv_content_$i'));
// Cada página recibe:
onMainFocusNodeCreated: (node) => _contentNodes[i] = node,
onRequestMenuFocus: () => _focusMenu(),
```

### Transferencia al contenido (`_focusContentOf`, `tv_shell.dart:650-703`)

```dart
void _focusContentOf(int index) {
  _transferringToContent = true;
  _transferTimer?.cancel();
  _collapseMenu(); _unfocusMenu();

  void tryFocus([int attempt = 0]) {
    final node = _contentNodeOf(index);
    if (node != null && node.canRequestFocus) {
      _unfocusMenu(); node.requestFocus(); _collapseMenu();
      _transferTimer = Timer(const Duration(milliseconds: 100), () { ... });
      return;
    }
    if (attempt < 12) {                      // 12 reintentos × 40 ms = 480 ms
      Future.delayed(const Duration(milliseconds: 40), () => tryFocus(attempt + 1));
    } else {
      _transferringToContent = false;
      _menuNodeOf(index).requestFocus();     // ← fallback: vuelve al menú
    }
  }
  WidgetsBinding.instance.addPostFrameCallback((_) => tryFocus());
}
```

**Por qué 12 reintentos**: la página destino puede ser un `IndexedStack` que
aún no ha construido su árbol, o un widget que carga datos asíncronamente. El
`postFrameCallback` inicial no basta.

### Handler central (`_onMenuKey`, `tv_shell.dart:758-847`)

```dart
if (_menuPosition == 'top') {
  final topOrder = [home, descubrir, fuentes, search, guardados, tv, descargas, settings, profile];
  if (key == arrowDown)       { _focusContentOf(_currentIndex); }
  if (key == arrowLeft)       { topOrder[i-1].requestFocus(); }
  if (key == arrowRight)      { topOrder[i+1].requestFocus(); }
  if (key == select || enter) { _enterPage(tabIndex); }
} else {
  if (key == arrowDown)       { _menuFocusNodes[i+1].requestFocus(); }
  if (key == arrowUp)         { _menuFocusNodes[i-1].requestFocus(); }
  if (key == arrowRight || select || enter) { _enterPage(tabIndex); }
}
if (key == escape || goBack || backspace) { _confirmExit(); }
return KeyEventResult.ignored;
```

### `ensureVisible` automático

Cuando un elemento recibe foco, hay que|centerlo en pantalla:

```dart
// tv_config_shared.dart — patrón repetido en todas las cards
onFocusChange: (hasFocus) {
  if (hasFocus) Scrollable.ensureVisible(context, alignment: 0.5,
                                          duration: const Duration(milliseconds: 200));
}
```

### Colapso del menú

```dart
// tv_shell.dart:549-573
// Focus listener con debounce de 60 ms
void _onMenuFocusChange(bool focused) {
  _focusTimer?.cancel();
  _focusTimer = Timer(const Duration(milliseconds: 60), () {
    if (mounted) setState(() => _menuActive = focused);
  });
}
```

## 1.4 Menú lateral vs. barra superior

`MenuPositionPref` (`tv_config_shared.dart:24`) es un `ChangeNotifier` con
`ValueNotifier<int> version` que notifica cambios.

| Layout | Fichero | Descripción |
|---|---|---|
| `side` (default) | `_SideMenu` (`tv_shell.dart:2065-2272`) | Rail colapsable **42 px ↔ 140 px** con `BackdropFilter` + reloj al pie |
| `top` | `_TopMenu` (`tv_shell.dart:1757-1896`) | Barra de **56 px** con logo + 5 tabs de texto + 3 iconos + avatar |

```dart
// tv_shell.dart:1318-1362
child: Padding(
  padding: EdgeInsets.only(
    left: _menuPosition == 'top' ? 0 : _railWidth,   // 52
    top:  _menuPosition == 'top' ? _topBarH : 0,      // 56
  ),
  child: IndexedStack(index: _currentIndex, children: [ _buildPage(0), ... ]),
),
```

## 1.5 Reloj

```dart
// tv_shell.dart:156-159
_clockTimer = Timer.periodic(const Duration(seconds: 30), (_) => _updateTime());
// :509
void _updateTime() {
  if (!mounted) return;
  final now = DateTime.now();
  final h = now.hour.toString().padLeft(2, '0');
  final m = now.minute.toString().padLeft(2, '0');
  setState(() => _clock = '$h:$m');
}
```

## 1.6 Botón "mira" en WebView

Problema: el control remoto no generaba clicks en el `WebView` embebido.
Solución: `MethodChannel 'tv_webview/click'` (`MainActivity.kt:22, 129-162`):

```kotlin
// MainActivity.kt:129-162
fun injectClick(x: Float, y: Float) {
  val root = window.decorView.findViewById<ViewGroup>(android.R.id.content) ?: return
  findWebView(root)?.let { webView ->
    val motionEvent = MotionEvent.obtain(
      SystemClock.uptimeMillis(), SystemClock.uptimeMillis(),
      MotionEvent.ACTION_DOWN, x, y, 0)
    webView.dispatchTouchEvent(motionEvent)
    // ... ACTION_UP
  }
}
```

---

# PARTE 2 · TVChanel (IPTV)

## 2.1 ¿Qué es?

Módulo de **TV en vivo**: gestiona listas M3U/M3U8 remotas y addons de canales
locales/GitHub, los parsea, agrupa por categoría y reproduce el canal con un
player **live** (sin seek, sin barra de progreso).

Tiene su propio `README.md` (155 líneas) con la especificación del formato de
addons.

## 2.2 Estructura

```
features/tvchanel/
├── README.md                     155 L   ← spec del formato de addons
├── models/tv_channel_models.dart 218 L
├── data/
│   ├── m3u_parser.dart           123 L
│   └── tvchanel_repository.dart  459 L
├── config/
│   ├── m3u8_page.dart            225 L   móvil
│   ├── m3u8_page_tv.dart         611 L   TV
│   ├── addon_chanel.dart         217 L   móvil
│   └── addon_chanel_tv.dart      897 L   TV
├── home/
│   ├── home_tvchanel.dart        292 L   móvil
│   └── home_tvchanel_tv.dart    1101 L   TV (3 columnas + preview)
└── player/
    ├── player_tvchanel.dart      490 L   móvil
    └── player_tvchaneltv.dart   1021 L   TV (fullscreen + guía de canales)
```

## 2.3 Modelos (`models/tv_channel_models.dart`)

| Clase | Líneas | Campos |
|---|---|---|
| `TvChannel` | 4-72 | `id, name, url, logo, group, country, language, extra`. `fromJson` acepta `group ?? category` (`:31`). `copyWith` completo |
| `TvCategory` | 75-98 | `name + List<TvChannel>`, default `"General"` |
| `TvAddonManifest` | 106-148 | `id, name, version, description, author, logoPath, website` |
| `TvAddon` | 151-166 | `manifest + categories + folderPath + config`, getter `totalChannels` |
| `M3uPlaylist` | 169-218 | `id (uuid), name, url, addedAt, enabled` |

## 2.4 Parser M3U (`data/m3u_parser.dart`)

```dart
// m3u_parser.dart:6-81
static List<TvChannel> parse(String content, {String sourceName = 'M3U'})
```

- Línea `#EXTINF:-1 tvg-id="..." tvg-logo="..." group-title="News",CNN`:
  - Nombre = lo que va **después de la última coma** (`:32-35`)
  - Atributos = lo que va antes (`:37-38`)
- Línea que no empieza por `#` → URL del stream → crea `TvChannel` con
  `id = '${sourceName}_${url.hashCode.abs()}'` (`:60`)

### Atributos soportados

| Atributo | Alias | Campo |
|---|---|---|
| `tvg-logo` | `logo` | `logo` |
| `group-title` | `group` | `group` |
| `tvg-country` | `country` | `country` |
| `tvg-language` | `language` | `language` |
| `tvg-id` | — | `extra['tvg_id']` |

`_extractAttr()` (`:107-122`) con regex para `key="value"` y `key='value'`
(case-insensitive).

### Agrupación por categoría (`groupByCategory`, `:84-105`)

```dart
// Agrupa por 'group', "General" SIEMPRE al final, resto alfabético
```

### ⚠️ Lo que NO soporta

- `tvg-chno` (número de canal)
- `#EXTVLCOPT:http-referrer` (URL tokens)
- `x-tvg-url`
- Streams `.ts` de doble línea
- **XMLTV → no hay EPG real**

## 2.5 Repositorio (`data/tvchanel_repository.dart`)

`TvChanelRepository.instance` — singleton (`:17`).

### 2.5.1 Listas M3U

Persistencia en `SharedPreferences` key `tvchanel_m3u_playlists` (`:12`).

| Método | Líneas |
|---|---|
| `getPlaylists` / `savePlaylists` | 23-42 |
| `addPlaylist` (uuid v4) | 44-56 |
| `removePlaylist` | 58-63 |
| `togglePlaylist` | 65-73 |
| `fetchM3uChannels(playlist)` | 76-94 |
| `getAllM3uCategories()` | 97-112 |

`fetchM3uChannels` (`:76-94`):

```dart
final res = await http.get(Uri.parse(playlist.url), headers: {
  'User-Agent': 'Mozilla/5.0 ... Chrome/120 Safari/537.36',
  'Accept': '*/*',
}).timeout(const Duration(seconds: 20));
if (res.statusCode != 200) throw Exception('HTTP ${res.statusCode}');
```

`getAllM3uCategories()` (`:97-112`): itera **solo playlists habilitadas** en
serie, con `continue` **silencioso** si una falla (`:106`).

### 2.5.2 Addons locales

Directorio: `Documents/tvchanel_addons/<addon>/` (`:118-125`).

`loadAddons()` (`:128-240`):

1. Por cada subcarpeta exige `manifest.json`
2. Busca `logo.png` / `logo.jpg` / `logo.jpeg`; si no hay, usa el sentinel
   `.nologo` y acepta **URL de logo** en `manifest.logo`/`manifest.icon` (`:171-176`)
3. Lee `config.json` (opcional)
4. **Prioridad de la fuente de la lista** (`:184-188`):

   ```
   config.m3u_url  →  config.m3u  →  manifest.m3u_url  →  manifest.m3u
   ```

   - Si es `http`: descarga y parsea en vivo (`:189-199`)
   - Si falla o no hay: `index.json` con `categories[]` o `channels[]` plano
     (`:202-219`)
5. Addons sin categorías se ignoran (`:221`)
6. Orden alfabético por nombre (`:234-238`)

### 2.5.3 `installFromGithub(ownerRepo, {branch})` (`:260-383`)

Acepta `owner/repo`, `owner/repo/path`, o URL completa de GitHub.

1. Intenta `main` y luego `master` (`:282-292`)
2. Descarga y escribe `manifest.json` (con indentación), `index.json`, `config.json`
3. Si el manifest trae `m3u_url` y no hay config, **crea el config** (`:331-337`)
4. Logo: `logo.png`/`logo.jpg`/`logo.jpeg`/`icon.png` → guarda como `logo.png`
   (`:341-351`), o descarga de URL (`:352-364`)
5. **Borra la carpeta previa** (`:306-308`)

`createExampleAddon()` (`:393-458`): genera un addon de ejemplo con
`test-streams.mux.dev` y el sample fMP4 de Apple.

## 2.6 Player en vivo

### 2.6.1 Móvil (`player/player_tvchanel.dart`, 490 líneas)

```dart
PlayerTvChanel({required TvChannel channel, required List<TvCategory> categories})  // :16-28
```

- Landscape forzado + `immersiveSticky` + `WakelockPlus` (`:53-58`)
- **`_init()` con 7 intentos de User-Agent** (`:123-219`)
- Controles:
  - Barra superior con **back**, **minimizar**, nombre del canal,
    **canal anterior/siguiente** (`:420-470`)
  - Play/pause central de 64 px (`:471-483`)
  - Auto-hide a 4 s (`:227-233`)
- **Minimizar → mini-player** (`:251-299`): handoff del controller con
  `tipo: 'live'`, `idcontenido: _channel.id.hashCode`, `poster: logo`
- `_next(delta)` (`:333-341`): navega por la lista plana de todos los canales de
  todas las categorías, con wraparound módulo
- `PopScope(canPop: false)` + diálogo "Salir del canal" (`:301-331, 366-370`)

### 2.6.2 TV (`player/player_tvchaneltv.dart`, 1021 líneas)

- `PlayerTvChanelTv` (`:17-31`) → init (`:56`), reloj (`:73-81`), `_initPlayer`
  (`:82-223`)
- **`_ChannelGuide` (`:561-880`)**: overlay con **2 filas de foco** — tabs de
  categorías (↑↓ lateralmente, ←→ verticalmente) + lista de canales de la
  categoría, con `ensureVisible` automático. Footer con ayuda de teclas
  (`:669-672`)
- `_TvExitConfirmDialog` (`:882-961`)
- `_switchChannel` (`:230-242`) / `_nextChannel(delta)` (`:244-251`)

> ⚠️ `_ChannelGuide` es una **guía de canales (zapping)**, **NO** una EPG de
> programación.

## 2.7 Home

### 2.7.1 Móvil (`home/home_tvchanel.dart`, 292 líneas)

```dart
HomeTvChanel  // :13-18
```

- **NO tiene Favoritos**, **NO tiene búsqueda**, **NO tiene EPG**
- `_load()` (`:32-71`): `Future.wait([getAllM3uCategories(), loadAddons()])` en
  paralelo, **mergea por nombre de categoría** (`:45-57`)
- Header: "TV en Vivo" + 3 botones (Listas M3U8, Addons, Actualizar) (`:104-130`)
- `_CategoryBlock` (`:210-292`): cards horizontales de 110×120 con logo + nombre
- Estado vacío con CTA "Añadir lista M3U8" (`:141-186`) + `RefreshIndicator`
  (`:187-201`)

### 2.7.2 TV (`home/home_tvchanel_tv.dart`, 1101 líneas) ⭐

```dart
HomeTvChanelTv({required VoidCallback onRequestMenuFocus,
                required ValueChanged<FocusNode> onMainFocusNodeCreated})  // :24-36
```

**Layout de 3 columnas** (`:20-23`):

```
[ Categorías ] [ Lista de canales ] [ Player preview ]
```

| Zona | Índice | Contenido |
|---|---|---|
| 0 | 0 | Categorías (virtuales `"ALL"` y `"Favorito"` + reales, `:74-83`) |
| 1 | 1 | Lista de canales |
| 2 | 2 | **Player preview embebido** |

- **Preview player**: `VideoPlayerController` (`:52-56`). Seleccionar un canal
  **actualiza el preview sin salir del home**; OK sobre el player → pantalla
  completa (`_selectChannel` `:234-302`, `_openFullscreen`)
- **Búsqueda por nombre** con `TextEditingController` + filtro case-insensitive
  (`:71-72, 101-105`)
- **Favoritos** (`:18, 140-165`): key `tvchanel_favorites_v1` en prefs,
  `Set<String>` de `channel.id`; toggle desde la lista (`:733, 806`) y desde el
  panel del player (`:822, 954-964`)
- **Foco por zonas** (`:59`): `_zone` 0=cat / 1=canales / 2=player, con 8
  `FocusNode` nombrados (`:60-67`), `_moveCat`/`_moveCh` (`:304-332`),
  `_ensureCatVisible`/`_ensureChVisible` (`:333-347`)
- 3 paneles: `_buildTopBar` (`:503`), `_buildCategoriesPanel` (`:612`),
  `_buildChannelsPanel` (`:708`), `_buildPlayerPanel` (`:820`)

### 2.7.3 ⚠️ El botón EPG no hace nada

```dart
// home_tvchanel_tv.dart:65, 978-987
// onPressed: → ScaffoldMessenger.of(context).showSnackBar(
//              SnackBar(content: Text('EPG próximamente')))
```

La EPG **no está implementada**. No hay parseo de XMLTV, ni `tvg-chno`, ni
`start`/`stop`.

## 2.8 Configuración

### 2.8.1 Listas M3U8

`config/m3u8_page.dart` (225, móvil) y `m3u8_page_tv.dart` (611, TV con D-pad,
`_M3uActionBtn`, `_goBack`).

- `_addPlaylist()` → diálogo con **Nombre** + **URL (.m3u / .m3u8)**
- Acciones por ítem: Activar/Desactivar, Eliminar
- Chip "Activa"/"Inactiva" con color de acento

### 2.8.2 Addons de canales

`config/addon_chanel.dart` (217, móvil) y `addon_chanel_tv.dart` (897, TV).

- Sección "Instalados (N)" con logo, nombre, "N canales" y botón borrar
- Sección "Comunidad (lol-tvchanel)" — `CommunityService.fetchTvChanelAddons()`
  busca en GitHub por topic
- `_install(ownerRepo)` → `TvChanelRepository.installFromGithub()`
- Instalación manual: diálogo Owner + Repo
- TV: navegación D-pad completa + **long-press para menú contextual**
  (`_NavCard`, `_ActionModal`, `_InstalledRow`, `_CommunityRow`)

## 2.9 Integración en Ajustes

`features/settings/presentation/tvchanel/tv_chanel_tab.dart` (223 líneas):
`TvChanelTab` con 2 `_ConfigCard` D-pad:

1. "Listas M3U8" → `M3u8PageTv` (`:37-45`)
2. "Addons de Canales" → `AddonChanelTv` (`:47-55`)

`_ConfigCard` local (`:109-223`): `Focus` + `onKeyEvent` con
OK/Enter → `onTap`, ↑→ `onUp`, ↓ → `onDown`, ← → `onLeft` (volver al menú
lateral). Reutiliza `kConfigAccent`/`kConfigCard` de `tv_config_shared.dart`.

## 2.10 Acceso desde el shell

| Shell | Sección | Widget |
|---|---|---|
| Móvil | Toggle en el AppBar de Home (`mobile_shell.dart:1425-1455`) | `HomeTvChanel(key: ValueKey('tvLiveHome'))` |
| TV | Índice 8 (`_kTvIndex = 8`) | `HomeTvChanelTv` |

En móvil se cambia entre modo VOD y modo TV en vivo con el toggle
`vod.png` / `tvlive.png` del AppBar.

---

## 2.11 Estado de funcionalidades

| Pedido | Estado |
|---|---|
| Configuración de lista M3U | ✅ Completo (add/toggle/delete, persistido) |
| Parseo de canales | ✅ `#EXTINF` completo (tvg-id/logo/group/country/language) |
| **EPG** | ❌ **NO EXISTE** — solo un Snackbar placeholder. No hay XMLTV, ni `tvg-chno`, ni `start`/`stop` |
| Reproductor TV en vivo | ✅ Excelente: 7 UAs, hints HLS/DASH, errores mapeados, zapping |
| **Guide** | ✅ `_ChannelGuide` en el player TV (navegación de canales), ❌ en móvil |
| Favoritos | ✅ Solo TV (`home_tvchanel_tv.dart`); ❌ en `home_tvchanel.dart` |
| Búsqueda de canales | ✅ Solo TV; ❌ en móvil |
| Mini-player de canal | ✅ Móvil; TV no tiene mini-player |
| Addons de canales de comunidad | ✅ GitHub topic `lol-tvchanel` |

---

## 2.12 Ejemplo de addon de canal

Según `features/tvchanel/README.md`, un addon es una carpeta en
`Documents/tvchanel_addons/<nombre>/` con:

```
Documents/tvchanel_addons/miaddon/
├── manifest.json
├── index.json         (o el M3U remoto vía config.json)
├── config.json        (opcional, para m3u_url)
└── logo.png           (o logo.jpg / logo.jpeg / URL en manifest)
```

`index.json` admite dos formas:

```json
// Opción A: con categorías
{
  "categories": [
    { "name": "Deportes", "channels": [ { "name": "ESPN HD", "url": "https://...m3u8", "logo": "..." } ] }
  ]
}

// Opción B: plana
{ "channels": [ { "name": "Canal", "url": "https://...m3u8" } ] }
```

Alternativamente, `config.json`:

```json
{ "m3u_url": "https://ejemplo.com/lista.m3u" }
```

---

## 2.13 Problemas

1. **La EPG está anunciada en la UI pero no existe** (botón que muestra
   "EPG próximamente"). Es una funcionalidad *prometida y ausente*.
2. **Móvil sin búsqueda, favoritos ni guide** — asimetría con TV.
3. El parser M3U no soporta `#EXTVLCOPT:http-referrer`, `tvg-chno` ni URLs
   con tokens, que son comunes en listas IPTV reales.
4. `getAllM3uCategories()` itera **en serie** con `continue` silencioso: una
   lista caída no avisa al usuario.
5. El "player preview" del home TV mantiene un `VideoPlayerController` vivo
   permanentemente — consumo de CPU/batería constante.