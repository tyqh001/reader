# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Reader (阅读) is a self-hosted web reading server that aggregates novel content from configurable "book sources" across third-party websites. It's a Spring Boot 2.1 + Vert.x 3.8 backend with a Vue 2.6 frontend, supporting both server-only and JavaFX desktop modes.

## Build & run commands

All development commands go through `build.sh`:

```bash
./build.sh web       # Start Vue dev server on port 8081 (hot reload)
./build.sh sync      # Build web frontend + copy output to src/main/resources/web
./build.sh serve     # Build and run server-only mode (default port 8080, customizable: ./build.sh serve 9090)
./build.sh run       # Build and run desktop (JavaFX) mode
./build.sh build     # Create debug desktop package
./build.sh mac       # Package macOS app
./build.sh win       # Package Windows app
./build.sh linux     # Package Linux app
./build.sh yarn      # Run yarn commands inside web/ directory
./build.sh cli       # Server-only Gradle commands (uses cli.gradle which excludes JavaFX)
```

Frontend-only commands (from `web/` directory):
```bash
yarn serve           # Vue dev server
yarn build           # Production build
yarn lint            # ESLint
```

The Gradle wrapper (`./gradlew`) works directly too, though `build.sh` handles Java home detection and the UI/CLI build file switching.

## Architecture

### Entry points

- **`ReaderApplication.kt`** — Headless/server entry. Boots Spring, deploys `YueduApi` as a Vert.x verticle. Used in Docker and server deployments.
- **`ReaderUIApplication.kt`** — JavaFX desktop entry. Launches JavaFX first, then boots Spring in a background thread. Displays the web UI in a `WebView`. Used for desktop packages.

The server-only build (`cli.gradle`) excludes `ReaderUIApplication.kt` to drop the JavaFX dependency.

### Backend (src/main/java)

Two main namespaces:

- **`com.htmake.reader`** — Application layer (Spring Boot wiring, API endpoints, config)
  - `api/YueduApi.kt` — The main Vert.x verticle. Extends `RestVerticle` and registers all `/reader3/*` routes by delegating to controllers.
  - `api/controller/` — Route handlers: `BookController`, `BookSourceController`, `RssSourceController`, `UserController`, `WebdavController`, `ReplaceRuleController`, `BookmarkController`
  - `verticle/RestVerticle.kt` — Abstract base verticle with CORS, session handling, request logging, coroutine handler helpers
  - `config/` — `AppConfig` and `BookConfig` (Spring configuration beans)

- **`io.legado.app`** — Core domain logic (ported from the Android Legado app)
  - `model/webBook/` — Web book parsing pipeline (`BookInfo`, `BookChapterList`, `BookContent`, `BookList`)
  - `model/analyzeRule/` — Content extraction via JSoup, XPath, JSONPath, and regex
  - `model/localBook/` — Local format readers (EPUB, UMD, CBZ, TXT/PDF)
  - `model/rss/` — RSS feed parsing
  - `help/http/` — HTTP client layer (OkHttp + Retrofit with Vert.x adapter)
  - `data/entities/` — Data classes for books, sources, rules, etc.
  - `utils/` — Miscellaneous utilities (encoding detection, file I/O, etc.)

Third-party libraries embedded in source:
- `org.kxml2` — XML pull parser
- `me.ag2s.epublib` — EPUB reading/writing
- `me.ag2s.umdlib` — UMD format reading

Two JAR dependencies are vendored in `src/lib/`:
- `rhino-1.7.13-1.jar` — JavaScript engine for executing book source rules
- `xmlpull-1.1.3.1.jar` — XML pull parser

### Frontend (web/)

Vue 2 app with Element UI, Vuex, and Vue Router.

- **Two routes**: `/` (Index/bookshelf) and `/reader` (reader view)
- `web/src/views/` — Page-level components (`Index.vue`, `Reader.vue`)
- `web/src/components/` — Feature components (BookShelf, BookSource, Content, Explore, etc.)
- `web/src/plugins/` — Axios wrapper, Vuex store, cache helpers, config, event bus

Build output is synced to `src/main/resources/web/` (served as static resources by Vert.x at runtime).

### Data storage

All data is stored as JSON files under the `storage/` directory (configurable via `reader.app.storagePath`). No database — the app reads/writes JSON files organized by user space (`storage/data/<userName>/`). System defaults live in `src/main/resources/defaultData/`.

### API pattern

All API endpoints are under `/reader3/`. The `RestVerticle` base class provides:
- Session management (7-day timeout, stored in memory via `LocalSessionStore`)
- CORS with `Access-Control-Allow-Credentials`
- Coroutine handler wrappers (`route.coroutineHandler { ctx -> ... }`)
- Consistent JSON responses via `ctx.success(data)` and `ctx.error(throwable)`

### Docker deployment

The production deployment uses Docker Compose (`docker-compose.yaml`) with three services:
- **reader** — the main application (port 4396 mapped to 8080)
- **watchtower** — auto-updates the reader container daily at 4 AM
- **remote-webview** (optional, commented out) — separate WebView rendering service for book sources that require browser execution

Environment variables configure auth (`READER_APP_SECURE`, `READER_APP_SECUREKEY`, `READER_APP_INVITECODE`), user limits, and caching. The `reader.sh` script is an interactive deployment helper for Linux servers.

## Key constraints

- Java 11+ required (build script enforces this)
- Kotlin compiles to JVM 1.8 target
- The project uses Vert.x coroutines (`CoroutineVerticle`) — handlers run on `Dispatchers.IO`
- Book sources define scraping rules using JS (Rhino), XPath, JSONPath, CSS selectors (JSoup), or regex
- The JavaFX flavor is only for desktop builds; server/Docker builds exclude it
