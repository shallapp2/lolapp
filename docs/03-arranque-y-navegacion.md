# 03 · Arranque y navegación

## 1. `main()` — Orden de arranque

`lib/main.dart:29-73`

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 1. Supabase central  ── BLOQUEA
  await AppSupabase.init();

  // 2. Addons (fuentes + catálogos por Git)  ── BLOQUEA
  await AddonManager.instance.init();

  // 3. Notificaciones (descargas + cast)  ── BLOQUEA
  await NotificationHelper.init();

  // 4. Aviso de nueva versión / parche  ── fire-and-forget
  unawaited(UpdateNotificationService.instance.checkAndNotify());

  // 5. Foreground task (descargas + Cast)
  FlutterForegroundTask.init(androidNotificationOptions: AndroidNotificationOptions(
    channelId: 'downloads_channel',
    autoRunOnBoot: false,
    allowWakeLock: true, allowWifiLock: true,
  ));

  // 6. Callback de la notificación → botón de Cast
  FlutterForegroundTask.addTaskDataCallback((data) {
    if (data is Map && data['cast_btn'] is String) {
      CastManager().handleNotificationButton(data['cast_btn'] as String);
    }
  });

  // 7. Inmersivo  ⚠️ sin await (fire-and-forget)
  SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersiveSticky);

  runApp(const MyApp());
}
```

### 1.1 Las 3 llamadas que bloquean el primer frame

| Paso | Qué hace | Tiempo típico |
|---|---|---|
| `AppSupabase.init()` | Si no está configurado, retorna en microsegundos. Si lo está, `Supabase.initialize()` (PKCE) | 0 ms sin config / ~200-600 ms con config |
| `AddonManager.instance.init()` | Lee `SharedPreferences` + **encadena 3 repos de Stremio** | 100-500 ms |
| `NotificationHelper.init()` | Crea canales de notificación | 10-50 ms |

`AddonManager.init()` (`addon_manager.dart:49-70`):

```dart
Future<void> init() async {
  if (_initialized) return;
  loading = true; notifyListeners();
  try {
    addons   = await _storage.loadAddons();
    packages = await _storage.loadPackages();
    await StremioAddonRepository.instance.init();
    await StremioCollectionRepository.instance.init();
    await StreamBadgeRepository.instance.init();
    _initialized = true; error = null;
  } catch (e, st) { error = e.toString(); }
  finally { loading = false; notifyListeners(); }
}
```

### 1.2 Degradación silenciosa

`AppSupabase.init()` **nunca lanza**. Falla en silencio y permite arrancar sin
backend (`supabase_client.dart:14-37`):

```dart
static Future<void> init() async {
  if (_initialized && _client != null) return;
  if (!isSupabaseConfigured) { _client = null; _initialized = false; return; }
  try {
    await Supabase.initialize(
      url: kSupabaseUrl.trim(), anonKey: kSupabaseAnonKey.trim(),
      authOptions: const FlutterAuthClientOptions(authFlowType: AuthFlowType.pkce),
    );
    _client = Supabase.instance.client; _initialized = true;
  } catch (_) { _client = null; _initialized = false; }
}
```

---

## 2. `MyApp` — el `MaterialApp` real

```dart
// main.dart:80-89
return MaterialApp(
  title: 'lolplustv',
  debugShowCheckedModeBanner: false,
  theme: ThemeData(
    brightness: Brightness.dark,
    useMaterial3: true,
    scaffoldBackgroundColor: Colors.black,
  ),
  home: const SplashScreen(),
);
```

**Implicaciones:**
- Sin `routes:`, sin `onGenerateRoute:`, sin `routerConfig:` → navegación 100 %
  imperativa (~99 llamadas a `Navigator.push*`)
- Tema **fijo dark**, inline. `buildAppTheme()` de `lib/app/app_theme.dart`
  **nunca se invoca**
- `ThemeMode` no aparece en ningún sitio del repo

---

## 3. `SplashScreen` — máquina de estados del arranque

`main.dart:93-481`. `StatefulWidget` con dos variables de estado:

```dart
String _screen = 'loading';   // loading | mode | disclaimer
String? _mode;                // mobile | tv
bool get _isTv => _mode == 'tv';

final FocusNode _mobileFocus = FocusNode(debugLabel: 'mode_mobile');
final FocusNode _tvFocus     = FocusNode(debugLabel: 'mode_tv');
final FocusNode _acceptFocus = FocusNode(debugLabel: 'disclaimer_accept');
final FocusNode _rejectFocus = FocusNode(debugLabel: 'disclaimer_reject');
```

### 3.1 Flujo documentado en el propio código (`main.dart:127-135`)

```
1) Modo (solo 1ª vez)
2) Disclaimer (solo 1ª vez)
3) Auth / Perfil (login-register o invitado + elegir perfil)
4) Algoritmo / Gustos (taste onboarding)
5) Addons onboarding
6) Home
```

### 3.2 `_bootstrap()` (`main.dart:137-153`)

```dart
Future<void> _bootstrap() async {
  await Future.delayed(const Duration(seconds: 2));   // ← splash artificial de 2 s
  final prefs = await SharedPreferences.getInstance();
  final savedMode = prefs.getString(kModeKey);        // 'app_mode'
  if (savedMode == null) {
    setState(() => _screen = 'mode');
    _focusAfterFrame(_mobileFocus);                   // ← ya arranca con foco de D-pad
    return;
  }
  _mode = savedMode;
  await _applyOrientation(savedMode);
  await _goDisclaimerOrNext();
}
```

### 3.3 Disclaimer (`main.dart:155-167`, `199-209`)

Si `disclaimer_accepted != true`, muestra la pantalla legal. **Rechazar cierra
la app** en Android:

```dart
onReject: () async {
  await prefs.setBool(kDisclaimerKey, false);
  if (mounted) { SystemNavigator.pop(); }             // main.dart:205-209
}
```

### 3.4 Orientación por modo (`main.dart:175-189`)

La **única** lógica de adaptación de plataforma:

```dart
Future<void> _applyOrientation(String mode) async {
  if (mode == 'tv') {
    await SystemChrome.setPreferredOrientations([
      DeviceOrientation.landscapeLeft, DeviceOrientation.landscapeRight,
    ]);
  } else {
    await SystemChrome.setPreferredOrientations([
      DeviceOrientation.portraitUp, DeviceOrientation.portraitDown,
      DeviceOrientation.landscapeLeft, DeviceOrientation.landscapeRight,
    ]);
  }
}
```

### 3.5 Auth / Perfil (`main.dart:220-257`)

```dart
var profileId = await SupabaseConfig.getCurrentProfileId();
var isGuest  = await SupabaseConfig.isGuest();
final hasActiveProfile = profileId != null || isGuest;

if (!hasActiveProfile) {
  final isTv = _mode == 'tv';
  await Navigator.of(context).push(PageRouteBuilder(
    pageBuilder: (_, __, ___) => isTv
        ? const TvProfileSelectionPage(allowDismiss: false)
        : const ProfileSelectionPage(allowDismiss: false),
    transitionDuration: const Duration(milliseconds: 350),
    transitionsBuilder: (_, a, __, child) => FadeTransition(opacity: a, child: child),
  ));
  profileId = await SupabaseConfig.getCurrentProfileId();
  isGuest   = await SupabaseConfig.isGuest();
  if (profileId == null && !isGuest) {
    if (mounted) await _goAuthAndOnboarding();   // ← RECURSIÓN
    return;
  }
}
```

⚠️ `_goAuthAndOnboarding()` se llama recursivamente si el usuario no elige nada.

### 3.6 Onboarding de primer arranque (`main.dart:259-317`)

```dart
final tasteDone  = prefs.getBool(kTasteOnboardingKey) ?? false;   // 'taste_onboarding_done_v1'
final addonsDone = !(await AddonsOnboardingPage.shouldShow());    // 'addons_onboarding_done_v1'
final isFirstTime = !tasteDone || !addonsDone;
if (!isFirstTime) { await _openHome(); return; }
// → TasteOnboardingPage / TasteOnboardingPageTv
// → AddonsOnboardingPage
```

### 3.7 `_openHome()` — el punto de decisión móvil/TV (`main.dart:319-335`)

```dart
final Widget home = mode == 'tv' ? const tv.MainHome() : const mobile.MainHome();

Navigator.of(context).pushAndRemoveUntil(
  PageRouteBuilder(
    pageBuilder: (_, __, ___) => home,
    transitionDuration: const Duration(milliseconds: 350),
    transitionsBuilder: (_, a, __, child) => FadeTransition(opacity: a, child: child),
  ),
  (_) => false,       // ← limpia TODA la pila de navegación
);
```

---

## 4. Selector de modo

`main.dart:363-415` (`_buildModeSelector`) y `_ModeButton` (485-599).

El `_ModeButton` soporta D-pad completo desde el primer frame:

```dart
onKeyEvent: (node, event) {
  if (key == LogicalKeyboardKey.select || key == LogicalKeyboardKey.enter) onTap();
  if (key == arrowDown) → siguiente botón;
  if (key == arrowUp)   → botón anterior;
}
```

⚠️ **No hay detección automática** de plataforma ni de tamaño de pantalla. La
decisión es 100 % del usuario y se persiste en `app_mode`.

---

## 5. `mobile_shell.dart` (2023 líneas)

### 5.1 Las 5 secciones

```dart
// mobile_shell.dart:43-49
/// ÍNDICES FIJOS:
///   0 → Home
///   1 → Biblioteca (Guardados + Historial + Descargas)
///   2 → Servicios
///   3 → Config
///   4 → Buscar

static const int _kHome       = 0;
static const int _kBiblioteca = 1;
static const int _kServicios  = 2;
static const int _kConfig     = 3;
static const int _kBuscar     = 4;
```

| # | Sección | Widget | Origen |
|---|---|---|---|
| 0 | Home | `HomePage` / `HomeTvChanel` | `features/home/presentation/home_page.dart:29` |
| 1 | Biblioteca | `BibliotecaUnificadaPage` | `features/favorites/presentation/favorites_page.dart:40` |
| 2 | Servicios | `ServiciosPage` | `features/discover/domain/mobile/pag.dart:15` ⚠️ UI dentro de `domain/` |
| 3 | Config | `ConfigPage` | `features/settings/presentation/settings_page.dart:29` |
| 4 | Buscar | `BuscarPage` | `features/search/presentation/search_page.dart:30` |

```dart
// mobile_shell.dart:947-969
Widget _buildPage(int index) {
  switch (index) {
    case _kHome: return _homeLoaded
      ? (_tvLiveMode ? const HomeTvChanel(key: ValueKey('tvLiveHome'))
                     : HomePage(key: _homeKey))
      : const SizedBox.shrink();
    case _kBiblioteca:   return _bibliotecaLoaded ? BibliotecaUnificadaPage(...) : ...;
    case _kServicios:    return _serviciosLoaded  ? ServiciosPage(...) : ...;
    case _kConfig:       return const ConfigPage(key: ValueKey('configPage'));
    case _kBuscar:       return const BuscarPage(key: ValueKey('buscarPage'));
    default:             return const SizedBox.shrink();
  }
}
```

**Lazy-load por pestaña**: `_homeLoaded`, `_bibliotecaLoaded`, `_serviciosLoaded`
(`:74-76`, `:947-969`). Se conservan con flags para no recargar al volver.

### 5.2 Barra inferior

`_buildBottomNav` (`:1208-1326`): 4 iconos en cápsula + 1 círculo de búsqueda.

```dart
_NavIcon(icon: Icons.home_rounded,      selected: _currentIndex == _kHome, ...),
_NavIcon(icon: Icons.bookmark_rounded, selected: _currentIndex == _kBiblioteca, ...),
_NavIcon(icon: Icons.grid_view_rounded, selected: _currentIndex == _kServicios, ...),
_ProfileNavIcon(avatarUrl: _profileAvatar, name: _profileName,
                selected: _currentIndex == _kConfig, onTap: () => _selectTab(_kConfig)),
```

El cuarto "tab" es el **avatar del perfil** (`_ProfileNavIcon`, `:1967-2023`) que
hace de acceso a Configuración.

**Colapso según scroll** (`:163-182`): `pixels > 40` ⇒ colapsa (solo ícono
activo + mini-player + buscar); `pixels <= 10` ⇒ expande. La tarjeta
"Continuar viendo" solo se muestra si `!_navCollapsed` (`:1480`).

### 5.3 Orientación forzada en runtime (`:815-846`)

```dart
bool get _isCurrentRoute {
  final route = ModalRoute.of(context);
  return route == null || route.isCurrent;
}
void _forcePortraitIfCurrent() {
  if (!_isCurrentRoute) return;
  SystemChrome.setPreferredOrientations([
    DeviceOrientation.portraitUp, DeviceOrientation.portraitDown,
  ]);
  SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge);
}
@override void didChangeDependencies()        { ...; _forcePortraitIfCurrent(); }
@override void didChangeAppLifecycleState(s)  { if (s == resumed) ...; }
@override void didChangeMetrics()             { _forcePortraitIfCurrent(); }
```

### 5.4 Post-arranque (`initState`, `:102-119`)

```dart
DisplayRefresh.requestHighest();                     // MethodChannel 'app/display_refresh'
WidgetsBinding.instance.addObserver(this);
DownloadNavBus.version.addListener(_onDownloadNavBus);
MiniPlayerService.instance.addListener(_onMiniPlayer);
MiniPlayerService.instance.loadPref();
_loadContinueItem();                                 // escanea cachePlayer_* en prefs
WidgetsBinding.instance.addPostFrameCallback((_) {
  _loadProfileSession();
  _bootstrapOfflineOrSetup();    // offline → salta a Biblioteca; online → modal playback
  _checkForUpdate();             // banner APK
  SessionWatchTimer.instance.start(context);
});
```

### 5.5 Modo offline (`:623-646`)

```dart
Future<void> _bootstrapOfflineOrSetup() async {
  final online = await _hasConnection();   // InternetAddress.lookup('example.com'), timeout 3 s
  if (!online) { _selectTab(_kBiblioteca); return; }
  await _maybeShowPlaybackSetup();
}
```

---

## 6. `tv_shell.dart` (2600 líneas)

### 6.1 Las 9 secciones

```dart
// tv_shell.dart:97-103
static const int _kPerfilIndex    = 6;
static const int _kFuentesIndex   = 2;
static const int _kDescargasIndex = 7;
static const int _kTvIndex        = 8;
static const double _railWidth = 52;
static const double _topBarH   = 56;
String _menuPosition = 'side';     // 'side' | 'top'  (preferido por el usuario)
```

| idx | Sección | Widget | `_contentNodeOf` |
|---|---|---|---|
| 0 | Inicio | `HomePage` (TV) | `:590` |
| 1 | Descubrir | `EscrubirPage` | `:592` |
| 2 | Fuentes / Catálogos | `ServiciosTvPage` | `:594` |
| 3 | Buscar | `BuscarPage` (TV) | `:596` (vía `getAKeyFocusNode()`) |
| 4 | Biblioteca | `GuardadosPage` | `:602` |
| 5 | Configuración | `ConfigPage` (TV) | `:604` |
| 6 | Perfil | `PerfilPage` | `:606` |
| 7 | Descargas | `DescargasPageTv` | `:608` |
| 8 | TV (en vivo) | `HomeTvChanelTv` | `:610` |

### 6.2 Layout

```dart
// tv_shell.dart:1318-1362
child: Padding(
  padding: EdgeInsets.only(
    left: _menuPosition == 'top' ? 0 : _railWidth,
    top:  _menuPosition == 'top' ? _topBarH : 0,
  ),
  child: IndexedStack(index: _currentIndex, children: [ _buildPage(0), ... ]),
),
```

Dos menús seleccionables (`MenuPositionPref`):
- **`_SideMenu`** (`:2065-2272`): rail colapsable 42 px ↔ 140 px con
  `BackdropFilter` + reloj al pie
- **`_TopMenu`** (`:1757-1896`): barra de 56 px con logo + 5 tabs de texto +
  3 iconos + avatar

### 6.3 La máquina de foco D-pad ⭐

Cada sección expone su `FocusNode` raíz al shell vía callback, y el shell
gestiona la transferencia menú↔contenido (`_focusContentOf`, `:650-703`):

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
    if (attempt < 12) {                                 // 12 reintentos × 40 ms = 480 ms
      Future.delayed(const Duration(milliseconds: 40), () => tryFocus(attempt + 1));
    } else {
      _transferringToContent = false;
      _menuNodeOf(index).requestFocus();                // ← fallback: vuelve al menú
    }
  }
  WidgetsBinding.instance.addPostFrameCallback((_) => tryFocus());
}
```

El handler central de teclado (`_onMenuKey`, `:758-847`):

```dart
if (_menuPosition == 'top') {
  final topOrder = [home, descubrir, fuentes, search, guardados, tv, descargas, settings, profile];
  if (key == arrowDown)      { _focusContentOf(_currentIndex); }
  if (key == arrowLeft)      { topOrder[i-1].requestFocus(); }
  if (key == arrowRight)     { topOrder[i+1].requestFocus(); }
  if (key == select || enter){ _enterPage(tabIndex); }
} else {
  if (key == arrowDown)      { _menuFocusNodes[i+1].requestFocus(); }
  if (key == arrowUp)        { _menuFocusNodes[i-1].requestFocus(); }
  if (key == arrowRight || select || enter) { _enterPage(tabIndex); }
}
if (key == escape || goBack || backspace) { _confirmExit(); }
```

### 6.4 Otras características del shell TV

| Característica | Detalle |
|---|---|
| Reloj en pantalla | `Timer.periodic(30 s)` (`:156-159`), `_updateTime` (`:509`) |
| Colapso del menú | `Focus` listener con debounce de 60 ms (`:549-573`) |
| Salida | `PopScope` → diálogo de confirmación directo (`:1305-1309`) |
| Mini-player | ❌ No existe en TV |
| "Continuar viendo" | ❌ No existe en TV |
| Banner de actualización | ✅ Con `_AbiPickerDialog` (`:1423`) |
| Detección de ABI | `_detectDeviceAbi()` (`:190-203`) — **código duplicado literal** del móvil |

---

## 7. Comparativa de los dos shells

| Aspecto | `mobile_shell.dart` (2023) | `tv_shell.dart` (2600) |
|---|---|---|
| Secciones | 5 | 9 |
| Navegación | `IndexedStack` + `_selectTab(int)` | `IndexedStack` + `_selectTab(int, {focusContent})` |
| Menú | Barra inferior con blur | Rail lateral **o** barra superior |
| Interacción | `GestureDetector` / `onTap` | **D-pad**: 9 `FocusNode` de menú + 1 por sección |
| Orientación | Forzada vertical en runtime | Asume landscape |
| `WidgetsBindingObserver` | ✅ Sí | ❌ No |
| Lazy-load | 3 flags | 6 flags + `_ensureLoaded()` |
| Reloj | No | Sí |
| Mini-player | Sí | No |
| Continuar viendo | Sí (escanea prefs) | No |
| Buscador | Sección propia | Sección propia con teclado virtual |

> ⚠️ **~200 líneas duplicadas literalmente** entre ambos shells:
> `_detectDeviceAbi`, `_checkForUpdate`, `_downloadAndInstallApk`,
> `_urlForAbi`, `_labelAbi`, `_buildUpdateBanner`.

---

## 8. Cambio de modo en runtime

Se puede cambiar en ambos sentidos desde Ajustes:

| Dirección | Ubicación | Código |
|---|---|---|
| Móvil → TV | `settings_page.dart:610` `_switchToTv` | `prefs.setString('app_mode','tv')` + landscape + `pushAndRemoveUntil(tv.MainHome)` |
| TV → Móvil | `appearance/tv_appearance_tab.dart:73` `_switchToMobile` | Inverso |

La preferencia también se relee en `profile_selection_page.dart:180` para
re-enrutar tras cambiar de perfil.

---

## 9. Navegación y deep links

### 9.1 Sin router

99 llamadas imperativas. Patrones usados:

```dart
Navigator.push(context, MaterialPageRoute(builder: (_) => const PageX()));
Navigator.pushReplacement(context, PageRouteBuilder(...));   // player
Navigator.pushAndRemoveUntil(context, route, (_) => false); // cambio de shell
showModalBottomSheet(...);  // opciones de contenido
showDialog(...);           //Quality selector, PIN, confirmación
```

### 9.2 Deep links muertos

`MainActivity.kt:168-213` (`handleIntent`) procesa:

- `lol://user/<id>`
- `lol://content/<id>`
- `lol://list/<filter>`
- URL arbitraria

y luego `invokeMethod("openDeepLink", ...)`. **No hay ningún
`setMethodCallHandler` con ese nombre en Dart**, así que el link se recibe y se
descarta.

---

## 10. MethodChannels (Kotlin ↔ Dart)

`MainActivity.kt:20-69` registra 3 canales:

| Canal | Dirección | Propósito |
|---|---|---|
| `com.example.lol/intent` | Kotlin → Dart | Deep links (`openDeepLink`). ⚠️ **Sin receptor Dart** |
| `tv_webview/click` | Dart → Kotlin | `injectClick(x, y)`: busca el `WebView` en el árbol de vistas y **sintetiza `MotionEvent` DOWN/UP** (`:129-162`). Necesario porque el D-pad de la TV no llegaba al WebView embebido |
| `app/display_refresh` | Dart → Kotlin | `requestHighestRefreshRate` / `getRefreshRate` |

`requestHighestRefreshRate()` (`:75-110`): selecciona el modo de display con
mayor Hz (`maxByOrNull { refreshRate }`) y en Android R+ aplica
`SurfaceControl.Transaction().setFrameRate()`. Se llama **antes** de que arranque
Flutter (`MainActivity.kt:26-31`).

⚠️ `findWebView()` declara `view: View?` (`:131`) y usa `view.childCount`
(`:159`). Compila por *smart cast* de `val`, pero es frágil.

---

## 11. Diagrama del arranque

```
┌─────────────────────────────────────────────────────────────┐
│ main()                                                      │
│   ensureInitialized()                                       │
│   await AppSupabase.init()            ───┐                  │
│   await AddonManager.instance.init()  ───┤ 3 awaits         │
│   await NotificationHelper.init()     ───┘ bloqueantes      │
│   unawaited(UpdateNotificationService.checkAndNotify())      │
│   FlutterForegroundTask.init(...)                            │
│   SystemChrome.setEnabledSystemUIMode(immersiveSticky)      │
│   runApp(MyApp) ──► MaterialApp(home: SplashScreen)          │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
                 SplashScreen(_bootstrap)
                   delay(2s) → leer 'app_mode'
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
       _screen='mode'            _applyOrientation()
     (solo 1ª vez)                       │
     Mobile / TV (D-pad)        ┌─────────┴─────────┐
                                 ▼                   ▼
                        disclaimer_accepted?     ¿perfil o invitado?
                           │                         │
                    ┌──────┴──────┐          ┌───────┴───────┐
                    ▼             ▼          ▼               ▼
                 Aceptar      Rechazar   ProfileSelection  (continuar)
                 (focus)    SystemNav.pop  (allowDismiss:false)
                                                       │
                                    ┌──────────────────┤
                                    ▼                  ▼
                              primer arranque?     ¿rechazó?
                                    │              recursion
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
           TasteOnboarding              AddonsOnboarding
                    └───────────────┬───────────────┘
                                    ▼
                              _openHome()
                   pushAndRemoveUntil(mobile|tv MainHome, (_) => false)
```