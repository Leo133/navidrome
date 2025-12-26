# Navidrome Architecture Snapshot
**Analysis Date:** December 25, 2025  
**Repository:** navidrome/navidrome (master branch)  
**Purpose:** Foundation for social-first music player + deep listening analytics platform

---

## Executive Summary

Navidrome is a **mature, production-grade music streaming server** built in Go with a React-based web UI. It provides:
- Multi-user music library management with sophisticated catalog indexing
- Subsonic/OpenSubsonic API compatibility (supports 50+ third-party clients)
- Native REST API for its own web UI
- WASM-based plugin system for extensibility
- Real-time Server-Sent Events (SSE) for UI updates
- Transcoding pipeline with caching
- Basic scrobbling support (Last.fm, ListenBrainz) via external services

The codebase demonstrates **clean separation of concerns**, **dependency injection via Wire**, and **extensive test coverage**. It is well-positioned for extension rather than fork-and-modify.

---

## 1. Directory Structure & Responsibilities

### Core Application Layers

#### `/cmd/` - Application Entry Point & CLI
**Inspected Files:**
- [root.go](cmd/root.go) - Main command runner, orchestrates service startup
- [wire_gen.go](cmd/wire_gen.go) - Dependency injection wiring (Google Wire)
- [svc.go](cmd/svc.go), [scan.go](cmd/scan.go), [user.go](cmd/user.go) - Administrative commands

**Responsibilities:**
- Application bootstrap and lifecycle management
- CLI command handlers (scan, backup, user management, inspect)
- Starts multiple goroutines for concurrent services (server, scheduler, scanner, playback server, plugin manager)
- Signal handling for graceful shutdown

**Key Finding:** Uses `errgroup.WithContext` to manage parallel services. Any fatal error cancels all services.

---

#### `/conf/` - Configuration Management
**Inspected Files:**
- [configuration.go](conf/configuration.go) - Viper-based config with extensive options

**Responsibilities:**
- Environment variables, config file (TOML/YAML/JSON), CLI flags
- Configuration validation and defaults
- Over 90 configuration options covering:
  - Server settings (address, port, TLS, base path)
  - Scanner behavior (automatic scan, watch mode, schedule)
  - Transcoding, caching, artwork
  - External services (Last.fm, ListenBrainz, Spotify)
  - Plugins (directory, config per plugin)
  - Security (session timeout, rate limiting, CORS)

**Key Finding:** Configuration is **immutable after initialization**. Plugin-specific configs stored in `PluginConfig` map[string]map[string]string.

---

#### `/db/` - Database Layer
**Inspected Files:**
- [db.go](db/db.go) - SQLite connection management
- [migrations/](db/migrations/) - 80+ Goose migrations (from 2020 to 2025)

**Responsibilities:**
- SQLite database initialization and connection pooling
- Schema migrations (creation, versioning, rollback)
- Backup/restore functionality

**Schema Overview (from migrations):**
- **Core entities:** `media_file`, `album`, `artist`, `folder`, `genre`, `playlist`, `tag`
- **Library support:** `library` table (multi-library feature added 2024)
- **User data:** `user`, `player`, `annotation`, `scrobbles`, `scrobble_buffer`
- **Sharing:** `share` table
- **Metadata:** `artist_info` (external metadata cache)
- **Play queue:** `play_queue` (per-user playback state)
- **Radio:** `radio` (internet radio stations)

**Key Tables for Our Use Case:**

**`annotation` table** (user-specific interactions):
```sql
user_id       VARCHAR(255) NOT NULL
item_id       VARCHAR(255) NOT NULL  -- media_file, album, or artist ID
item_type     VARCHAR(255) NOT NULL  -- 'media_file', 'album', 'artist'
play_count    INTEGER      DEFAULT 0
play_date     DATETIME                -- timestamp of last play
rating        INTEGER      DEFAULT 0  -- 1-5 star rating
rated_at      DATETIME
starred       BOOL         DEFAULT FALSE
starred_at    DATETIME
UNIQUE(user_id, item_id, item_type)
```

**`scrobbles` table** (added Dec 2024 in migration `20251206013022`):
```sql
media_file_id    VARCHAR(255) NOT NULL REFERENCES media_file(id)
user_id          VARCHAR(255) NOT NULL REFERENCES user(id)
submission_time  INTEGER      NOT NULL  -- Unix timestamp
INDEX scrobbles_date (submission_time)
```

**`scrobble_buffer` table** (retry queue for external scrobblers):
```sql
id              VARCHAR(255) PRIMARY KEY
service         VARCHAR(255) NOT NULL  -- 'lastfm', 'listenbrainz', plugin name
user_id         VARCHAR(255) NOT NULL
play_time       DATETIME     NOT NULL
enqueue_time    DATETIME     NOT NULL
media_file_id   VARCHAR(255) NOT NULL
```

**Key Finding:** The `scrobbles` table is **new** (added 3 weeks ago). It only stores `(media_file_id, user_id, submission_time)` - minimal scrobble history. No position/duration data. This is a **foundation we can build on**.

---

#### `/model/` - Domain Models & Repository Interfaces
**Inspected Files:**
- [datastore.go](model/datastore.go) - Central DataStore interface
- [mediafile.go](model/mediafile.go), [album.go](model/album.go), [artist.go](model/artist.go)
- [user.go](model/user.go), [player.go](model/player.go), [playlist.go](model/playlist.go)
- [annotation.go](model/annotation.go), [scrobble.go](model/scrobble.go), [scrobble_buffer.go](model/scrobble_buffer.go)

**Responsibilities:**
- Domain entity structs (tags for JSON/DB serialization)
- Repository interfaces (CRUD operations)
- Business logic embedded in models (e.g., `Playlist.IsSmartPlaylist()`)

**Key Entities:**

**`User`:**
- Fields: ID, UserName, Name, Email, IsAdmin, LastLoginAt, LastAccessAt, Password
- Multi-library support: `Libraries` (many-to-many relationship)
- Methods: `HasLibraryAccess(libraryID int) bool`

**`Player`:**
- Represents a client device/app accessing Navidrome
- Fields: ID, Name, UserAgent, UserId, Client, IP, LastSeen, TranscodingId, MaxBitRate, ScrobbleEnabled
- **Key:** `ScrobbleEnabled` controls whether plays are tracked

**`MediaFile`:**
- 90+ fields covering all audio metadata (tags, MusicBrainz IDs, ReplayGain, lyrics, etc.)
- Embedded `Annotations` struct (play_count, play_date, rating, starred)
- Embedded `Bookmarkable` struct (for resume position)
- `Participants` field (multi-artist support)

**`Annotations`:**
```go
PlayCount int64      `json:"playCount,omitempty"`
PlayDate  *time.Time `json:"playDate,omitempty"`
Rating    int        `json:"rating,omitempty"`
RatedAt   *time.Time `json:"ratedAt,omitempty"`
Starred   bool       `json:"starred,omitempty"`
StarredAt *time.Time `json:"starredAt,omitempty"`
```

**`Scrobble` (new):**
```go
MediaFileID    string
UserID         string
SubmissionTime time.Time
```

**DataStore Interface:**
- Aggregates all repository interfaces
- Context-aware (user/player extracted from context)
- Transaction support: `WithTx()`, `WithTxImmediate()`

**Key Finding:** The domain model is **rich and well-structured**. The `DataStore` interface provides a **clean abstraction** for persistence. Adding new repository types (e.g., `ListenEventRepository`) would follow the existing pattern.

---

#### `/persistence/` - Repository Implementations
**Inspected Files:**
- [persistence.go](persistence/persistence.go) - DataStore implementation
- [sql_annotations.go](persistence/sql_annotations.go) - Annotation operations
- [scrobble_repository.go](persistence/scrobble_repository.go)
- [mediafile_repository.go](persistence/mediafile_repository.go), etc.

**Responsibilities:**
- SQL query construction using Squirrel (SQL builder)
- CRUD operations for all domain entities
- Annotation management (play counts, ratings, stars)
- Transaction management
- Filtering, sorting, pagination

**Key Methods:**

**`IncPlayCount(itemID string, ts time.Time)`** (in `sql_annotations.go`):
- Atomic increment of play_count
- Updates play_date to max(play_date, ts)
- Inserts new annotation row if none exists
- Used for media_file, album, and artist

**`RecordScrobble(mediaFileID string, submissionTime time.Time)`** (in `scrobble_repository.go`):
- Simple INSERT into scrobbles table
- UserID extracted from context
- No deduplication or validation

**Key Finding:** All repositories use **context for user/player extraction**. The `loggedUser(ctx)` helper retrieves the current user from request context. This pattern would extend naturally to telemetry event ingestion.

---

#### `/scanner/` - Library Scanning & Indexing
**Inspected Files:**
- [scanner.go](scanner/scanner.go), [controller.go](scanner/controller.go)
- [watcher.go](scanner/watcher.go) - File system watcher
- [phase_1_folders.go](scanner/phase_1_folders.go), etc. - Multi-phase scan strategy

**Responsibilities:**
- Full scan: recursively walk music folders, extract metadata (via taglib adapter)
- Incremental scan: detect changed/deleted files
- Folder-based organization (maintains folder tree hierarchy)
- Multi-library support (scan one or all libraries)
- Playlist import (M3U/PLS files)
- Sends progress events via SSE broker

**Scan Phases:**
1. **Phase 1:** Folder traversal, detect new/changed files
2. **Phase 2:** Process missing tracks (deletions)
3. **Phase 3:** Refresh album/artist aggregations
4. **Phase 4:** Import/sync playlists

**Key Finding:** Scanner is **CPU-intensive** but **well-optimized** (parallel processing, caching). It operates independently of streaming/playback. Adding hooks for post-scan analytics (e.g., "recommend newly added albums") is feasible.

---

#### `/core/` - Business Logic Layer
**Inspected Files:**
- [media_streamer.go](core/media_streamer.go) - Streaming/transcoding orchestration
- [scrobbler/](core/scrobbler/) - Play tracking and external scrobbling
- [players.go](core/players.go) - Player management
- [playlists.go](core/playlists.go), [share.go](core/share.go)
- [agents/](core/agents/) - External metadata providers (Last.fm, Spotify, etc.)
- [ffmpeg/](core/ffmpeg/) - FFmpeg transcoding
- [artwork/](core/artwork/) - Cover art extraction/caching

**Responsibilities:**
- High-level business operations (stream, scrobble, share, playlist management)
- Orchestration across multiple repositories
- External service integrations (agents, scrobblers)
- Caching strategies

**Key Components:**

**`MediaStreamer`:**
- Handles streaming requests (raw or transcoded)
- Selects transcoding format/bitrate based on player settings
- Uses transcoding cache (disk-based)
- Returns `Stream` interface (io.ReadCloser + io.Seeker)

**`PlayTracker`** (in `core/scrobbler/play_tracker.go`):
- **Central to our use case!**
- Manages "Now Playing" state (TTL-based in-memory cache)
- Increments play counts (`IncPlayCount` on media_file, album, artist)
- Records scrobbles to DB (`scrobbles` table)
- Dispatches to external scrobblers (Last.fm, ListenBrainz, plugins)
- Methods:
  - `NowPlaying(ctx, playerId, playerName, trackId, position)` - Update now-playing state
  - `Submit(ctx, []Submission)` - Record completed listens
  - `GetNowPlaying(ctx)` - Retrieve current now-playing list
- Uses `scrobble_buffer` for retry logic when external services fail

**Key Finding:** `PlayTracker` is the **existing telemetry ingestion point**. It currently:
- Records **only successful scrobbles** (timestamp when track was submitted as "played")
- Does **not** record play/pause/seek events
- Does **not** track listen intervals or completion percentage
- **This is our primary extension point.**

---

#### `/server/` - HTTP Server & API Layers
**Inspected Files:**
- [server.go](server/server.go) - Main HTTP server (chi router)
- [auth.go](server/auth.go) - JWT-based authentication
- [subsonic/](server/subsonic/) - Subsonic/OpenSubsonic API implementation
- [nativeapi/](server/nativeapi/) - Native REST API for web UI
- [public/](server/public/) - Public share endpoints
- [events/](server/events/) - Server-Sent Events (SSE) broker

**Responsibilities:**
- HTTP routing and middleware (CORS, rate limiting, logging, auth)
- Request/response serialization (JSON, XML for Subsonic)
- JWT token generation and validation
- SSE event broadcasting

**API Surfaces:**

**1. Subsonic API** (`/rest/*`):
- Full Subsonic 1.16.1 + OpenSubsonic extensions
- Endpoints: search, browse, stream, scrobble, playlists, ratings, bookmarks, etc.
- Authentication: username/password hash (token + salt), or JWT
- Response format: XML (default) or JSON
- **Key endpoints:**
  - `stream?id=<id>` - Stream audio (with optional transcoding)
  - `scrobble?id=<id>&time=<timestamp>&submission=true` - Record listen
  - `getNowPlaying` - Get currently playing tracks across all users
  - `star`, `unstar`, `setRating` - User annotations

**2. Native REST API** (`/api/*`):
- RESTful JSON API for the web UI
- Uses react-admin patterns (GetList, GetOne, Update, Create, Delete)
- Resources: song, album, artist, playlist, user, player, radio, transcoding, etc.
- Authentication: JWT (refresh on each request via middleware)
- **Key endpoints:**
  - `/api/song`, `/api/album`, `/api/artist` - Standard CRUD
  - `/api/queue` - Play queue management
  - `/api/share` - Sharing management
  - `/api/keepalive` - Session heartbeat

**3. Public Endpoints** (`/share/*`):
- Unauthenticated share URLs
- `/share/<id>` - View shared content
- `/share/<id>/stream/<trackId>` - Stream shared track
- `/share/<id>/download` - Download shared content

**4. Server-Sent Events** (`/api/events`):
- Real-time event stream for web UI updates
- Events: `scanStatus`, `refreshResource`, `serverStart`, `nowPlayingCount`, `keepAlive`
- Client connects once, server pushes updates
- **Key for UI reactivity**

**Authentication Flow:**
1. POST `/auth/login` with username/password → returns JWT + Subsonic token
2. Subsequent requests include JWT in `Authorization: Bearer <token>` header
3. Subsonic clients use `u=<username>&t=<token>&s=<salt>` query params
4. Middleware extracts user/player from token, adds to request context

**Key Finding:** The API layer is **mature and stable**. The Subsonic API has **50+ client implementations**, making it a de-facto standard. Adding new endpoints (e.g., telemetry ingestion) should follow the native API pattern. The SSE broker provides **real-time push** capability.

---

#### `/plugins/` - WASM Plugin System
**Inspected Files:**
- [manager.go](plugins/manager.go) - Plugin lifecycle manager
- [README.md](plugins/README.md) - Comprehensive plugin documentation
- [api/api.proto](plugins/api/), [host/](plugins/host/) - Plugin protocol definitions

**Responsibilities:**
- Plugin discovery (scans plugins directory for manifest.json)
- WASM compilation and caching
- Plugin instance pooling (default 8 instances, 1m TTL)
- Host service provision (HTTP, scheduler, config, cache, artwork, WebSocket, Subsonic API)
- Capability-based loading (MetadataAgent, Scrobbler, SchedulerCallback, WebSocketCallback)

**Capabilities:**
- **MetadataAgent:** Fetch artist/album info, images (e.g., a plugin for Discogs)
- **Scrobbler:** External scrobbling (e.g., a plugin for Maloja, Funkwhale)
- **SchedulerCallback:** Periodic tasks (e.g., generate weekly stats)
- **WebSocketCallback:** Handle WebSocket events (e.g., chat integration)

**Host Services Available to Plugins:**
- **HTTP:** Make external requests (GET, POST, etc.)
- **Config:** Access plugin-specific configuration
- **Scheduler:** Schedule delayed/recurring tasks
- **Cache:** TTL-based key-value cache
- **Artwork:** Generate public artwork URLs
- **SubsonicAPI:** Call Navidrome's Subsonic API (for data access)
- **WebSocket:** Publish messages to WebSocket clients

**Key Finding:** The plugin system is **powerful and well-architected**. Plugins are **sandboxed WASM** (security, portability). If we want to add social/telemetry features as plugins (rather than in-core), this system supports it. However, for **performance-critical telemetry ingestion**, in-core implementation is preferable.

---

#### `/ui/` - React Web UI
**Inspected Files:**
- [package.json](ui/package.json) - React 17, react-admin 3.19, Material-UI 4
- [src/](ui/src/) - React components organized by feature

**Responsibilities:**
- Single-page app (SPA) for music browsing, playback, playlist management
- Built with react-admin (CRUD framework on top of React)
- Material-UI for components and theming
- Custom audio player (`navidrome-music-player` package)
- Websocket/SSE integration for real-time updates

**Key Features:**
- Library browsing (albums, artists, songs, genres, folders)
- Search (full-text, with filters)
- Playlists (create, edit, drag-and-drop reordering)
- Now playing / queue management
- User settings (transcoding, theme, language, scrobbling)
- Admin panel (user management, library management)

**Key Finding:** The UI is **feature-complete** for basic music server functionality. To add social/analytics features, we'd need to:
- Extend the native API with new endpoints
- Add new React components/pages
- Implement new dataProvider methods (for API calls)
- Add UI for privacy controls (incognito mode, visibility settings)

---

#### `/utils/`, `/log/`, `/consts/` - Supporting Utilities
**Responsibilities:**
- Logging (structured logging with logrus, redaction of sensitive data)
- Constants (version, paths, defaults)
- Utilities (caching, random, string manipulation, image processing, etc.)

---

## 2. Authentication & Authorization

**Mechanism:** JWT-based (HS256 signature)

**Flow:**
1. User POSTs credentials to `/auth/login`
2. Server validates against `user` table (password is encrypted with AES-256-GCM)
3. Server generates JWT with claims: `sub` (username), `uid` (user ID), `adm` (isAdmin), `iat` (issued at), `exp` (expiration)
4. JWT secret stored in DB (`property` table, key=`JWTSecret`), encrypted with server key
5. JWT has session timeout (default 24h, configurable)
6. Middleware verifies JWT on each request, injects user into context

**User Model:**
- ID, UserName, Name, Email, IsAdmin
- Password (encrypted, never sent to client)
- LastLoginAt, LastAccessAt (tracked automatically by middleware)
- Multi-library access control (many-to-many `user_library` join table)

**Player Model:**
- Automatically created when a client connects
- Identified by: (UserID, Client name, UserAgent) tuple
- Stores per-player settings: TranscodingId, MaxBitRate, ScrobbleEnabled
- Tracked in `player` table

**Context Pattern:**
- Request context carries: User, Player, ClientUniqueId (for tracking multiple tabs)
- Extracted via: `request.UserFrom(ctx)`, `request.PlayerFrom(ctx)`
- All repository methods receive context → automatic user scoping

**Key Finding:** Auth is **mature and secure**. User/Player context propagation is **consistent throughout the codebase**. Any new telemetry ingestion would inherit this context automatically.

---

## 3. Scrobbling & Play Tracking (Current State)

**Current Capabilities:**

**1. Internal Play Tracking:**
- `PlayTracker` (singleton in `core/scrobbler/play_tracker.go`)
- Tracks "Now Playing" state (in-memory, per player)
- Records play counts in `annotation` table (per user, per item)
- Records scrobbles in `scrobbles` table (media_file_id, user_id, timestamp)

**2. External Scrobblers:**
- Last.fm (OAuth, sends now-playing + scrobbles)
- ListenBrainz (token-based, sends listens)
- Plugin-based scrobblers (via WASM plugins)

**Workflow:**

**Now Playing Update:**
1. Client calls Subsonic `scrobble?id=<id>&submission=false&position=<seconds>` OR Native API `/api/song/<id>/nowplaying`
2. Server calls `PlayTracker.NowPlaying(ctx, playerId, playerName, trackId, position)`
3. PlayTracker:
   - Stores `NowPlayingInfo` in memory (with TTL = remaining track duration + 5s)
   - Broadcasts `nowPlayingCount` event via SSE
   - If player.ScrobbleEnabled: enqueues nowPlaying dispatch to external scrobblers

**Scrobble Submission:**
1. Client calls Subsonic `scrobble?id=<id>&submission=true&time=<timestamp>`
2. Server calls `PlayTracker.Submit(ctx, []Submission)`
3. PlayTracker:
   - Calls `IncPlayCount` on media_file, album, artist (updates `annotation` table)
   - If `conf.Server.EnableScrobbleHistory`: inserts into `scrobbles` table
   - If player.ScrobbleEnabled: dispatches to external scrobblers (Last.fm, ListenBrainz, plugins)
   - Broadcasts `refreshResource` event via SSE

**Scrobble Buffer (Retry Queue):**
- External scrobbler calls can fail (network issues, API downtime)
- Failed scrobbles are enqueued in `scrobble_buffer` table
- Background worker retries periodically

**Data Captured (Current):**
- **`annotation` table:** play_count (integer), play_date (datetime of last play)
- **`scrobbles` table:** media_file_id, user_id, submission_time (when scrobbled)
- **Not captured:** play duration, pause/resume events, seek events, listen intervals, skip events

**Key Finding:** Current play tracking is **coarse-grained**. A "scrobble" means "user listened to this track" (submitted at some point, usually after 50% completion or 240s, per Subsonic/Last.fm conventions). **We have no visibility into actual playback behavior** (pauses, seeks, partial listens, replays, skips).

---

## 4. Media Streaming & Transcoding

**Streaming Flow:**
1. Client requests `GET /rest/stream?id=<id>&maxBitRate=<kbps>&format=<format>`
2. Server loads MediaFile from DB
3. `MediaStreamer.DoStream()` decides:
   - **Raw streaming:** If format matches and bitrate ≤ original → serve file directly
   - **Transcoding:** Otherwise, spawn FFmpeg process, cache output, stream
4. Server responds with audio stream (supports HTTP Range requests for seeking)

**Transcoding:**
- FFmpeg-based (formats: mp3, opus, ogg, aac, webm, etc.)
- Per-user/per-player settings (MaxBitRate, PreferredFormat)
- Output cached to disk (`[CacheFolder]/transcoding/`)
- Cache key: `<fileId>.<updatedAt>.<bitrate>.<format>.<offset>`

**Key Finding:** Streaming is **independent of play tracking**. The client must **explicitly call `scrobble`** to record a listen. This is by design (Subsonic API spec). For deep telemetry, we need the client to send more events (play, pause, seek, etc.) to a new endpoint.

---

## 5. Database Schema (Relevant Tables)

### Core Music Catalog
```sql
library(id, name, path, ...)
folder(id, library_id, path, name, parent_id, ...)
media_file(id, folder_id, library_id, path, title, album_id, artist_id, duration, bitrate, ...)
album(id, library_id, name, artist_id, year, song_count, duration, ...)
artist(id, name, album_count, song_count, ...)
genre(id, name)
tag(tag_name, library_id, count)  -- free-form tags from ID3
```

### User Interactions
```sql
annotation(
  user_id, item_id, item_type,  -- PK (user, item, type)
  play_count, play_date,
  rating, rated_at,
  starred, starred_at
)

scrobbles(
  media_file_id, user_id, submission_time
)  -- Recently added (Dec 2024), simple history log

scrobble_buffer(
  id, service, user_id, play_time, enqueue_time, media_file_id
)  -- Retry queue for external scrobblers
```

### Playlists & Social
```sql
playlist(id, name, comment, owner_id, public, sync, created_at, updated_at, rules)
playlist_track(id, playlist_id, media_file_id)
share(id, user_id, description, downloadable, expires_at, resource_ids, resource_type, ...)
```

### Playback State
```sql
player(id, name, user_agent, user_id, client, ip, last_seen, transcoding_id, max_bit_rate, scrobble_enabled)
play_queue(id, user_id, current, position, changed_by, created_at, updated_at)
bookmark(id, user_id, item_id, item_type, position, comment, created_at, updated_at)
```

### Users & Auth
```sql
user(id, user_name, name, email, is_admin, password, last_login_at, last_access_at, created_at, updated_at)
user_props(user_id, key, value)  -- User preferences (JSON-encoded)
property(id, value)  -- System properties (JWTSecret, version, etc.)
```

---

## 6. Extension Points & Integration Patterns

Based on the architecture, here are the **cleanest extension points** for adding social and deep analytics:

### Option A: In-Core Extensions (Recommended for Performance)
1. **New Repository Interfaces:**
   - `ListenEventRepository` (for fine-grained telemetry: play, pause, seek, interval)
   - `SocialGraphRepository` (for follows, likes, comments)
   - `FeedRepository` (for activity feeds)

2. **New API Endpoints:**
   - Native API: `/api/events/listen` (POST bulk events from client)
   - Native API: `/api/social/*` (follows, likes, comments, feed)
   - Native API: `/api/analytics/*` (stats, heatmaps, recommendations)

3. **Enhanced PlayTracker:**
   - Extend `PlayTracker.Submit()` to accept detailed event payloads
   - Add `PlayTracker.RecordEvent(ctx, eventType, payload)` method
   - Maintain backward compatibility with existing scrobble endpoints

4. **Client Changes:**
   - Custom web UI client (or fork navidrome-music-player) to emit detailed events
   - Buffered event queue (send in batches to reduce requests)
   - Respect incognito mode (client-side flag, no events sent)

### Option B: Overlay Services (Better for Decoupling)
1. **Telemetry Ingest Service (separate microservice):**
   - Receives events from custom clients via HTTP/WebSocket
   - High-throughput ingestion (buffered, batch writes)
   - Stores raw events in append-only table/store
   - Generates derived metrics (rollups, aggregates) asynchronously

2. **Social Service (separate microservice):**
   - Manages social graph, activity feed, comments
   - Exposes GraphQL or REST API
   - Listens to Navidrome events via webhook/SSE (play counts, new playlists, etc.)

3. **Navidrome Changes (Minimal):**
   - Add webhook/event emission for key actions (play, like, playlist create)
   - Add optional reverse proxy config (route `/api/social/*` to Social Service)
   - Client authenticates with Navidrome JWT, services validate via shared secret or public key

### Option C: Hybrid (Pragmatic)
1. **Small Navidrome changes:**
   - Add `ListenEventRepository` and POST `/api/events/listen` endpoint
   - Store raw events in new `listen_event` table
   - Emit events to message queue (NATS/Redis) asynchronously

2. **External Analytics Service:**
   - Consumes events from queue
   - Computes derived metrics (heatmaps, completion %, skip rates)
   - Stores aggregates in time-series DB (InfluxDB, TimescaleDB)
   - Exposes query API for dashboards

3. **External Social Service:**
   - Standalone (see Option B)

---

## 7. Technology Stack

### Backend
- **Language:** Go 1.25
- **Web Framework:** chi (lightweight HTTP router)
- **Database:** SQLite 3 (with dbx query builder + Squirrel)
- **Migrations:** Goose
- **DI:** Google Wire (compile-time dependency injection)
- **Auth:** JWT (jwtauth/v5, lestrrat-go/jwx)
- **Logging:** logrus (structured logging)
- **Testing:** Ginkgo + Gomega (BDD-style)
- **FFmpeg:** Embedded (external binary, wrapped in Go)
- **Plugins:** WASM (tetratelabs/wazero runtime)

### Frontend
- **Framework:** React 17
- **UI Library:** Material-UI 4
- **Admin Framework:** react-admin 3.19
- **State Management:** Redux + redux-saga
- **Audio Player:** navidrome-music-player (custom fork of SoundManager2)
- **Build Tool:** Vite
- **Package Manager:** npm

### External Integrations
- **Scrobblers:** Last.fm, ListenBrainz
- **Metadata:** Last.fm, Spotify, Deezer (via agents)
- **Auth:** External authentication (LDAP, OAuth) via reverse proxy headers

---

## 8. Deployment & Operations

**Deployment Targets:**
- Standalone binary (Linux, macOS, Windows, FreeBSD)
- Docker (official images: `deluan/navidrome`)
- Kubernetes (community helm charts)
- Systemd service (contrib/navidrome.service)

**Operational Characteristics:**
- **Resource Usage:** Very low (10-50MB RAM, <1% CPU idle)
- **Concurrency:** Handles 100+ concurrent streams easily
- **Database:** Single SQLite file (WAL mode, concurrent reads)
- **Scaling:** Vertical (single instance, limited by SQLite write throughput)
- **Observability:** Prometheus metrics endpoint (optional), structured logs

**Key Constraints:**
- SQLite limits write concurrency (~1000 writes/sec)
- Transcoding cache can grow large (cleanup via LRU)
- Scanner is CPU-bound (can pause/resume manually)

---

## 9. Data Flow Diagrams

### Current Play Tracking Flow
```
┌──────────┐
│  Client  │ (Web UI, mobile app, desktop app)
└────┬─────┘
     │ 1. Stream request: GET /rest/stream?id=123
     ├──────────────────────────────────────────────┐
     │                                              │
     ▼                                              │
┌─────────────────┐                                │
│ Subsonic API    │                                │
│ /rest/stream    │                                │
└────┬────────────┘                                │
     │ 2. Load MediaFile                           │
     ├──────────────────────────┐                  │
     │                          │                  │
     ▼                          ▼                  │
┌─────────────┐         ┌──────────────┐          │
│ MediaFile   │         │ MediaStreamer│          │
│ Repository  │         └──────┬───────┘          │
└─────────────┘                │                  │
                               │ 3. Stream audio  │
                               └──────────────────┘
                                                   │
     ┌─────────────────────────────────────────────┘
     │ 4. During/after playback:
     │    POST /rest/scrobble?id=123&submission=true&time=<timestamp>
     ▼
┌─────────────────┐
│ Subsonic API    │
│ /rest/scrobble  │
└────┬────────────┘
     │ 5. Call PlayTracker.Submit()
     ▼
┌──────────────────────┐
│ PlayTracker          │
│ (core/scrobbler)     │
└────┬─────────────────┘
     │ 6. Update annotations
     ├───────────────────────────┐
     │                           │
     ▼                           ▼
┌─────────────────┐     ┌──────────────────┐
│ IncPlayCount()  │     │ RecordScrobble() │
│ (annotation)    │     │ (scrobbles)      │
└─────────────────┘     └──────────────────┘
     │                           │
     │ 7. If ScrobbleEnabled     │
     ▼                           ▼
┌──────────────────────────────────┐
│ External Scrobblers              │
│ (Last.fm, ListenBrainz, plugins) │
└──────────────────────────────────┘
```

### Current Data Access Pattern
```
HTTP Request → Middleware (Auth) → Context (User, Player) → Handler → DataStore → Repository → SQL
```

### Event Broadcasting (SSE)
```
Business Logic → Broker.SendMessage(event) → SSE Clients (Web UI)
```

---

## 10. Key Findings & Recommendations

### Strengths (Leverage These)
1. **Clean Architecture:** Clear separation (model/persistence/core/server)
2. **Extensible Plugin System:** WASM-based, secure, well-documented
3. **Rich Domain Model:** MediaFile has 90+ fields, comprehensive metadata
4. **Context-Aware Persistence:** User/player scoping is automatic
5. **Real-Time Events:** SSE broker enables live UI updates
6. **Mature API:** Subsonic compatibility = 50+ client apps work out-of-box
7. **Transaction Support:** `WithTx()` for atomic operations

### Gaps (Address These)
1. **Coarse Play Tracking:** Only records "listened" (no intervals, pauses, seeks)
2. **No Social Features:** No follows, likes (beyond stars), comments, feed
3. **Limited Analytics:** Just play counts, no completion %, skip points, replay patterns
4. **Single-Server Constraint:** SQLite limits horizontal scaling
5. **Client-Driven Scrobbling:** Server trusts client timestamps (spoofing possible)

### Recommended Integration Pattern: **Hybrid (Option C)**

**Rationale:**
- **Minimize Navidrome changes:** Navidrome is upstream-maintained; deep forking creates merge conflicts
- **High-performance telemetry:** In-core event ingestion (low latency, no network hop)
- **Decoupled analytics:** Separate service for heavy computation (heatmaps, ML, dashboards)
- **Decoupled social:** Separate service for social graph (scales independently)
- **Reuse Navidrome's strengths:** Catalog, streaming, Subsonic API, auth

**Implementation:**
1. **Navidrome changes (1-2 weeks):**
   - Add `listen_event` table (user_id, media_file_id, event_type, timestamp, position, metadata)
   - Add `POST /api/events/listen` endpoint (bulk insert events)
   - Add `ListenEventRepository` interface + SQL implementation
   - Optionally: emit events to message queue (NATS/Redis) for async processing
   - Add privacy controls: per-user `incognito_mode` flag (stored in `user_props`)

2. **Custom Client (2-3 weeks):**
   - Fork navidrome-music-player or build lightweight client
   - Emit events: play, pause, resume, seek, ended, error
   - Buffer events locally (IndexedDB), send in batches every 10s
   - Respect incognito mode (no events sent)
   - Derive "listened interval" from play/pause/seek events

3. **Analytics Service (3-4 weeks):**
   - Consume events from Navidrome (via message queue or direct DB read)
   - Compute derived metrics:
     - Completion % (sum of listened intervals / track duration)
     - Skip cliff detection (frequent pause/skip at timestamp X)
     - Replay hotspots (multiple play events for same track/timestamp)
     - Session stats (consecutive plays, genre diversity, etc.)
   - Store aggregates in time-series DB (TimescaleDB, ClickHouse)
   - Expose GraphQL API for queries

4. **Social Service (4-5 weeks):**
   - Standalone microservice (Go/Node.js/Python)
   - Data model: users, follows, likes, comments, feed_events
   - API: GraphQL or REST (follows, like, comment, feed)
   - Listen to Navidrome events (new playlists, top tracks) via webhook
   - PostgreSQL for relational data, Redis for feed caching

5. **Privacy Layer (1 week):**
   - UI controls: "Incognito session" toggle, "Profile visibility" (private/friends/public)
   - Per-device tracking opt-out (stored in `user_props` keyed by player_id)
   - GDPR-style export (JSON dump of all user data)
   - GDPR-style delete (cascade delete with audit log)

---

## 11. References & File Paths

### Key Files Inspected
- [cmd/root.go](cmd/root.go) - Application entry point
- [cmd/wire_gen.go](cmd/wire_gen.go) - Dependency injection
- [server/server.go](server/server.go) - HTTP server
- [server/auth.go](server/auth.go) - Authentication
- [server/subsonic/api.go](server/subsonic/api.go) - Subsonic API routing
- [server/subsonic/media_annotation.go](server/subsonic/media_annotation.go) - Scrobbling
- [server/nativeapi/native_api.go](server/nativeapi/native_api.go) - Native API
- [server/events/events.go](server/events/events.go) - SSE broker
- [core/media_streamer.go](core/media_streamer.go) - Streaming
- [core/scrobbler/play_tracker.go](core/scrobbler/play_tracker.go) - **Critical!**
- [core/scrobbler/interfaces.go](core/scrobbler/interfaces.go) - Scrobbler interface
- [model/datastore.go](model/datastore.go) - DataStore interface
- [model/user.go](model/user.go), [model/player.go](model/player.go)
- [model/annotation.go](model/annotation.go), [model/scrobble.go](model/scrobble.go)
- [persistence/sql_annotations.go](persistence/sql_annotations.go) - Annotation SQL
- [persistence/scrobble_repository.go](persistence/scrobble_repository.go)
- [db/migrations/20251206013022_create_scrobbles_table.sql](db/migrations/20251206013022_create_scrobbles_table.sql)
- [plugins/README.md](plugins/README.md) - Plugin system docs
- [plugins/manager.go](plugins/manager.go) - Plugin manager

### Build & Deploy
- [Makefile](Makefile) - Build targets
- [Dockerfile](Dockerfile) - Container image
- [go.mod](go.mod) - Go dependencies

---

**End of Architecture Snapshot**
