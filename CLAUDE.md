# CLAUDE.md — AI Assistant Guide for FongMi/TV

This file provides guidance for AI coding assistants (Claude, Copilot, etc.) working on this repository.

---

## Project Overview

**FongMi TV** is an open-source Android application for VOD (Video on Demand) and live TV streaming. It supports both Android TV (Leanback UI) and mobile devices. The app uses a plugin-based "Spider" system that loads content crawlers written in Java (JAR), JavaScript (QuickJS engine), or Python (Chaquopy engine).

---

## Repository Structure

```
TV/
├── app/                  # Main application module
│   ├── src/main/         # Shared business logic, common UI components
│   ├── src/leanback/     # Android TV (Leanback) UI flavor
│   └── src/mobile/       # Mobile UI flavor
├── catvod/               # Spider abstraction layer and OkHttp networking
├── quickjs/              # QuickJS JavaScript engine wrapper
├── chaquo/               # Chaquopy Python engine wrapper (Python 3.10)
├── tvbus/                # TVBus live streaming engine
├── thunder/              # Thunder streaming engine
├── zlive/                # Z-Live streaming engine (ARM v7a only)
├── forcetech/            # ForceTech streaming engine
├── hook/                 # Package manager override/hook utilities
├── jianpian/             # P2P streaming engine
├── docs/                 # Project documentation
│   ├── CONFIG.md         # Full VOD/Live config JSON schema
│   ├── SPIDER.md         # Spider API specification
│   └── LOCAL.md          # Local HTTP API for playback control
├── other/                # Miscellaneous tools
├── build.gradle          # Root Gradle config (shared dependency versions)
├── settings.gradle       # Multi-module configuration
└── gradle.properties     # JVM args, build caching, AndroidX settings
```

### Core App Package Layout (`com.fongmi.android.tv`)

```
api/
  config/       # VodConfig, LiveConfig, WallConfig, BaseConfig
  loader/       # BaseLoader, JarLoader, JsLoader, PyLoader
bean/           # ~50 data-model/entity classes
db/             # Room database (AppDatabase, Migrations, DAO interfaces)
event/          # EventBus event classes (ConfigEvent, PlayerEvent, etc.)
exception/      # Custom exceptions
gson/           # Custom Gson type adapters
impl/           # Callback implementations (Callback, DohCallback, NewPipeImpl…)
model/          # ViewModels
player/
  danmaku/      # Bullet-screen (danmaku) implementation
  exo/          # ExoPlayer/Media3 integration
  extractor/    # URL extraction logic
  Players.java  # Central player controller
  ParseJob.java # Parse job manager
receiver/       # Broadcast receivers
server/         # Local NanoHTTPD HTTP server
service/        # Android services
ui/
  activity/     # Activities
  adapter/      # RecyclerView adapters
  custom/       # Custom views
  dialog/       # Dialogs
utils/          # Utility helpers
App.java        # Application singleton (thread pools, global singletons)
Constant.java   # App-wide constants (timeouts, intervals)
Setting.java    # SharedPreferences wrapper
Startup.java    # Startup configuration
```

---

## Build System

- **Build tool:** Gradle 8.14.3 with Android Gradle Plugin 8.13.2
- **Compile SDK:** 36 — **Min SDK:** 24 — **Target SDK:** 28 (app), 36 (libraries)
- **Java:** Java 17 (core library desugaring enabled)
- **NDK ABI filters:** `arm64-v8a`, `armeabi-v7a`

### Key Shared Dependency Versions (defined in root `build.gradle`)

| Dependency | Version |
|------------|---------|
| Gson | 2.13.2 |
| Glide | 5.0.5 |
| Media3 (ExoPlayer) | 1.9.2 |
| OkHttp | 5.3.2 |
| Room | 2.8.4 |

### Build Variants / Flavors

The app has two product flavors:
- **leanback** — Android TV UI using the Leanback library
- **mobile** — Standard phone/tablet UI

Build command examples:
```bash
# Assemble debug APK for leanback flavor
./gradlew assembleLeanbackDebug

# Assemble release APK for mobile flavor
./gradlew assembleMobileRelease

# Build all variants
./gradlew assemble
```

### Gradle Properties Highlights

- JVM max heap: `-Xmx4g`
- Parallel builds and caching enabled
- AndroidX and Jetifier enabled

---

## Technology Stack

| Category | Library / Framework |
|----------|-------------------|
| Video playback | AndroidX Media3 / ExoPlayer 1.9.2 |
| Networking | OkHttp 5.3.2, DoH support |
| JSON | Gson 2.13.2 |
| Image loading | Glide 5.0.5 |
| Local DB | Room 2.8.4 |
| Event bus | EventBus (Greenrobot) 3.3.1 |
| TV UI | Leanback 1.2.0 |
| Local HTTP server | NanoHTTPD 2.3.1 |
| DLNA/UPnP | Cling 2.1.1 |
| QR scanning | ZXing 3.5.4 |
| Animations | Lottie 6.7.1 |
| SMB | SMBJ 0.14.0 |
| JavaScript engine | QuickJS (wrapper-java 3.2.3) |
| Python engine | Chaquopy 17.0.0 (Python 3.10) |
| YouTube | NewPipe Extractor v0.26.0 |

---

## Key Architectural Patterns

### Spider Plugin System

The app loads content via "spiders" — pluggable crawlers defined in `catvod`. Spiders can be:
- **Java JAR**: loaded dynamically via `JarLoader`
- **JavaScript**: executed via QuickJS (`JsLoader`)
- **Python**: executed via Chaquopy (`PyLoader`)

The `Spider` interface (in `catvod`) defines these required methods:
- `init(Context ctx, String extend)`
- `homeContent(boolean filter)`
- `categoryContent(String tid, String pg, boolean filter, HashMap<String, String> extend)`
- `detailContent(List<String> ids)`
- `searchContent(String key, boolean quick)`
- `playerContent(String flag, String id, List<String> vipFlags)`

See `docs/SPIDER.md` for the full specification.

### EventBus Communication

Components communicate via EventBus (Greenrobot). Event classes live in `event/`. Subscribe with:
```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onPlayerEvent(PlayerEvent event) { ... }
```
Always unregister in `onDestroy()` to prevent leaks.

### Thread Management

Defined in `App.java`:
- **General executor**: fixed pool of 5 threads — `App.execute(runnable)`
- **Search executor**: fixed pool of 20 threads — `App.submitSearch(runnable)`
- **UI updates**: `App.post(runnable)` posts to the main handler

Never perform network or heavy work on the main thread; always use these executors.

### Configuration System

- `VodConfig` / `LiveConfig` — singleton config loaders parsed from remote or local JSON
- Config JSON schema documented in `docs/CONFIG.md`
- `Setting.java` wraps SharedPreferences with typed `get*()`/`put*()` helpers

### Room Database

- `AppDatabase` — single database instance
- `Migrations.java` — all schema migrations live here; always add a migration when changing schema
- DAOs use the `*Dao` naming convention

---

## Code Conventions

### Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Packages | Lowercase, reverse-domain | `com.fongmi.android.tv.ui` |
| Classes | PascalCase | `ParseJob`, `PlayerEvent` |
| Methods | camelCase | `homeContent()`, `getTimeout()` |
| Constants | UPPER_SNAKE_CASE | `Constant.TIMEOUT_PLAY` |
| Resources | snake_case | `activity_player.xml`, `ic_play.png` |
| Callbacks | `*Callback` suffix | `DohCallback`, `ParseCallback` |
| DAOs | `*Dao` suffix | `HistoryDao`, `CollectDao` |

### Localization

String resources support:
- `values/` — Default (English fallback)
- `values-zh-rCN/` — Simplified Chinese
- `values-zh-rTW/` — Traditional Chinese

Always add new user-visible strings to all three locales.

### Preferences

Use `Setting.java` exclusively for SharedPreferences access — never call `SharedPreferences` directly. Example:
```java
Setting.putTimeout(5000);
int timeout = Setting.getTimeout();
```

---

## Documentation Files

| File | Contents |
|------|---------|
| `docs/CONFIG.md` | Complete JSON schema for VOD/live config (sites, parses, lives, DoH, proxy, ads…) |
| `docs/SPIDER.md` | Spider interface specification and data structures (Result, Vod, Class, Filter…) |
| `docs/LOCAL.md` | Local HTTP API endpoints, remote push (danmaku, subtitles), multi-device sync |
| `README.md` | Project overview in Traditional Chinese |

---

## Testing

There are **no automated tests** in this project. All testing is manual. When making changes:
- Build the relevant flavor and test on a physical device or emulator
- For Spider changes, test against a real config/spider source
- For UI changes, test both `leanback` and `mobile` flavors

---

## CI/CD

No CI/CD pipelines are configured. Releases are built locally and distributed via Telegram. The `.github/` directory only contains `FUNDING.yml` (sponsorship links).

---

## Important Files to Know

| File | Purpose |
|------|---------|
| `app/build.gradle` | App-level dependencies, flavors, signing config |
| `catvod/build.gradle` | Spider library dependencies |
| `app/proguard-rules.pro` | ProGuard/R8 rules — update when adding new libraries |
| `chaquo/requirements.txt` | Python packages bundled into the app |
| `app/src/main/java/.../Constant.java` | All timeout/interval constants |
| `app/src/main/java/.../Setting.java` | All user preferences keys and accessors |
| `app/src/main/java/.../App.java` | Application lifecycle, thread pools, global singletons |

---

## Common Development Tasks

### Adding a New Preference Setting

1. Add constant key to `Setting.java`
2. Add `get*()` and `put*()` methods to `Setting.java`
3. Add UI entry in the appropriate settings dialog/activity

### Adding a New Spider Loader Type

1. Extend `BaseLoader` in `api/loader/`
2. Register in `BaseConfig` loading logic
3. Document in `docs/SPIDER.md`

### Adding a New Bean/Model Class

1. Create class in `app/src/main/java/.../bean/`
2. Use Gson annotations for JSON field mapping if needed
3. Add Room `@Entity` annotation and DAO if persistence is required
4. Register migration in `db/Migrations.java` if schema changes

### Adding a New Event

1. Create event class in `event/`
2. Post with `EventBus.getDefault().post(new MyEvent(...))`
3. Subscribe in target component; unregister in `onDestroy()`

---

## License

GNU General Public License v3.0 — see `LICENSE.md`.
