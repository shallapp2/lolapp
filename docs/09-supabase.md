# 09 · Supabase (cuentas, perfiles y sincronización)

Backend opcional para cuentas de usuario, perfiles con PIN, y sincronización de
guardados/historial/likes entre dispositivos (móvil ↔ TV).

> ⚠️ **Estado actual: las credenciales están vacías en el repo.** La app arranca
> **en modo 100 % local (invitado)** hasta que se rellenen.
> Ver [§3](#3-configuración) y [§12](#12-el-sql-que-falta).

---

## 1. Inventario

| Fichero | Líneas | Responsabilidad |
|---|---|---|
| `supabase_constants.dart` | 15 | URL + anon key (**ambos vacíos**) + `isSupabaseConfigured` |
| `supabase_client.dart` | 49 | `AppSupabase.init()`, `currentUser`, `isLoggedIn`, `signOut()` |
| `supabase_config.dart` | 153 | Sesión de perfil + modo invitado + caché local de perfiles |
| `supabase_auth.dart` | 137 | signUp, signIn, signOut, resetPassword, updatePassword |
| `supabase_profiles.dart` | 186 | CRUD de perfiles (máx 5), selección con PIN, settings JSON |
| `supabase_data.dart` | 391 | Guardados / Historial / Likes con sincronización Supabase ↔ local |
| `guardados_service.dart` | 70 | Fachada de compatibilidad sobre `SupabaseData` |
| `supabase_tv_link.dart` | 126 | Emparejamiento TV↔móvil con código de 6 dígitos |

---

## 2. Flujo de arranque

```
main.dart:33  await AppSupabase.init()
  │
  ├─ isSupabaseConfigured == false  →  _client = null; return   (todo local)
  └─ Supabase.initialize(url, anonKey, authFlowType: AuthFlowType.pkce)
       ├─ OK  → _client = Supabase.instance.client; _initialized = true
       └─ ERR → _client = null; _initialized = false            (nunca lanza)
```

### 2.1 `AppSupabase.init()` (`supabase_client.dart:14-37`)

```dart
static Future<void> init() async {
  if (_initialized && _client != null) return;
  if (!isSupabaseConfigured) { _client = null; _initialized = false; return; }
  try {
    await Supabase.initialize(
      url: kSupabaseUrl.trim(),
      anonKey: kSupabaseAnonKey.trim(),
      authOptions: const FlutterAuthClientOptions(authFlowType: AuthFlowType.pkce),
    );
    _client = Supabase.instance.client;
    _initialized = true;
  } catch (_) { _client = null; _initialized = false; }
}
```

**PKCE** como flujo de auth. **Nunca lanza**: cualquier fallo degrada a
"sin Supabase".

---

## 3. Configuración

### 3.1 `supabase_constants.dart`

```dart
const String kSupabaseUrl     = '';   // → https://TU_PROYECTO.supabase.co
const String kSupabaseAnonKey = '';   // → eyJ...

bool get isSupabaseConfigured {
  final url = kSupabaseUrl.trim();
  final key = kSupabaseAnonKey.trim();
  return !url.contains('TU_PROYECTO') && !key.contains('TU_ANON_KEY')
      && url.isNotEmpty && key.isNotEmpty && url.startsWith('https://');
}
```

Con estos valores, `isSupabaseConfigured == false` → **toda la capa Supabase queda
inactiva**.

### 3.2 Checklist de puesta en marcha

- [ ] Crear proyecto en [supabase.com](https://supabase.com)
- [ ] Pegar el SQL del [§12](#12-el-sql-que-falta) en el SQL Editor
- [ ] Rellenar `kSupabaseUrl` y `kSupabaseAnonKey` en `supabase_constants.dart`
- [ ] En Supabase → Authentication → Providers → Email → **desactivar
      "Confirm email"** (si no, el login siempre pedirá confirmar)
- [ ] `flutter run`

---

## 4. Esquema de base de datos

### 4.1 `auth.users` (gestionado por Supabase)

| Campo | Tipo |
|---|---|
| `id` | UUID (PK) |
| `email` | text |
| `password` | (manejado por Auth) |
| `created_at` | timestamptz |

### 4.2 `profiles`

Columnas **confirmadas por el SELECT explícito** (`supabase_profiles.dart:21, 60`):

```dart
.select('id, name, avatar_url, backdrop_url, pin, settings, created_at, updated_at')
.eq('account_id', AppSupabase.currentUser!.id)
```

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID (PK) | |
| `account_id` | UUID → `auth.users.id` | usado en `WHERE`/`INSERT` (`:22, 49`) |
| `name` | text | |
| `avatar_url` | text | |
| `backdrop_url` | text | |
| `pin` | text nullable | PIN de seguridad; `''` ⇒ `null` (`:88`) |
| `settings` | jsonb | `settings ?? {}` (`:54`) |
| `created_at` / `updated_at` | timestamptz | `ORDER BY created_at` (`:23`) |

### 4.3 `historial`

SELECT en `supabase_data.dart:130`:

```dart
.select('id, tmdb_id, poster, tipo, progress_seconds, season, episode, '
        'titulo, backdrop, duration_seconds, created_at, updated_at')
.eq('profile_id', profileId).order('updated_at', ascending: false)
```

**Clave de conflicto UNIQUE**: `'profile_id,tmdb_id,tipo,season,episode'` (`:217`)

> ⚠️ **Truco del centinela 0** (`:162-167`):
>
> ```dart
> // PostgreSQL UNIQUE trata NULL como distinto → películas no harían upsert.
> // Usamos 0 como sentinela para movie (y TV sin s/e).
> final seasonVal  = tipoNorm == 'tv' ? (season  ?? 0) : 0;
> final episodeVal = tipoNorm == 'tv' ? (episode ?? 0) : 0;
> ```

### 4.4 `guardados` y `likes`

| Tabla | Columnas | UNIQUE |
|---|---|---|
| `guardados` | `id, profile_id, tmdb_id, poster, tipo, created_at` (`:30`) | `'profile_id,tmdb_id,tipo'` (`:73`) |
| `likes` | `id, profile_id, tmdb_id, poster, tipo, created_at` (`:283`) | `'profile_id,tmdb_id,tipo'` (`:326`) |

Orden: `created_at desc`.

### 4.5 `tv_link`

| Campo | Uso |
|---|---|
| `code` | código de 6 dígitos (clave lógica) |
| `used` | bool |
| `expires_at` | expiración 15 min |
| `account_id` | cuenta emparejada |
| `id` | PK |

### 4.6 RLS y triggers

Según `lib/docs/INSTRUCCIONES.md`:

| Línea | Afirmación |
|---|---|
| `:164` | *"RLS protege que un usuario solo vea sus propios datos."* |
| `:163` | *"El límite de 5 perfiles se aplica con un trigger en la base de datos."* |
| `:165` | *"El código de TV expira en 15 minutos."* |

El cliente **solo filtra por `account_id`/`profile_id` en el WHERE** — no hay
policies escritas en Dart, y **ninguna policy será efectiva sin el SQL**.

---

## 5. `SupabaseAuth` (137 líneas)

```dart
AuthResult { AuthResponse? response; String? error; bool needsEmailConfirm; }
```

Getters: `ok` (`:13`), `hasSession` (`:14`).

### 5.1 `signUp({email, password})` (`:22-59`)

```dart
await _c!.auth.signUp(email.trim(), password);
if (user != null && session == null) {
  // → needsEmailConfirm: true
  error = 'Debes confirmar el email. En Supabase desactiva "Confirm email" '
         'para evitar esto.';
}
// Éxito → SupabaseConfig.clearGuestMode()
```

### 5.2 `signIn({email, password})` (`:62-97`)

`auth.signInWithPassword`. `AuthException` con `"email not confirmed"` /
`"not confirmed"` → mensaje accionable:

> *"Desactiva 'Confirm email' en Supabase → Authentication → Providers → Email."*

Éxito → `clearGuestMode()`.

### 5.3 `signOut()` (`:100-105`) — cascada completa

```dart
await AppSupabase.signOut();
await SupabaseConfig.clearCurrentProfile();
await SupabaseConfig.clearGuestMode();
await SupabaseConfig.clearLocalProfiles();
```

Otros: `resetPassword(email)` (`:107-116`), `updatePassword(newPassword)`
(`:118-126`), `currentEmail` (`:128`), `accountCreatedAt` (`:130-134`),
`isLoggedIn` (`:136`).

---

## 6. `SupabaseConfig` (153 líneas) — sesión local

Claves `SharedPreferences` (`:9-14`):

```dart
'current_profile_id'
'current_profile_name'
'current_profile_avatar'
'current_profile_backdrop'
'is_guest_mode'
'local_profiles_cache'
```

| Método | Líneas | Detalle |
|---|---|---|
| `setCurrentProfile({id, name, avatarUrl, backdropUrl})` | 38-58 | Al fijar perfil **fuerza `is_guest_mode = false`** (`:57`) |
| `setGuestMode()` | 75-82 | Pone `is_guest_mode = true` y **borra** las 4 claves de perfil |
| `getLocalProfiles` / `saveLocalProfiles` / `addLocalProfile` / `removeLocalProfiles` / `clearLocalProfiles` | 91-124 | Caché de perfiles sin conexión |

Alias de compatibilidad: `isSupabaseActive()` (`:129-133`) =
`profileId != null || guest`; `getCurrentUserName()`, `getCurrentUserId()`,
`isLoggedIn()`, `hasCredentials() => true` (`:148`),
`getAskProfileEveryLaunch() => false` (`:150`).

---

## 7. `SupabaseProfiles` (186 líneas) — perfiles (máx 5)

| Método | Líneas | Detalle |
|---|---|---|
| `list()` | 12-34 | `SELECT ... WHERE account_id = uid ORDER BY created_at`. **Cachea** en `local_profiles_cache`. Si no hay sesión → devuelve la caché local |
| `create({name, avatarUrl, backdropUrl, pin, settings})` | 37-69 | INSERT con `account_id: uid`, `settings: settings ?? {}`. Cachea. **Devuelve `null` silenciosamente** si no hay sesión o si el trigger de 5 perfiles lanza (`catch (_) { return null; }` `:66-68`) |
| `update({...})` | 72-121 | UPDATE; actualiza caché; si es el perfil actual refresca la sesión. `pin: '' → null` (`:88`) |
| `delete(id)` | 124-140 | DELETE; quita de caché; si era el actual, `clearCurrentProfile()` |
| `select(profile, {pinInput})` | 143-163 | **Verificación de PIN**: `if (profile['pin'] != '' && pinInput != storedPin) return false;` Si pasa → `setCurrentProfile(...)` |
| `getSettings()` / `saveSettings(map)` | 166-185 | Lee/escribe el blob JSON `settings` del perfil actual |

> El límite de 5 **no está en Dart**: se aplica en UI
> (`manage_profiles_page.dart:134`, `profile_selection_page.dart:336`) y
> realmente en el **trigger de BD**. El mensaje de error de TV lo refleja:
> *"Error al guardar. ¿Máximo 5 perfiles o sin sesión?"*
> (`tv_edit_profile_page.dart:210`)

---

## 8. `SupabaseData` (391 líneas) — Guardados / Historial / Likes

**Patrón común de las 3 secciones**:

> Si `isGuest || profileId == null || !SupabaseAuth.isLoggedIn` → **solo
> `SharedPreferences`**.
> Si no → Supabase, **con caché local de respaldo en cada `catch`**.

Helpers de caché (`:372-390`): `_getLocalList(key)` / `_saveLocalList(key, list)`
sobre JSON en prefs.

### 8.1 Guardados

| Método | Líneas | Detalle |
|---|---|---|
| `getGuardados()` | 19-42 | `SELECT id, tmdb_id, poster, tipo, created_at WHERE profile_id ORDER BY created_at DESC`. Guarda en `local_guardados` |
| `addGuardado({tmdbId, poster, tipo})` | 44-78 | `upsert(..., onConflict: 'profile_id,tmdb_id,tipo')` |
| `removeGuardado` | 80-105 | delete por las 3 columnas |
| `isGuardado` | 107-113 | `any(...)` sobre la lista |

### 8.2 Historial

| Método | Líneas | Detalle |
|---|---|---|
| `getHistorial()` | 119-142 | SELECT de 13 columnas, `ORDER BY updated_at DESC`, cache `local_historial` |
| `saveHistorial({...})` | 148-223 | `upsert(..., onConflict: 'profile_id,tmdb_id,tipo,season,episode')` con el **centinela 0**. En local mantiene solo los **últimos 100** (`:194-195`) |
| `removeHistorial` | 225-266 | delete por las 5 columnas |

### 8.3 Likes

Espejo de guardados con tabla `likes` (`:272-366`), caché `local_likes`.

### 8.4 Consumidores

| Llamada | Origen |
|---|---|
| `SupabaseData.saveHistorial` | `player_page.dart:401`, `tv_player_page.dart:731` |
| `getHistorial` | `favorites_page.dart:143`, `history_page.dart:246`, `player_page.dart:446`, `tv_player_page.dart:766` |

---

## 9. `GuardadosService` (70 líneas) — fachada legacy

Mapea al formato antiguo con clave `idcontenido` (`:11-20`).

```dart
Future<bool> toggle(item) async {
  // normaliza tipo (tv|serie → tv, resto → movie)
  // consulta isGuardado, y hace add o remove devolviendo si quedó guardado
}
Future<void> remove(idcontenido) async;   // borra en AMBOS tipos
Future<void> clearAll() async;            // itera y borra
```

Consumido por `content_page.dart:41-49` y `tv_content_page.dart:41-51`.

---

## 10. `SupabaseTvLink` (126 líneas) — vinculación TV↔móvil

**Flujo documentado** (`:6-11`):

```
1. TV genera código → createCode()
2. Móvil (logueado) introduce el código → linkCode(code)
3. TV hace polling → pollCode(code) hasta que recibe account_id
```

### 10.1 Generación del código (`:15-18`)

```dart
static String _generateCode() {
  final rnd = Random.secure();
  return List.generate(6, (_) => rnd.nextInt(10)).join();
}
```

### 10.2 Los 3 métodos

| Método | Líneas | Detalle |
|---|---|---|
| `createCode()` | 22-36 | INSERT `{'code': code, 'used': false}`. **Devuelve el código incluso si el INSERT falla** (`catch (_) { return code; }` `:33-35`) para poder mostrarlo |
| `linkCode(code)` | 39-72 | Exige sesión; valida longitud == 6; SELECT `id, used, expires_at`; rechaza si `used == true`; rechaza si expiró; luego UPDATE `{'account_id': uid, 'used': true}` |
| `pollCode(code)` | 75-99 | SELECT `account_id, used, expires_at`; devuelve `account_id` si `used == true && account_id != null`; `null` si expiró |

### 10.3 ⚠️ Limitación reconocida (`:107-125`)

```dart
// SIEMPRE devuelve []
// La TV no tiene sesión Auth, por lo que esta consulta solo funciona si
// la policy lo permite o si usamos service_role (no recomendado en cliente).
static Future<List<Map<String,dynamic>>> getProfilesForAccount(String accountId) async {
  return [];
}
```

### 10.4 UI — `features/settings/presentation/account/account_settings_page.dart`

| Versión | Líneas | Comportamiento |
|---|---|---|
| Móvil (`:145-168`) | Input de 6 dígitos (`maxLength:6`, `letterSpacing:8`) → `_linkTvCode()` (`:49-69`) |
| TV (`:171-218`) | Botón "Generar código" → `_generateTvCode()` (`:71-102`): tras `createCode()` hace **polling de 60 intentos × 3 s = 3 minutos**, mostrando el código en 42pt con `letterSpacing:12` y el texto "Esperando vinculación…" |

Cambio de contraseña con mínimo 6 caracteres (`:31-47`).

---

## 11. UI de cuentas y perfiles

### 11.1 Login / Registro

| Fichero | Líneas | Contenido |
|---|---|---|
| `features/auth/presentation/login_register_page.dart` | 565 | Fondo negro + `RadialGradient`, logo `assets/logo.png`, campos `_GlassField` con `BackdropFilter(blur 14)` (`:316-441`), `_GlassButton` con foco D-pad (`:446-565`). Validación: campos obligatorios, contraseñas coinciden, **mín. 6 caracteres** (`:58-69`). Toggle login/registro con un solo set de controllers |
| `features/auth/presentation/tv_login_register_page.dart` | 961 | Versión TV con `FocusTraversalGroup(OrderedTraversalPolicy)` (`:566`) |

### 11.2 Perfiles

| Fichero | Líneas | Rol |
|---|---|---|
| `profile_selection_page.dart` | 666 | **Selector estilo Netflix**. Grid de 2 columnas con última fila impar centrada (`:359-381`). Backdrop reactivo al hover/focus (`:298-316`), fallback a un wallpaper de Crunchyroll (`:39-40`). Auto-abrió editor si no hay perfiles; **auto-selecciona** si hay exactamente 1 y no tiene PIN (`:85-94`). Diálogo PIN de 6 dígitos (`:196-280`). Tarjetas "Añadir" (solo si `<5`, `:336`) e "Invitado" (solo si no hay perfiles, `:348-357`). Botones: "Editar perfiles", "Invitado", "Iniciar sesión / Crear cuenta", "Cerrar sesión" |
| `manage_profiles_page.dart` | 159 | Alta/edición/baja. Botón crear solo si `<5` (`:134`); mensaje *"Has alcanzado el máximo de 5 perfiles."* (`:146-154`); confirmación de borrado con aviso *"Se borrarán su historial, guardados y likes"* (`:60`) |
| `edit_profile_page.dart` | 290 | Editor móvil con avatares predefinidos DiceBear (`:30-35`) y backdrops TMDB `original` (`:38-44`) |
| `tv_edit_profile_page.dart` | 1560 | Editor TV completo; `_save()` (`:166-216`) llama `create`+`select` auto; `_delete()` (`:218-265`) |
| `tv_manage_profiles_page.dart` | 338 | Gestión TV |
| `tv_profile_selection_page.dart` | 1012 | Selector TV con `FocusNode` por perfil, sincronizados por `_syncProfileFocus(count)` (`:67-74`) |
| `tv_picker_page.dart` | 939 | Selector adicional TV |
| `tv_text_input_dialog.dart` | 276 | Diálogo de texto TV |
| `profile_page.dart` | 822 | Página de perfil; abre selector con `allowDismiss: true` (`:254`) |

### 11.3 `profile_page.dart` — componentes

| Componente | Líneas |
|---|---|
| `_WatchStats` | 376 |
| `_ProfileBanner` | 429 |
| `_StatCard` | 597 |
| `_DeviceCard` | 703 (con `device_info_plus` + `package_info_plus`) |

### 11.4 Ajustes

`features/settings/presentation/supabase/supabase_section.dart` (96 líneas):
3 tiles — Cuenta / Perfiles / (si logueado) Gestionar. Estado mostrado:

```
'Perfil: $name'  |  'Modo invitado'  |  'Sin sesión'
```

Equivalente TV: `tv_supabase_tab.dart` (169 líneas).

---

## 12. El SQL que falta

**No hay ningún archivo `.sql` en el repo.** Verificado:
`find . -name "*.sql"` → 0 resultados.

Sin embargo, `lib/docs/INSTRUCCIONES.md:20` instruye pegar manualmente
`sql/01_tablas_completas.sql` — ese archivo **no está en el repo**. Hay que
reconstruirlo a partir de lo que revela el código.

### 12.1 SQL mínimo funcional

```sql
-- ============================================================
-- 1) PERFILES (máx 5 por cuenta)
-- ============================================================
create table if not exists public.profiles (
  id           uuid primary key default gen_random_uuid(),
  account_id   uuid not null references auth.users(id) on delete cascade,
  name         text not null,
  avatar_url   text,
  backdrop_url text,
  pin          text,
  settings     jsonb not null default '{}'::jsonb,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);
create index if not exists profiles_account_idx on public.profiles(account_id);

-- Trigger: máximo 5 perfiles por cuenta
create or replace function public.enforce_max_5_profiles()
returns trigger language plpgsql security definer as $$
begin
  if (select count(*) from public.profiles where account_id = new.account_id) >= 5 then
    raise exception 'Máximo 5 perfiles por cuenta';
  end if;
  return new;
end $$;

create trigger trg_max_5_profiles
  before insert on public.profiles
  for each row execute function public.enforce_max_5_profiles();

-- Mantener updated_at
create or replace function public.touch_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end $$;

create trigger trg_profiles_touch before update on public.profiles
  for each row execute function public.touch_updated_at();

-- ============================================================
-- 2) HISTORIAL
-- ============================================================
create table if not exists public.historial (
  id               uuid primary key default gen_random_uuid(),
  profile_id       uuid not null references public.profiles(id) on delete cascade,
  tmdb_id          integer not null,
  poster           text,
  backdrop         text,
  titulo           text,
  tipo             text not null check (tipo in ('movie','tv')),
  progress_seconds integer not null default 0,
  season           integer not null default 0,   -- 0 = centinela para movies
  episode          integer not null default 0,   -- 0 = centinela para movies
  duration_seconds integer,
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  unique (profile_id, tmdb_id, tipo, season, episode)
);
create index if not exists historial_profile_idx on public.historial(profile_id, updated_at desc);

-- ============================================================
-- 3) GUARDADOS
-- ============================================================
create table if not exists public.guardados (
  id         uuid primary key default gen_random_uuid(),
  profile_id uuid not null references public.profiles(id) on delete cascade,
  tmdb_id    integer not null,
  poster     text,
  tipo       text not null check (tipo in ('movie','tv')),
  created_at timestamptz not null default now(),
  unique (profile_id, tmdb_id, tipo)
);

-- ============================================================
-- 4) LIKES
-- ============================================================
create table if not exists public.likes (
  id         uuid primary key default gen_random_uuid(),
  profile_id uuid not null references public.profiles(id) on delete cascade,
  tmdb_id    integer not null,
  poster     text,
  tipo       text not null check (tipo in ('movie','tv')),
  created_at timestamptz not null default now(),
  unique (profile_id, tmdb_id, tipo)
);

-- ============================================================
-- 5) TV LINK (código de 6 dígitos, expira 15 min)
-- ============================================================
create table if not exists public.tv_link (
  id         uuid primary key default gen_random_uuid(),
  code       text not null unique,
  used       boolean not null default false,
  account_id uuid references auth.users(id) on delete cascade,
  expires_at timestamptz not null default (now() + interval '15 minutes'),
  created_at timestamptz not null default now()
);

-- ============================================================
-- 6) RLS: cada usuario solo ve sus propios datos
-- ============================================================
alter table public.profiles  enable row level security;
alter table public.historial enable row level security;
alter table public.guardados enable row level security;
alter table public.likes     enable row level security;
alter table public.tv_link   enable row level security;

create policy "profiles: own account" on public.profiles
  for all using (account_id = auth.uid())
  with check (account_id = auth.uid());

create policy "profiles: read by account_id" on public.profiles
  for select using (account_id is not null);   -- para el polling de la TV

create policy "historial: own profile" on public.historial
  for all using (profile_id in (select id from public.profiles where account_id = auth.uid()))
  with check (profile_id in (select id from public.profiles where account_id = auth.uid()));

create policy "guardados: own profile" on public.guardados
  for all using (profile_id in (select id from public.profiles where account_id = auth.uid()))
  with check (profile_id in (select id from public.profiles where account_id = auth.uid()));

create policy "likes: own profile" on public.likes
  for all using (profile_id in (select id from public.profiles where account_id = auth.uid()))
  with check (profile_id in (select id from public.profiles where account_id = auth.uid()));

create policy "tv_link: read by code" on public.tv_link
  for select using (true);

create policy "tv_link: insert anonymous" on public.tv_link
  for insert with check (true);

create policy "tv_link: update linked" on public.tv_link
  for update using (auth.uid() is not null);
```

> ⚠️ **Ajusta las policies según tu modelo de seguridad.** Las de `tv_link`
> son abiertas por diseño (la TV no tiene sesión Auth), lo que implica que
> cualquiera que conozca un código de 6 dígitos podría vincularlo. Códigos de
> 6 dígitos tienen solo 10⁶ combinaciones: considera rate limiting o códigos más
> largos en producción.

---

## 13. Modo invitado

Si el usuario elige "Invitado" en la pantalla de perfiles:

- No necesita cuenta
- Historial, guardados y likes se guardan **solo en `SharedPreferences`**
  (claves `local_historial`, `local_guardados`, `local_likes`)
- Al crear cuenta después, **no se migran automáticamente**

---

## 14. Diagrama de decisión

```
                    ┌──────────────────────────┐
                    │ SplashScreen             │
                    └────────────┬─────────────┘
                                 ▼
                    SupabaseConfig.getCurrentProfileId()
                    SupabaseConfig.isGuest()
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
        profileId != null     isGuest==true      ambos null/false
              │                  │                  │
              ▼                  ▼                  ▼
      ir a Home          ir a Home        ProfileSelectionPage
      (sincroniza)      (solo local)     (allowDismiss:false)
                                                 │
                            ┌────────────────────┼────────────────────┐
                            ▼                    ▼                    ▼
                        Invitado          Login/Registro          Perfil existente
                            │                    │                    │
                            ▼                    ▼                    ▼
                     is_guest_mode=true    SupabaseAuth        PIN si lo tiene
                     (borra perfil)       .signIn/signUp       SupabaseProfiles
                                                                       .select()
                                                          ┌────────┴────────┐
                                                          ▼                 ▼
                                                 con sesión           sin sesión
                                              (sync Supabase)     (solo local)
```

---

## 15. Problemas

1. **El SQL no está en el repo** pese a que `INSTRUCCIONES.md` lo referencia.
   Sin él, la app arranca en modo invitado y nada más.
2. **Credenciales vacías** en `supabase_constants.dart`.
3. `getProfilesForAccount()` **siempre devuelve `[]`** (limitación conocida y
   documentada en el propio código).
4. `create()` devuelve `null` **en silencio** si el trigger de 5 perfiles
   revienta — el usuario solo ve un error genérico.
5. Elokuador de "Confirm email" de Supabase **rompe el registro/login** si
   está activado, y el error solo se explica en el mensaje.
6. **No hay migración de datos de invitado a cuenta**.
7. **Sin rate limiting** en el pairing por código de 6 dígitos.