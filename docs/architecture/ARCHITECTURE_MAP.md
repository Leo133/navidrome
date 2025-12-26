# Architecture Map

This document provides detailed architecture diagrams and data flow documentation for the Navidrome codebase.

## System Overview

### High-Level Architecture

```mermaid
graph TB
    subgraph "Clients"
        WebUI[Web UI<br/>React + Material-UI]
        SubsonicClient[Subsonic Clients<br/>iOS/Android/Desktop]
        MPVClient[MPV Playback<br/>Jukebox Mode]
    end

    subgraph "Navidrome Server"
        subgraph "API Layer"
            NativeAPI[Native REST API<br/>/api/*]
            SubsonicAPI[Subsonic API<br/>/rest/*]
            PublicAPI[Public API<br/>/share/*]
            SSE[SSE Broker<br/>/api/events]
        end

        subgraph "Core Layer"
            MediaStreamer[Media Streamer]
            PlayTracker[Play Tracker]
            Scanner[Library Scanner]
            Artwork[Artwork Manager]
            Agents[Metadata Agents]
        end

        subgraph "Data Layer"
            DataStore[DataStore Interface]
            Repositories[SQL Repositories]
            SQLite[(SQLite DB)]
        end

        subgraph "Plugins"
            PluginMgr[Plugin Manager]
            WASMRuntime[WASM Runtime]
            Scrobblers[External Scrobblers]
        end
    end

    subgraph "External Services"
        LastFM[Last.fm]
        ListenBrainz[ListenBrainz]
        Spotify[Spotify API]
    end

    WebUI --> NativeAPI
    WebUI --> SSE
    SubsonicClient --> SubsonicAPI
    MPVClient --> SubsonicAPI

    NativeAPI --> DataStore
    SubsonicAPI --> DataStore
    SubsonicAPI --> MediaStreamer
    SubsonicAPI --> PlayTracker
    PublicAPI --> DataStore
    PublicAPI --> MediaStreamer

    MediaStreamer --> DataStore
    PlayTracker --> DataStore
    PlayTracker --> Scrobblers
    Scanner --> DataStore
    Artwork --> DataStore
    Agents --> Spotify
    Agents --> LastFM

    DataStore --> Repositories
    Repositories --> SQLite

    Scrobblers --> LastFM
    Scrobblers --> ListenBrainz
```

## Component Details

### API Layer

```mermaid
graph LR
    subgraph "Router (Chi)"
        MW[Middleware Stack]
    end

    subgraph "Middleware"
        Auth[Auth Middleware]
        CORS[CORS]
        RateLimit[Rate Limiting]
        Logger[Request Logger]
    end

    subgraph "Handlers"
        Native[Native API<br/>CRUD + Custom]
        Subsonic[Subsonic API<br/>XML/JSON]
        Public[Public Share<br/>Unauthenticated]
        Events[SSE Broker]
    end

    MW --> Auth
    Auth --> CORS
    CORS --> RateLimit
    RateLimit --> Logger
    Logger --> Native
    Logger --> Subsonic
    Logger --> Public
    Logger --> Events
```

### Data Flow: Authentication

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Handler
    participant U as User Repository
    participant J as JWT Generator

    C->>A: POST /auth/login<br/>{username, password}
    A->>U: GetByUsername(username)
    U-->>A: User record
    A->>A: Verify password (bcrypt)
    
    alt Password valid
        A->>J: Generate JWT<br/>{sub, uid, adm, exp}
        J-->>A: JWT token
        A->>A: Generate Subsonic token
        A-->>C: {token, subsonicSalt, subsonicToken, ...}
    else Password invalid
        A-->>C: 401 Unauthorized
    end
```

### Data Flow: Track Streaming

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Stream Handler
    participant M as MediaStreamer
    participant T as Transcoder
    participant FS as File System
    participant Cache as Transcode Cache

    C->>S: GET /rest/stream?id=xxx
    S->>M: GetStream(mediaFileId, options)
    M->>M: Load MediaFile from DB
    M->>M: Decide: raw or transcode?

    alt Raw streaming
        M->>FS: Open file
        FS-->>M: File handle
        M-->>C: Stream bytes (with Range support)
    else Transcoding required
        M->>Cache: Check cache
        alt Cache hit
            Cache-->>M: Cached stream
            M-->>C: Stream cached bytes
        else Cache miss
            M->>T: Start FFmpeg process
            T->>FS: Read input file
            T-->>M: Transcoded stream
            M->>Cache: Store in cache
            M-->>C: Stream transcoded bytes
        end
    end
```

### Data Flow: Scrobble Submission

```mermaid
sequenceDiagram
    participant C as Client
    participant H as Scrobble Handler
    participant PT as PlayTracker
    participant AR as Annotation Repo
    participant SR as Scrobble Repo
    participant ES as External Scrobblers
    participant SB as Scrobble Buffer

    C->>H: POST /rest/scrobble<br/>{id, time, submission=true}
    H->>PT: Submit(scrobbles)
    
    PT->>AR: IncPlayCount(mediaFile)
    PT->>AR: IncPlayCount(album)
    PT->>AR: IncPlayCount(artist)
    
    PT->>SR: RecordScrobble(mediaFileId, time)
    
    loop For each external scrobbler
        PT->>ES: Scrobble(track, time)
        alt Success
            ES-->>PT: OK
        else Failure
            PT->>SB: Enqueue(scrobble, service)
        end
    end
    
    PT-->>H: Success
    H-->>C: 200 OK
```

### Data Flow: Library Scan

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant SC as Scanner Controller
    participant P1 as Phase 1: Folders
    participant P2 as Phase 2: Missing
    participant P3 as Phase 3: Refresh
    participant P4 as Phase 4: Playlists
    participant DB as Database
    participant SSE as SSE Broker

    S->>SC: Trigger scan
    SC->>SSE: Broadcast: scanStatus = started

    SC->>P1: Scan folders
    P1->>P1: Walk filesystem
    P1->>P1: Extract metadata (TagLib)
    P1->>DB: Upsert media_file, album, artist
    P1-->>SC: Phase 1 complete

    SC->>P2: Check missing
    P2->>DB: Find orphaned records
    P2->>DB: Delete missing tracks
    P2-->>SC: Phase 2 complete

    SC->>P3: Refresh aggregates
    P3->>DB: Update album stats
    P3->>DB: Update artist stats
    P3-->>SC: Phase 3 complete

    SC->>P4: Import playlists
    P4->>P4: Parse M3U/PLS files
    P4->>DB: Upsert playlists
    P4-->>SC: Phase 4 complete

    SC->>SSE: Broadcast: scanStatus = completed
    SC->>SSE: Broadcast: refreshResource
```

## Database Schema

### Entity Relationship Diagram

```mermaid
erDiagram
    USER {
        string id PK
        string user_name UK
        string name
        string email
        boolean is_admin
        string password
        datetime last_login_at
    }

    LIBRARY {
        int id PK
        string name
        string path
    }

    MEDIA_FILE {
        string id PK
        int library_id FK
        string folder_id FK
        string album_id FK
        string artist_id FK
        string title
        int duration
        string path
    }

    ALBUM {
        string id PK
        int library_id FK
        string name
        string artist_id FK
        int song_count
    }

    ARTIST {
        string id PK
        string name
        int album_count
    }

    ANNOTATION {
        string user_id FK
        string item_id FK
        string item_type
        int play_count
        datetime play_date
        int rating
        boolean starred
    }

    SCROBBLES {
        string media_file_id FK
        string user_id FK
        int submission_time
    }

    PLAYLIST {
        string id PK
        string name
        string owner_id FK
        boolean public
    }

    PLAYER {
        string id PK
        string user_id FK
        string name
        boolean scrobble_enabled
    }

    USER ||--o{ ANNOTATION : "has"
    USER ||--o{ SCROBBLES : "creates"
    USER ||--o{ PLAYLIST : "owns"
    USER ||--o{ PLAYER : "uses"
    
    LIBRARY ||--o{ MEDIA_FILE : "contains"
    LIBRARY ||--o{ ALBUM : "contains"
    
    ALBUM ||--o{ MEDIA_FILE : "contains"
    ARTIST ||--o{ ALBUM : "creates"
    
    MEDIA_FILE ||--o{ ANNOTATION : "annotated by"
    MEDIA_FILE ||--o{ SCROBBLES : "scrobbled"
    
    PLAYLIST ||--o{ MEDIA_FILE : "contains"
```

## Extension Points

### For Telemetry (Phase 2)

```mermaid
graph TB
    subgraph "Current"
        PT[PlayTracker]
        SR[Scrobble Repo]
    end

    subgraph "Fork Additions"
        LE[Listen Event Repo]
        EA[Events API]
        NE[NATS Emitter]
    end

    PT --> SR
    PT -.-> LE
    PT -.-> NE
    EA -.-> LE

    style LE fill:#9f9
    style EA fill:#9f9
    style NE fill:#9f9
```

### For Social Features (Phase 3)

```mermaid
graph TB
    subgraph "Navidrome Core"
        API[Native API]
        SSE[SSE Broker]
    end

    subgraph "Social Service (External)"
        SocialAPI[GraphQL API]
        SocialDB[(PostgreSQL)]
        Feed[Activity Feed]
    end

    subgraph "Message Queue"
        NATS[NATS JetStream]
    end

    API --> NATS
    NATS --> SocialAPI
    SocialAPI --> SocialDB
    SocialAPI --> Feed

    style SocialAPI fill:#99f
    style SocialDB fill:#99f
    style Feed fill:#99f
    style NATS fill:#ff9
```

## Key Files Reference

### Core Components

| Component | Primary File | Purpose |
|-----------|--------------|---------|
| Entry Point | `cmd/root.go` | Application bootstrap |
| Configuration | `conf/configuration.go` | Config parsing |
| DataStore | `model/datastore.go` | Repository interfaces |
| PlayTracker | `core/scrobbler/play_tracker.go` | **Telemetry ingestion point** |
| MediaStreamer | `core/media_streamer.go` | Audio streaming |
| Scanner | `scanner/controller.go` | Library scanning |

### API Handlers

| API | Primary File | Auth |
|-----|--------------|------|
| Native REST | `server/nativeapi/native_api.go` | JWT |
| Subsonic | `server/subsonic/api.go` | JWT or Token |
| Public | `server/public/public.go` | None |
| SSE | `server/events/broker.go` | JWT |

### Database

| Component | File | Purpose |
|-----------|------|---------|
| Connection | `db/db.go` | SQLite connection |
| Migrations | `db/migrations/*.sql` | Schema evolution |
| Repositories | `persistence/*.go` | SQL implementations |

## Performance Characteristics

### Request Latency (Typical)

| Operation | P50 | P95 | P99 |
|-----------|-----|-----|-----|
| Auth (login) | 50ms | 100ms | 200ms |
| List albums | 10ms | 50ms | 100ms |
| Stream (raw) | 5ms | 20ms | 50ms |
| Stream (transcode) | 200ms | 500ms | 1s |
| Scrobble | 20ms | 50ms | 100ms |
| Full scan (1000 tracks) | 30s | 60s | 120s |

### Resource Usage

| Scenario | CPU | Memory | Disk I/O |
|----------|-----|--------|----------|
| Idle | <1% | 50MB | Low |
| 10 concurrent streams | 10-30% | 100MB | Medium |
| Full scan | 50-100% | 200MB | High |

## Security Considerations

### Authentication

- JWT with HS256 signature
- Session timeout configurable
- Rate limiting on login endpoint

### Data Access

- All queries scoped by user ID from context
- Admin flag checked for sensitive operations
- Shared content requires explicit share records

### Plugin Sandboxing

- WASM isolation
- Limited host services
- No direct filesystem access

## References

- [ARCHITECTURE_SNAPSHOT.md](../../ARCHITECTURE_SNAPSHOT.md) - Detailed codebase analysis
- [Navidrome Documentation](https://www.navidrome.org/docs/)
- [Subsonic API Documentation](http://www.subsonic.org/pages/api.jsp)
