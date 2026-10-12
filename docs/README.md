# lolplustv (`jostinsantos/lolapp`) — Documentación técnica

Documentación completa del proyecto **lolplustv** (paquete Flutter `lolbeta`),
fork de [jostinsantos/lolapp](https://github.com/jostinsantos/lolapp).

| | |
|---|---|
| **Nombre del paquete** | `lolbeta` |
| **Nombre visible** | `lolplustv` / `lol` |
| **Lenguaje** | Dart (Flutter 3.35.x stable, SDK `^3.13.4`) |
| **Plataforma real** | Android (móvil + Android TV, una sola APK) |
| **Ficheros Dart** | 325 (~153.000 líneas) en `lib/` |
| **Dependencias** | 45 declaradas (~22 realmente usadas) |
| **Tests** | 0 (`test/widget_test.dart` está vacío) |
| **Licencia** | Sin `LICENSE` en el repo |

> ⚠️ **Aviso legal**: este proyecto es un agregador de streams que hace scraping
> de sitios web de cine y series y resuelve URLs de reproducción. No aloja ni
> distribuye contenido. El uso es responsabilidad del usuario y debe respetar
> la legislación aplicable.

---

## Índice de la documentación

### Fundamentos
| # | Documento | Contenido |
|---|---|---|
| 00 | [Instalación y puesta en marcha](00-instalacion.md) | Requisitos, setup, build, primer arranque |
| 01 | [Arquitectura general](01-arquitectura.md) | Capas reales vs. declaradas, flujo de datos |
| 02 | [Estructura de carpetas](02-estructura.md) | Mapa completo de `lib/` con descripción |
| 03 | [Arranque y navegación](03-arranque-y-navegacion.md) | `main.dart`, splash, shells móvil/TV, D-pad |

### Núcleo de datos
| # | Documento | Contenido |
|---|---|---|
| 04 | [Scrapers y catálogo](04-scrapers.md) | Las 5 fuentes con scraping HTML |
| 05 | [Extractores y HLS](05-extractores.md) | Resolución de URL de reproducción (3 niveles) |
| 06 | [Fuentes y agregación](06-fuentes-y-agregadores.md) | `MainFuentes`, verificación, calificación |
| 07 | [Addons y Stremio](07-addons.md) | Sistema de addons JS (QuickJS) + protocolo Stremio |
| 08 | [TMDB](08-tmdb.md) | APIs, claves, filtros, caché |
| 09 | [Supabase](09-supabase.md) | Auth, perfiles, sync, schema BD |

### Features
| # | Documento | Contenido |
|---|---|---|
| 10 | [Reproductor](10-reproductor.md) | Playback, calidad, subtítulos, resume, cast |
| 11 | [TV, D-pad e IPTV](11-tv-iptv.md) | Android TV, TVChanel (M3U), zapping |
| 12 | [Inteligencia artificial](12-ia.md) | Lolbot, "Para ti", Top 10 regional, cuotas |
| 13 | [Features de UI](13-features-ui.md) | Todas las pantallas y flujos |

### Referencia
| # | Documento | Contenido |
|---|---|---|
| 14 | [Almacenamiento y caché](14-almacenamiento.md) | SharedPreferences, claves, TTLs, invalidación |
| 15 | [Configuración (preferencias)](15-configuracion.md) | Todas las preferencias y su efecto |
| 16 | [Build y despliegue Android](16-build-android.md) | Gradle, manifest, canales, firma, ABIs |
| 17 | [Problemas conocidos y deuda técnica](17-deuda-tecnica.md) | Todo lo que hay que arreglar |
| 18 | [Glosario y acrónimos](18-glosario.md) | TMDB, HLS, D-pad, PoW, etc. |

---

## Los 5 datos que hay que saber antes de tocar nada

1. **La arquitectura declarada no es la real.** `lib/core/`, `lib/domain/`,
   `lib/app/` y `lib/core/{result,errors,network,storage}` son *scaffolding*
   vacío (~4.800 líneas que nadie importa). El código real vive en
   `lib/data/**`, `lib/features/*/presentation/**`, `lib/presentation/**` y
   `lib/supabase/**`.
   → Ver [01-arquitectura.md](01-arquitectura.md)

2. **No hay inyección de dependencias.** Ni `provider` ni `get_it` se usan
   (aunque `provider` esté declarado). Todo son *singletons* estáticos
   (`static final X.instance`) + `ChangeNotifier` + `SharedPreferences`.

3. **No hay router.** `go_router` está declarado pero sin usar. Hay ~99 llamadas
   imperativas a `Navigator.push`.

4. **No hay base de datos local.** Todo se guarda en `SharedPreferences`
   (~70 ficheros con claves hardcodeadas). Supabase es el único backend real.

5. **El motor de vídeo es `video_player` (ExoPlayer).** `media_kit`, `chewie`,
   `fvp` y `libtorrent_flutter` están declaradas pero **no se usan** (~15-25 MB
   de APK de relleno).

---

## Mapa mental del proyecto en 30 segundos

```
Usuario abre la app
   └─> main.dart: Supabase → AddonManager → Notificaciones → runApp
        └─> SplashScreen
             ├─ Elegir modo (móvil | TV)          [solo 1ª vez]
             ├─ Disclaimer legal                  [solo 1ª vez]
             ├─ Auth / Perfil (o Invitado)
             ├─ Onboarding de gustos (IA)
             ├─ Onboarding de addons
             └─> MainHome (mobile_shell.dart | tv_shell.dart)

Sección Home
   └─> TmdbHomeMobileService / TmdbHomeService  (11 requests a TMDB)
   └─> Addons de catálogo Stremio  (secciones extra)
   └─> "Para ti" + Top 10 regional  (IA, cada 20-24 h)

Usuario abre una película
   └─> PageContenido (tmdb_content.dart + OMDb)
        └─> ServidoresModal
             └─> MainFuentes.fetchProgressive()   [13 fuentes nativas + addons]
                  └─> extraer servidores (embeds)
                       └─> verificarConExtractor()  [WebView 1x1 o resolver nativo]
                            └─> m3u8
   └─> PlayerScreen (video_player / ExoPlayer)
        ├─> HlsQualityParser (selector de calidad)
        ├─> OpenSubtitles (subtítulos)
        ├─> IntroDB / SkipDB (skip intro)
        ├─> Chromecast (dart_cast)
        └─> guardar progreso (prefs + Supabase, cada 30 s)
```