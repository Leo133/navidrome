# Integration Pattern Decision
**Project:** Social-First Music Player + Deep Listening Analytics  
**Base:** Navidrome Fork  
**Date:** December 25, 2025

---

## Executive Summary

**Decision: Hybrid Architecture (Option C)**

We will build the social-first music player using a **hybrid integration pattern**:
- **Minimal Navidrome changes** for high-performance telemetry ingestion (in-core)
- **External services** for analytics computation and social features (overlay)
- **Custom client(s)** that emit detailed playback events and consume social APIs

This approach balances:
- **Performance:** Low-latency event ingestion without network hops
- **Decoupling:** Social/analytics services scale independently, different technology stacks allowed
- **Upstream compatibility:** Minimal divergence from Navidrome mainline (easier to pull upstream updates)
- **Flexibility:** Can deploy components independently, swap implementations

---

## Pattern Comparison

### Pattern A: Tight Integration (Extend Navidrome Directly)

**Approach:** Add all social + telemetry features directly to Navidrome codebase.

**Pros:**
- Single binary, simple deployment
- No inter-service communication overhead
- Can leverage existing DataStore/Repository patterns
- Consistent auth/context model

**Cons:**
- **Large fork burden:** Difficult to merge upstream changes (Navidrome is actively developed)
- **Scaling constraints:** SQLite write throughput limits (telemetry is write-heavy)
- **Technology lock-in:** Must use Go for all features
- **Monolith risks:** Social features (feeds, comments) have different scaling needs than streaming
- **Testing complexity:** Large codebase, harder to test in isolation

**Fork Divergence Estimate:** 5,000-10,000 LoC added, 50+ files changed
**Upstream Merge Difficulty:** High (conflicts in core files)

### Pattern B: Overlay Architecture (External Services Only)

**Approach:** Keep Navidrome completely untouched, build all new features as separate microservices.

**Pros:**
- **Zero fork burden:** Can upgrade Navidrome without conflicts
- **Independent scaling:** Each service scales based on its workload
- **Technology freedom:** Use best tool for each job (Go, Python, Node.js, etc.)
- **Team autonomy:** Different teams can own different services
- **Gradual rollout:** Deploy features incrementally

**Cons:**
- **Network latency:** Telemetry events require HTTP calls to external service (10-50ms overhead)
- **Complex auth:** Must replicate or proxy Navidrome JWT validation
- **Data duplication:** External services need copies of user/catalog data (eventual consistency issues)
- **Operational overhead:** More services to deploy, monitor, version
- **Client complexity:** Custom clients must talk to multiple APIs (Navidrome + Social + Analytics)

**Fork Divergence Estimate:** 0 LoC in Navidrome (maybe add webhook config option)
**Upstream Merge Difficulty:** Zero (no changes)

### Pattern C: Hybrid (Recommended)

**Approach:** Small Navidrome changes for telemetry ingestion + external services for analytics/social.

**Pros:**
- **Low-latency ingestion:** Events written directly to Navidrome DB (same-process)
- **Minimal fork:** Only add ~500 LoC (new table, endpoint, repository)
- **Decoupled processing:** Heavy analytics in separate service (won't impact streaming)
- **Flexible analytics:** Can use specialized DB (TimescaleDB, ClickHouse) for time-series
- **Scalable social:** Social graph in dedicated service (PostgreSQL, graph DB)
- **Reuse Navidrome auth:** Custom client uses Navidrome JWT, services validate via shared secret

**Cons:**
- **Some fork burden:** Must maintain small patch against upstream
- **Dual deployment:** Need Navidrome + external services (but can start with just Navidrome)
- **Data flow complexity:** Events ingested in Navidrome, consumed by analytics service

**Fork Divergence Estimate:** 500-800 LoC in Navidrome (new repository, endpoint, migration)
**Upstream Merge Difficulty:** Low (changes are isolated, no modifications to existing code)

---

## Decision: Hybrid Pattern (C)

### Rationale

1. **Performance is Critical for Telemetry**
   - Telemetry events are high-frequency (every pause/seek/resume)
   - Network latency (even 10ms) adds up: 100 events → 1 second overhead
   - In-process writes to SQLite are <1ms (no serialization, no network)

2. **Navidrome's Strengths Should Be Leveraged**
   - Mature catalog indexing and search
   - Reliable streaming and transcoding
   - Subsonic API compatibility (50+ clients)
   - JWT auth and user management
   - **We don't need to reimplement these**

3. **Analytics and Social Scale Differently**
   - Analytics: read-heavy, time-series aggregations, ML workloads → specialized DB/compute
   - Social: graph queries (followers-of-followers), feed generation → Redis/PostgreSQL
   - Streaming: high concurrency, low CPU → Navidrome already optimized
   - **Monolith would be suboptimal for all three**

4. **Minimal Fork Reduces Maintenance Burden**
   - Navidrome is actively developed (~10-20 commits/week)
   - Large forks require manual conflict resolution on every upstream release
   - Small, isolated changes are easier to rebase/reapply
   - **Could even upstream telemetry changes if they're generic enough**

5. **Gradual Migration Path**
   - Phase 0-2: Deploy only Navidrome changes (basic telemetry)
   - Phase 3: Add analytics service (read from Navidrome DB)
   - Phase 4: Add social service (independent deployment)
   - **Can defer external services until needed**

---

## Hybrid Architecture Design

### Component Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                      Custom Web/Mobile Client                    │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ Audio Player   │  │ Social UI       │  │ Analytics UI     │  │
│  │ (emit events)  │  │ (feed, follows) │  │ (heatmaps, stats)│  │
│  └────────┬───────┘  └────────┬────────┘  └────────┬─────────┘  │
└───────────┼──────────────────┼───────────────────────┼───────────┘
            │                  │                       │
            │ JWT Auth         │ JWT Auth              │ JWT Auth
            ▼                  ▼                       ▼
┌───────────────────────────────────────────────────────────────────┐
│                         Navidrome (Modified)                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Existing APIs:                                             │  │
│  │ • Subsonic API (/rest/*)     - streaming, catalog, scrobble│  │
│  │ • Native API (/api/*)        - CRUD, playlists, users      │  │
│  │ • Public API (/share/*)      - unauthenticated shares      │  │
│  │ • SSE (/api/events)          - real-time push              │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ NEW: Telemetry Ingestion                                   │  │
│  │ • POST /api/events/listen    - bulk event ingestion        │  │
│  │ • ListenEventRepository      - write to listen_event table │  │
│  │ • Message Queue Emitter      - publish to NATS/Redis       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Database (SQLite)                                          │  │
│  │ • Existing: media_file, album, artist, annotation, user    │  │
│  │ • NEW: listen_event (event log, append-only)               │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────┬──────────────────┬───────────────────────┘
                          │ Message Queue    │ Direct DB Read
                          │ (async)          │ (batch)
                          ▼                  ▼
        ┌─────────────────────────────┐  ┌─────────────────────────┐
        │   Analytics Service         │  │   Social Service        │
        │  ┌──────────────────────┐   │  │  ┌──────────────────┐   │
        │  │ Event Consumer       │   │  │  │ Social Graph     │   │
        │  │ (NATS/Redis)         │   │  │  │ (follows, likes) │   │
        │  └──────────────────────┘   │  │  └──────────────────┘   │
        │  ┌──────────────────────┐   │  │  ┌──────────────────┐   │
        │  │ Metrics Computation  │   │  │  │ Activity Feed    │   │
        │  │ (completion %, skip) │   │  │  │ (aggregation)    │   │
        │  └──────────────────────┘   │  │  └──────────────────┘   │
        │  ┌──────────────────────┐   │  │  ┌──────────────────┐   │
        │  │ Time-Series DB       │   │  │  │ Comments/Moments │   │
        │  │ (TimescaleDB)        │   │  │  │ (timestamped)    │   │
        │  └──────────────────────┘   │  │  └──────────────────┘   │
        │  ┌──────────────────────┐   │  │  ┌──────────────────┐   │
        │  │ Query API            │   │  │  │ PostgreSQL       │   │
        │  │ (GraphQL)            │   │  │  │ (relational)     │   │
        │  └──────────────────────┘   │  │  └──────────────────┘   │
        └─────────────────────────────┘  └─────────────────────────┘
```

### Data Flow

**1. Telemetry Ingestion (High-Frequency, Low-Latency)**
```
Client → POST /api/events/listen (batch: 10-50 events) → Navidrome
         → ListenEventRepository.Insert(events)
         → SQLite (listen_event table, append-only)
         → [Optional] Publish to message queue (NATS/Redis)
```

**2. Analytics Processing (Asynchronous, Batch)**
```
Analytics Service:
  Option A: Read from message queue (real-time stream)
  Option B: Periodically read new events from listen_event table (batch)
  
  → Compute derived metrics (rollups, aggregates)
  → Store in TimescaleDB/ClickHouse
  → Expose GraphQL API for queries
```

**3. Social Interactions (Moderate Frequency)**
```
Client → Social Service API directly (follows, likes, comments)
       → Social Service validates JWT (shared secret with Navidrome)
       → Write to PostgreSQL (social graph, feed_events)
       → [Optional] Listen to Navidrome SSE for catalog events (new playlists, etc.)
```

**4. Streaming (Unchanged)**
```
Client → GET /rest/stream → Navidrome → MediaStreamer → FFmpeg (if transcoding) → Stream
```

---

## Navidrome Modifications (Minimal Fork)

### 1. New Database Table: `listen_event`

```sql
CREATE TABLE listen_event (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         VARCHAR(255) NOT NULL REFERENCES user(id) ON DELETE CASCADE,
    player_id       VARCHAR(255) NOT NULL REFERENCES player(id) ON DELETE CASCADE,
    media_file_id   VARCHAR(255) NOT NULL REFERENCES media_file(id) ON DELETE CASCADE,
    event_type      VARCHAR(50)  NOT NULL,  -- 'play', 'pause', 'resume', 'seek', 'ended', 'error'
    timestamp       INTEGER      NOT NULL,  -- Unix timestamp (milliseconds)
    position        INTEGER      NOT NULL,  -- Playback position (milliseconds)
    metadata        TEXT,                   -- JSON: {session_id, client_version, incognito, ...}
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_listen_event_user_ts ON listen_event(user_id, timestamp);
CREATE INDEX idx_listen_event_media_file ON listen_event(media_file_id);
CREATE INDEX idx_listen_event_timestamp ON listen_event(timestamp);
```

**Size Estimate:**
- ~100 bytes per event
- 1000 events/day/user → 100 KB/day/user
- 100 users → 10 MB/day → 3.6 GB/year
- **Manageable for SQLite**, but plan for archival/purge policy

### 2. New Repository Interface: `ListenEventRepository`

```go
// model/listen_event.go
package model

import "time"

type ListenEvent struct {
    ID           int64     `json:"id"`
    UserID       string    `json:"userId"`
    PlayerID     string    `json:"playerId"`
    MediaFileID  string    `json:"mediaFileId"`
    EventType    string    `json:"eventType"`  // play, pause, resume, seek, ended, error
    Timestamp    time.Time `json:"timestamp"`
    Position     int       `json:"position"`  // milliseconds
    Metadata     string    `json:"metadata"` // JSON-encoded
    CreatedAt    time.Time `json:"createdAt"`
}

type ListenEventRepository interface {
    Insert(events []ListenEvent) error
    GetByUser(userID string, since time.Time, limit int) ([]ListenEvent, error)
    DeleteOlderThan(timestamp time.Time) (int64, error)  // For purging old events
}
```

**Implementation:** `persistence/listen_event_repository.go` (~100 LoC)

### 3. New API Endpoint: `POST /api/events/listen`

```go
// server/nativeapi/listen_events.go
package nativeapi

func (api *Router) RecordListenEvents(w http.ResponseWriter, r *http.Request) {
    var events []model.ListenEvent
    if err := json.NewDecoder(r.Body).Decode(&events); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    // Inject user/player from context
    user, _ := request.UserFrom(r.Context())
    player, _ := request.PlayerFrom(r.Context())
    
    for i := range events {
        events[i].UserID = user.ID
        events[i].PlayerID = player.ID
    }
    
    // Check incognito mode (from metadata or user prefs)
    if isIncognito(user, player, events) {
        log.Debug(r.Context(), "Ignoring events in incognito mode", "user", user.UserName)
        w.WriteHeader(http.StatusNoContent)
        return
    }
    
    if err := api.ds.ListenEvent(r.Context()).Insert(events); err != nil {
        log.Error(r.Context(), "Failed to insert listen events", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    // [Optional] Emit to message queue for real-time processing
    api.eventEmitter.Emit(events)
    
    w.WriteHeader(http.StatusCreated)
}
```

**Integration:** Add route in `nativeapi/native_api.go`:
```go
r.Post("/events/listen", api.RecordListenEvents)
```

### 4. Optional: Message Queue Emitter

If we want real-time analytics, add an event emitter:

```go
// server/events/queue_emitter.go
package events

type QueueEmitter interface {
    Emit(events []model.ListenEvent) error
}

// Implementation could use NATS, Redis Streams, Kafka, etc.
type natsEmitter struct {
    nc *nats.Conn
}

func (e *natsEmitter) Emit(events []model.ListenEvent) error {
    data, _ := json.Marshal(events)
    return e.nc.Publish("navidrome.listen_events", data)
}
```

**Configuration:** Add to `conf/configuration.go`:
```go
type configOptions struct {
    // ...
    MessageQueue struct {
        Enabled bool
        URL     string  // nats://localhost:4222
        Subject string  // navidrome.listen_events
    }
}
```

### 5. Privacy: Incognito Mode

Add to `user_props` table:
```
Key: "incognito_mode"
Value: "true" or "false" (per user)
```

Add to `player` table (or `user_props` keyed by player_id):
```
Key: "tracking_enabled"
Value: "true" or "false" (per device)
```

Client sends in metadata field of events:
```json
{
    "metadata": "{\"incognito\": true, \"session_id\": \"abc123\"}"
}
```

Server checks before inserting:
```go
func isIncognito(user *model.User, player *model.Player, events []model.ListenEvent) bool {
    // Check user-level preference
    if userProps, _ := api.ds.UserProps(ctx).Get(user.ID, "incognito_mode"); userProps == "true" {
        return true
    }
    
    // Check player-level preference
    if playerProps, _ := api.ds.UserProps(ctx).Get(user.ID, "tracking_enabled_"+player.ID); playerProps == "false" {
        return true
    }
    
    // Check event-level metadata
    for _, event := range events {
        var meta map[string]interface{}
        json.Unmarshal([]byte(event.Metadata), &meta)
        if meta["incognito"] == true {
            return true
        }
    }
    
    return false
}
```

### Summary of Navidrome Changes

**Files to Add:**
- `model/listen_event.go` (~50 LoC)
- `persistence/listen_event_repository.go` (~150 LoC)
- `server/nativeapi/listen_events.go` (~100 LoC)
- `db/migrations/20260101000000_create_listen_event_table.go` (~50 LoC)
- [Optional] `server/events/queue_emitter.go` (~50 LoC)

**Files to Modify:**
- `model/datastore.go` - Add `ListenEvent(ctx) ListenEventRepository`
- `persistence/persistence.go` - Add `ListenEvent(ctx) model.ListenEventRepository`
- `server/nativeapi/native_api.go` - Add route for `/events/listen`
- `cmd/wire_injectors.go` - Add QueueEmitter dependency (if using message queue)

**Total LoC Added:** ~500-800 (depending on message queue integration)

**Test Coverage:** Add unit tests (~300 LoC)

---

## External Services

### Analytics Service

**Responsibilities:**
- Consume listen events (from message queue or DB polling)
- Compute derived metrics:
  - **Completion %:** Sum of listened intervals / track duration
  - **Skip cliffs:** Frequent pause/skip at specific timestamps
  - **Replay hotspots:** Multiple plays at same timestamp range
  - **Session stats:** Genre diversity, skip rate, avg session length
  - **Taste vectors:** ML embeddings for recommendations
- Store aggregates in time-series DB (TimescaleDB, ClickHouse)
- Expose query API (GraphQL, REST)

**Technology Stack:**
- **Language:** Go (for performance) or Python (for ML libraries)
- **Database:** TimescaleDB (PostgreSQL extension for time-series)
- **Message Queue:** NATS (lightweight) or Redis Streams
- **API:** GraphQL (Hasura or custom with gqlgen)

**Deployment:**
- Docker container
- Reads from Navidrome DB (read-only replica for large deployments)
- Scales horizontally (stateless workers)

### Social Service

**Responsibilities:**
- Social graph (follows, followers)
- Activity feed (personalized, aggregated)
- Likes (on tracks, albums, playlists)
- Comments (on tracks, albums, with timestamps for "moment reactions")
- Privacy controls (per-item visibility: private, friends, public)

**Technology Stack:**
- **Language:** Go, Node.js, or Python
- **Database:** PostgreSQL (relational for social graph)
- **Cache:** Redis (for feed generation)
- **API:** GraphQL (for flexible queries) or REST

**Data Model:**
```sql
-- Social graph
CREATE TABLE follows (
    follower_id VARCHAR(255) NOT NULL REFERENCES user(id),
    following_id VARCHAR(255) NOT NULL REFERENCES user(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (follower_id, following_id)
);

-- Likes
CREATE TABLE likes (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL REFERENCES user(id),
    item_type VARCHAR(50) NOT NULL,  -- 'track', 'album', 'playlist'
    item_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, item_type, item_id)
);

-- Comments
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL REFERENCES user(id),
    item_type VARCHAR(50) NOT NULL,  -- 'track', 'album', 'playlist'
    item_id VARCHAR(255) NOT NULL,
    timestamp_ms INTEGER,  -- For "moment reactions" on tracks (optional)
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Activity feed (denormalized)
CREATE TABLE feed_events (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,  -- Actor (who did the action)
    event_type VARCHAR(50) NOT NULL,  -- 'listened', 'liked', 'commented', 'created_playlist'
    item_type VARCHAR(50),
    item_id VARCHAR(255),
    metadata JSONB,  -- Event-specific data (track title, album art URL, etc.)
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_feed_events_user_ts ON feed_events(user_id, created_at DESC);
```

**Feed Generation:**
1. When user performs action (like, comment, playlist create):
   - Insert into `feed_events` table
2. When user requests feed:
   - Query `feed_events` WHERE user_id IN (users I follow) ORDER BY created_at DESC LIMIT 50
   - Cache in Redis (TTL 5 minutes)
3. For high-activity users, pre-compute feeds asynchronously

**Deployment:**
- Docker container
- Horizontal scaling (stateless API servers)
- PostgreSQL primary (for writes) + read replicas (for feed queries)

---

## Client Architecture

### Requirements
- Emit detailed playback events (play, pause, resume, seek, ended)
- Buffer events locally (reduce network requests)
- Send events in batches (every 10s or 50 events, whichever comes first)
- Respect incognito mode (client-side flag, no events sent)
- Derive "listened intervals" from events
- Authenticate with Navidrome JWT

### Event Schema (Client-Side)

```typescript
interface ListenEvent {
    eventType: 'play' | 'pause' | 'resume' | 'seek' | 'ended' | 'error';
    timestamp: number;  // Unix timestamp (ms)
    position: number;   // Playback position (ms)
    metadata?: {
        sessionId: string;
        clientVersion: string;
        incognito: boolean;
        previousPosition?: number;  // For seek events
    };
}
```

### Event Buffer (Client-Side)

```typescript
class EventBuffer {
    private events: ListenEvent[] = [];
    private flushInterval = 10000;  // 10 seconds
    private maxBatchSize = 50;
    
    constructor(private api: NavidromeAPI) {
        setInterval(() => this.flush(), this.flushInterval);
    }
    
    push(event: ListenEvent) {
        if (this.isIncognitoMode()) {
            return;  // Don't record events in incognito
        }
        
        this.events.push(event);
        
        if (this.events.length >= this.maxBatchSize) {
            this.flush();
        }
    }
    
    async flush() {
        if (this.events.length === 0) return;
        
        const batch = this.events.splice(0, this.maxBatchSize);
        
        try {
            await this.api.post('/api/events/listen', batch);
        } catch (error) {
            // On error, re-queue events (with limit to prevent memory leak)
            this.events.unshift(...batch.slice(0, 100));
        }
    }
    
    private isIncognitoMode(): boolean {
        return localStorage.getItem('incognitoMode') === 'true';
    }
}
```

### Player Integration

```typescript
class TelemetryPlayer extends NavidromePlayer {
    private eventBuffer = new EventBuffer(this.api);
    private currentSession: string;
    
    play(track: Track, position: number = 0) {
        super.play(track, position);
        
        this.eventBuffer.push({
            eventType: 'play',
            timestamp: Date.now(),
            position: position,
            metadata: {
                sessionId: this.currentSession,
                clientVersion: APP_VERSION,
                incognito: this.isIncognitoMode()
            }
        });
    }
    
    pause() {
        super.pause();
        
        this.eventBuffer.push({
            eventType: 'pause',
            timestamp: Date.now(),
            position: this.getCurrentPosition()
        });
    }
    
    seek(position: number) {
        const previousPosition = this.getCurrentPosition();
        super.seek(position);
        
        this.eventBuffer.push({
            eventType: 'seek',
            timestamp: Date.now(),
            position: position,
            metadata: {
                sessionId: this.currentSession,
                previousPosition: previousPosition
            }
        });
    }
    
    // ... similar for resume, ended, error
}
```

### Listen Interval Derivation (Server-Side or Client-Side)

**Option A: Server-Side (in Analytics Service)**
```python
def compute_listened_intervals(events: List[ListenEvent]) -> List[Tuple[int, int]]:
    """
    Compute listened intervals from play/pause/seek events.
    Returns list of (start_ms, end_ms) tuples.
    """
    intervals = []
    play_start = None
    
    for event in sorted(events, key=lambda e: e.timestamp):
        if event.event_type == 'play':
            play_start = event.position
        elif event.event_type in ['pause', 'seek', 'ended']:
            if play_start is not None:
                intervals.append((play_start, event.position))
                play_start = None
        elif event.event_type == 'resume':
            play_start = event.position
    
    return merge_overlapping_intervals(intervals)

def compute_completion_percentage(intervals: List[Tuple[int, int]], duration_ms: int) -> float:
    """
    Compute % of track listened.
    """
    total_listened = sum(end - start for start, end in intervals)
    return min(100, (total_listened / duration_ms) * 100)
```

**Option B: Client-Side (before sending)**
- Client tracks "current play segment" in memory
- On pause/seek/ended, close current segment and append to batch
- Send already-computed intervals to server

**Recommendation:** Server-side derivation (more flexible, can reprocess if algorithm changes)

---

## Deployment Strategy

### Phase 1: Navidrome Only (Weeks 1-2)
- Deploy modified Navidrome with listen_event table
- Client emits events, stored in SQLite
- No external services yet
- Users can query their own events via API (basic stats)

### Phase 2: Add Analytics Service (Weeks 3-4)
- Deploy Analytics Service (reads from Navidrome DB)
- Compute derived metrics (completion %, skip cliffs)
- Expose GraphQL API
- Update client to show analytics dashboards

### Phase 3: Add Social Service (Weeks 5-6)
- Deploy Social Service (independent DB)
- Add social UI components (follow, like, comment)
- Integrate with Navidrome SSE for catalog updates

### Phase 4: Scale Out (Weeks 7+)
- Add message queue (NATS) between Navidrome and Analytics
- Add read replicas for Navidrome DB (if needed)
- Add Redis cache for social feeds
- Implement purge policies for old events

---

## Risk Mitigation

### Risk: Fork Maintenance Burden
**Mitigation:**
- Keep changes minimal and well-documented
- Use feature flags (`conf.Server.EnableTelemetry`) to disable new code
- Contribute generic changes upstream (e.g., ListenEventRepository without social-specific logic)
- Automate rebase with CI (test against upstream commits)

### Risk: SQLite Write Throughput
**Mitigation:**
- Batch inserts (50-100 events at a time)
- Use `WithTxImmediate()` for bulk writes
- Monitor DB size and write latency
- Plan for migration to PostgreSQL if throughput becomes issue (Navidrome already supports Postgres)

### Risk: Privacy/GDPR Compliance
**Mitigation:**
- Explicit opt-in for telemetry (default: enabled, but clear disclosure)
- Incognito mode (client-side and server-side enforcement)
- User data export (JSON dump of all listen_events)
- User data deletion (CASCADE DELETE with audit log)
- Data retention policy (purge events older than 1 year)

### Risk: Client Spoofing (Fake Events)
**Mitigation:**
- Rate limiting on `/api/events/listen` endpoint (max 1000 events/hour/user)
- Server-side validation (position <= track duration, timestamps monotonic)
- Anomaly detection in Analytics Service (flag suspicious patterns)
- For critical metrics (leaderboards), use server-side scrobbles only

### Risk: External Service Latency
**Mitigation:**
- Clients don't wait for analytics/social services (fire-and-forget)
- Use CDN/edge caching for social feeds
- Implement circuit breakers (fail gracefully if service is down)

---

## Alternative Considered: Full Overlay (Pattern B)

We considered keeping Navidrome completely unchanged and building everything externally. This was rejected because:

1. **Latency:** Even with WebSockets, external ingestion adds 10-50ms per event. With 100 events/session, that's 1-5 seconds of overhead.

2. **Auth Complexity:** External service must validate Navidrome JWTs. This requires:
   - Shared JWT secret (security risk if external service is compromised)
   - OR: Public key validation (requires Navidrome to expose public key endpoint)
   - OR: Proxy authentication (Navidrome validates, forwards to external service)

3. **Data Duplication:** External service needs user/track/album data for queries. Either:
   - Duplicate data (eventual consistency issues)
   - OR: Make API calls to Navidrome for every query (latency, load)

4. **Operational Complexity:** Three separate services (Navidrome, Analytics, Social) from day one. Deployment is harder, monitoring is more complex.

**Hybrid pattern solves these issues** while keeping most benefits of overlay architecture.

---

## Conclusion

The **Hybrid (Pattern C)** approach strikes the optimal balance for this project:
- **Fast:** In-process event ingestion (no network overhead)
- **Decoupled:** Heavy analytics and social features in separate services
- **Maintainable:** Minimal Navidrome changes (~500 LoC), easy to rebase
- **Flexible:** Can deploy incrementally (start with just Navidrome, add services later)
- **Scalable:** Analytics and social services scale independently

This decision enables us to:
1. **Leverage Navidrome's strengths** (catalog, streaming, auth)
2. **Add deep telemetry** without sacrificing performance
3. **Build social features** with appropriate technology (PostgreSQL, Redis)
4. **Scale services** based on actual load (not prematurely optimizing)
5. **Maintain upstream compatibility** (small, isolated fork)

Next step: **Phased Implementation Plan** (see separate document).

---

**End of Integration Pattern Decision**
