# Phase 2: Telemetry v1 Foundations
**Duration:** 3-4 weeks  
**Team Size:** 4-5 engineers  
**Dependencies:** Phase 1 complete

---

## Objective

Build the **core telemetry infrastructure** to capture detailed playback events:
1. Add `listen_event` table to Navidrome database
2. Implement `ListenEventRepository` for event storage
3. Create `/api/events/listen` endpoint for bulk event ingestion
4. Add privacy controls (incognito mode, per-device tracking opt-out)
5. Implement event validation and rate limiting
6. Build custom web player that emits detailed events
7. Deploy and validate telemetry pipeline (ingestion → storage)

**No analytics computation yet** - this phase only captures raw events. Phase 5 will add derived metrics.

---

## Scope

### In Scope
- ✅ `listen_event` table (schema, migration, indexes)
- ✅ `ListenEventRepository` interface and SQL implementation
- ✅ `POST /api/events/listen` endpoint (bulk ingestion)
- ✅ Event validation (schema, rate limits, timestamp checks)
- ✅ Privacy controls (incognito mode, device opt-out, user preferences)
- ✅ Custom web player (fork `navidrome-music-player`, add event emission)
- ✅ Event buffering (client-side queue, batch sends)
- ✅ Basic telemetry dashboard (show raw events for debugging)

### Out of Scope
- ❌ Derived metrics (completion %, skip cliffs) - covered in Phase 5
- ❌ Analytics service (external service) - covered in Phase 5
- ❌ Social features - covered in Phase 3-4
- ❌ Recommendations - covered in Phase 6
- ❌ Mobile client telemetry - covered in later phase (web first)

---

## Key Technical Work Items

### 1. Database Schema: `listen_event` Table

**Goal:** Create table to store raw playback events with efficient querying.

**Schema Design:**

```sql
CREATE TABLE listen_event (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         VARCHAR(255) NOT NULL,
    player_id       VARCHAR(255) NOT NULL,
    media_file_id   VARCHAR(255) NOT NULL,
    event_type      VARCHAR(50)  NOT NULL,  -- 'play', 'pause', 'resume', 'seek', 'ended', 'error'
    timestamp       INTEGER      NOT NULL,  -- Unix timestamp (milliseconds)
    position        INTEGER      NOT NULL,  -- Playback position (milliseconds)
    metadata        TEXT,                   -- JSON: {session_id, client_version, incognito, previous_position, ...}
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE,
    FOREIGN KEY (player_id) REFERENCES player(id) ON DELETE CASCADE,
    FOREIGN KEY (media_file_id) REFERENCES media_file(id) ON DELETE CASCADE
);

-- Index for user's history queries (most common)
CREATE INDEX idx_listen_event_user_ts 
    ON listen_event(user_id, timestamp DESC);

-- Index for track-level analytics (which tracks have most plays/skips)
CREATE INDEX idx_listen_event_media_file 
    ON listen_event(media_file_id, timestamp DESC);

-- Index for time-range queries (recent events, daily rollups)
CREATE INDEX idx_listen_event_timestamp 
    ON listen_event(timestamp DESC);

-- Index for session-based queries (analyze listening sessions)
CREATE INDEX idx_listen_event_user_player 
    ON listen_event(user_id, player_id, timestamp DESC);
```

**Design Decisions:**
- **AUTOINCREMENT ID:** Simple, monotonically increasing (useful for pagination)
- **Timestamp in milliseconds:** Higher precision for accurate interval calculation
- **Position in milliseconds:** Match HTML5 Audio API units (easier for client)
- **Metadata as JSON TEXT:** Flexible for future extensions (no schema changes needed)
- **Foreign keys with CASCADE DELETE:** Automatic cleanup when user/player/track deleted
- **Indexes optimized for queries:**
  - User history (dashboard, export)
  - Track analytics (heatmaps, skip analysis)
  - Time-range analytics (daily/weekly rollups)
  - Session analysis (detect binge sessions, skip patterns)

**Size Estimation:**
- ~150 bytes per event (with typical metadata)
- Assumptions:
  - 100 users
  - 10 tracks/day/user (average)
  - 5 events/track (play, pause, resume, pause, ended)
  - Total: 100 * 10 * 5 = 5,000 events/day
- Daily storage: 5,000 * 150 bytes = 750 KB/day = 23 MB/month = 274 MB/year
- **For 1,000 users:** 2.74 GB/year (manageable for SQLite)
- **For 10,000 users:** 27.4 GB/year (approaching SQLite limits, plan for archival)

**Migration File:** `db/migrations/20260110000001_create_listen_event_table.go`

```go
package migrations

import (
    "context"
    "database/sql"
    "github.com/pressly/goose/v3"
)

func init() {
    goose.AddMigrationContext(upCreateListenEventTable, downCreateListenEventTable)
}

func upCreateListenEventTable(_ context.Context, tx *sql.Tx) error {
    _, err := tx.Exec(`
        CREATE TABLE listen_event (
            id              INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id         VARCHAR(255) NOT NULL,
            player_id       VARCHAR(255) NOT NULL,
            media_file_id   VARCHAR(255) NOT NULL,
            event_type      VARCHAR(50)  NOT NULL,
            timestamp       INTEGER      NOT NULL,
            position        INTEGER      NOT NULL,
            metadata        TEXT,
            created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
            
            FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE,
            FOREIGN KEY (player_id) REFERENCES player(id) ON DELETE CASCADE,
            FOREIGN KEY (media_file_id) REFERENCES media_file(id) ON DELETE CASCADE
        );
        
        CREATE INDEX idx_listen_event_user_ts 
            ON listen_event(user_id, timestamp DESC);
        CREATE INDEX idx_listen_event_media_file 
            ON listen_event(media_file_id, timestamp DESC);
        CREATE INDEX idx_listen_event_timestamp 
            ON listen_event(timestamp DESC);
        CREATE INDEX idx_listen_event_user_player 
            ON listen_event(user_id, player_id, timestamp DESC);
    `)
    return err
}

func downCreateListenEventTable(_ context.Context, tx *sql.Tx) error {
    _, err := tx.Exec(`DROP TABLE IF EXISTS listen_event;`)
    return err
}
```

**Tasks:**
- [ ] Create migration file
- [ ] Test migration up/down locally
- [ ] Test migration on copy of production DB (if available)
- [ ] Document schema in `DATABASE_SCHEMA.md`
- [ ] Add to `model/datastore.go` interface

**Deliverable:** Migration file + updated schema documentation

**Acceptance Criteria:**
- Migration runs successfully on fresh DB
- Migration can be rolled back cleanly
- Indexes are created and improve query performance (verified with EXPLAIN QUERY PLAN)

---

### 2. Domain Model & Repository

**Goal:** Add Go structs and repository interface for `listen_event`.

**Domain Model:** `model/listen_event.go`

```go
package model

import (
    "encoding/json"
    "time"
)

// ListenEvent represents a single playback event
type ListenEvent struct {
    ID           int64             `structs:"id" json:"id"`
    UserID       string            `structs:"user_id" json:"userId"`
    PlayerID     string            `structs:"player_id" json:"playerId"`
    MediaFileID  string            `structs:"media_file_id" json:"mediaFileId"`
    EventType    ListenEventType   `structs:"event_type" json:"eventType"`
    Timestamp    time.Time         `structs:"timestamp" json:"timestamp"`
    Position     int               `structs:"position" json:"position"`  // milliseconds
    Metadata     ListenEventMeta   `structs:"metadata" json:"metadata"`
    CreatedAt    time.Time         `structs:"created_at" json:"createdAt"`
}

// ListenEventType is an enum for event types
type ListenEventType string

const (
    EventTypePlay   ListenEventType = "play"
    EventTypePause  ListenEventType = "pause"
    EventTypeResume ListenEventType = "resume"
    EventTypeSeek   ListenEventType = "seek"
    EventTypeEnded  ListenEventType = "ended"
    EventTypeError  ListenEventType = "error"
)

// Valid returns true if the event type is valid
func (t ListenEventType) Valid() bool {
    switch t {
    case EventTypePlay, EventTypePause, EventTypeResume, EventTypeSeek, EventTypeEnded, EventTypeError:
        return true
    }
    return false
}

// ListenEventMeta contains optional metadata for events
type ListenEventMeta struct {
    SessionID        string `json:"sessionId,omitempty"`
    ClientVersion    string `json:"clientVersion,omitempty"`
    Incognito        bool   `json:"incognito,omitempty"`
    PreviousPosition int    `json:"previousPosition,omitempty"`  // For seek events
    ErrorMessage     string `json:"errorMessage,omitempty"`      // For error events
}

// MarshalDB converts metadata to JSON string for DB storage
func (m ListenEventMeta) MarshalDB() string {
    bytes, _ := json.Marshal(m)
    return string(bytes)
}

// UnmarshalDB parses JSON string from DB
func (m *ListenEventMeta) UnmarshalDB(data string) error {
    return json.Unmarshal([]byte(data), m)
}

type ListenEvents []ListenEvent

// ListenEventRepository defines the interface for listen event storage
type ListenEventRepository interface {
    // Insert bulk inserts events (optimized for batch writes)
    Insert(events []ListenEvent) error
    
    // GetByUser returns events for a user, optionally filtered by time range
    GetByUser(userID string, since time.Time, until time.Time, limit int) (ListenEvents, error)
    
    // GetByMediaFile returns events for a specific track (for heatmaps)
    GetByMediaFile(mediaFileID string, since time.Time, limit int) (ListenEvents, error)
    
    // GetBySession returns events for a specific session (for session analysis)
    GetBySession(userID string, playerID string, sessionID string) (ListenEvents, error)
    
    // DeleteOlderThan purges old events (for data retention policy)
    DeleteOlderThan(timestamp time.Time) (int64, error)
    
    // Count returns total number of events (for stats)
    Count(userID string, since time.Time) (int64, error)
}
```

**Repository Implementation:** `persistence/listen_event_repository.go`

```go
package persistence

import (
    "context"
    "time"
    
    . "github.com/Masterminds/squirrel"
    "github.com/navidrome/navidrome/log"
    "github.com/navidrome/navidrome/model"
    "github.com/pocketbase/dbx"
)

type listenEventRepository struct {
    sqlRepository
}

func NewListenEventRepository(ctx context.Context, db dbx.Builder) model.ListenEventRepository {
    r := &listenEventRepository{}
    r.ctx = ctx
    r.db = db
    r.tableName = "listen_event"
    return r
}

func (r *listenEventRepository) Insert(events []model.ListenEvent) error {
    if len(events) == 0 {
        return nil
    }
    
    // Bulk insert using transaction for performance
    return r.executeWithTransaction(func(tx dbx.Builder) error {
        for _, event := range events {
            values := map[string]interface{}{
                "user_id":       event.UserID,
                "player_id":     event.PlayerID,
                "media_file_id": event.MediaFileID,
                "event_type":    string(event.EventType),
                "timestamp":     event.Timestamp.UnixMilli(),
                "position":      event.Position,
                "metadata":      event.Metadata.MarshalDB(),
                "created_at":    time.Now(),
            }
            
            insert := Insert(r.tableName).SetMap(values)
            if _, err := tx.NewQuery(insert.ToSql()).Execute(); err != nil {
                return err
            }
        }
        return nil
    })
}

func (r *listenEventRepository) GetByUser(userID string, since time.Time, until time.Time, limit int) (model.ListenEvents, error) {
    query := Select("*").
        From(r.tableName).
        Where(Eq{"user_id": userID}).
        OrderBy("timestamp DESC")
    
    if !since.IsZero() {
        query = query.Where(GtOrEq{"timestamp": since.UnixMilli()})
    }
    if !until.IsZero() {
        query = query.Where(LtOrEq{"timestamp": until.UnixMilli()})
    }
    if limit > 0 {
        query = query.Limit(uint64(limit))
    }
    
    var events model.ListenEvents
    err := r.queryAll(query, &events)
    return events, err
}

func (r *listenEventRepository) GetByMediaFile(mediaFileID string, since time.Time, limit int) (model.ListenEvents, error) {
    query := Select("*").
        From(r.tableName).
        Where(Eq{"media_file_id": mediaFileID}).
        OrderBy("timestamp DESC")
    
    if !since.IsZero() {
        query = query.Where(GtOrEq{"timestamp": since.UnixMilli()})
    }
    if limit > 0 {
        query = query.Limit(uint64(limit))
    }
    
    var events model.ListenEvents
    err := r.queryAll(query, &events)
    return events, err
}

func (r *listenEventRepository) GetBySession(userID string, playerID string, sessionID string) (model.ListenEvents, error) {
    // Query by user_id and player_id, then filter by sessionID in metadata (JSON query)
    // Note: SQLite JSON support is limited, so we fetch and filter in Go
    query := Select("*").
        From(r.tableName).
        Where(Eq{"user_id": userID, "player_id": playerID}).
        OrderBy("timestamp ASC")
    
    var allEvents model.ListenEvents
    err := r.queryAll(query, &allEvents)
    if err != nil {
        return nil, err
    }
    
    // Filter by sessionID (parsed from metadata JSON)
    var sessionEvents model.ListenEvents
    for _, event := range allEvents {
        if event.Metadata.SessionID == sessionID {
            sessionEvents = append(sessionEvents, event)
        }
    }
    
    return sessionEvents, nil
}

func (r *listenEventRepository) DeleteOlderThan(timestamp time.Time) (int64, error) {
    del := Delete(r.tableName).Where(Lt{"timestamp": timestamp.UnixMilli()})
    count, err := r.executeSQL(del)
    if err != nil {
        return 0, err
    }
    if count > 0 {
        log.Info(r.ctx, "Purged old listen events", "count", count, "before", timestamp)
    }
    return count, nil
}

func (r *listenEventRepository) Count(userID string, since time.Time) (int64, error) {
    query := Select("COUNT(*)").
        From(r.tableName)
    
    if userID != "" {
        query = query.Where(Eq{"user_id": userID})
    }
    if !since.IsZero() {
        query = query.Where(GtOrEq{"timestamp": since.UnixMilli()})
    }
    
    var count int64
    err := r.queryOne(query, &count)
    return count, err
}
```

**Add to DataStore:** `model/datastore.go`

```go
type DataStore interface {
    // ... existing methods
    ListenEvent(ctx context.Context) ListenEventRepository
}
```

**Add to Persistence:** `persistence/persistence.go`

```go
func (d *dataStore) ListenEvent(ctx context.Context) model.ListenEventRepository {
    return NewListenEventRepository(ctx, d.db)
}
```

**Tasks:**
- [ ] Create `model/listen_event.go`
- [ ] Create `persistence/listen_event_repository.go`
- [ ] Update `model/datastore.go` and `persistence/persistence.go`
- [ ] Write unit tests for repository (insert, query, delete)
- [ ] Test with large batches (1000 events)

**Deliverable:** Domain model + repository + tests

**Acceptance Criteria:**
- Repository methods are tested (>80% coverage)
- Bulk insert handles 1000 events in <100ms
- Queries use indexes (verified with EXPLAIN QUERY PLAN)

---

### 3. API Endpoint: `POST /api/events/listen`

**Goal:** Create endpoint for clients to submit events in bulk.

**Endpoint Design:**

```
POST /api/events/listen
Authorization: Bearer <JWT>
Content-Type: application/json

Body:
[
  {
    "mediaFileId": "track-123",
    "eventType": "play",
    "timestamp": 1704067200000,
    "position": 0,
    "metadata": {
      "sessionId": "session-abc",
      "clientVersion": "1.0.0",
      "incognito": false
    }
  },
  {
    "mediaFileId": "track-123",
    "eventType": "pause",
    "timestamp": 1704067230000,
    "position": 30000,
    "metadata": {
      "sessionId": "session-abc",
      "clientVersion": "1.0.0"
    }
  }
]

Response (201 Created):
{
  "accepted": 2,
  "rejected": 0
}

Response (400 Bad Request):
{
  "error": "Invalid event: eventType must be one of [play, pause, resume, seek, ended, error]",
  "rejected": 1
}

Response (429 Too Many Requests):
{
  "error": "Rate limit exceeded",
  "retryAfter": 60
}
```

**Implementation:** `server/nativeapi/listen_events.go`

```go
package nativeapi

import (
    "encoding/json"
    "net/http"
    "time"
    
    "github.com/navidrome/navidrome/conf"
    "github.com/navidrome/navidrome/log"
    "github.com/navidrome/navidrome/model"
    "github.com/navidrome/navidrome/model/request"
)

const (
    maxEventsPerRequest = 100
    maxRequestsPerMinute = 60  // 60 requests/minute = 6000 events/minute
)

type listenEventRequest struct {
    MediaFileID string                 `json:"mediaFileId"`
    EventType   string                 `json:"eventType"`
    Timestamp   int64                  `json:"timestamp"`  // Unix milliseconds
    Position    int                    `json:"position"`   // milliseconds
    Metadata    map[string]interface{} `json:"metadata"`
}

type listenEventResponse struct {
    Accepted int    `json:"accepted"`
    Rejected int    `json:"rejected"`
    Error    string `json:"error,omitempty"`
}

func (api *Router) RecordListenEvents(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    user, _ := request.UserFrom(ctx)
    player, _ := request.PlayerFrom(ctx)
    
    // Parse request body
    var requests []listenEventRequest
    if err := json.NewDecoder(r.Body).Decode(&requests); err != nil {
        log.Error(ctx, "Invalid JSON in listen events request", err)
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    // Validate batch size
    if len(requests) == 0 {
        w.WriteHeader(http.StatusNoContent)
        return
    }
    if len(requests) > maxEventsPerRequest {
        log.Warn(ctx, "Too many events in single request", "count", len(requests), "max", maxEventsPerRequest)
        http.Error(w, "Too many events (max 100)", http.StatusBadRequest)
        return
    }
    
    // Check if telemetry is enabled (global and per-user)
    if !conf.Server.EnableTelemetry {
        log.Debug(ctx, "Telemetry is disabled globally")
        w.WriteHeader(http.StatusNoContent)
        return
    }
    if api.isIncognitoMode(ctx, user, player) {
        log.Debug(ctx, "Telemetry disabled for user/player", "user", user.UserName, "player", player.Name)
        w.WriteHeader(http.StatusNoContent)
        return
    }
    
    // Validate and convert events
    var events []model.ListenEvent
    var rejected int
    for _, req := range requests {
        event, err := api.validateEvent(req, user.ID, player.ID)
        if err != nil {
            log.Warn(ctx, "Invalid event", "error", err, "event", req)
            rejected++
            continue
        }
        events = append(events, event)
    }
    
    // Insert events
    if len(events) > 0 {
        if err := api.ds.ListenEvent(ctx).Insert(events); err != nil {
            log.Error(ctx, "Failed to insert listen events", err)
            http.Error(w, "Internal server error", http.StatusInternalServerError)
            return
        }
    }
    
    // Return response
    resp := listenEventResponse{
        Accepted: len(events),
        Rejected: rejected,
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteStatus(http.StatusCreated)
    json.NewEncoder(w).Encode(resp)
}

func (api *Router) validateEvent(req listenEventRequest, userID string, playerID string) (model.ListenEvent, error) {
    // Validate event type
    eventType := model.ListenEventType(req.EventType)
    if !eventType.Valid() {
        return model.ListenEvent{}, fmt.Errorf("invalid event type: %s", req.EventType)
    }
    
    // Validate timestamp (not in future, not too old)
    now := time.Now().UnixMilli()
    if req.Timestamp > now + 60000 {  // Allow 1 minute clock skew
        return model.ListenEvent{}, fmt.Errorf("timestamp in future: %d", req.Timestamp)
    }
    if req.Timestamp < now - 86400000*7 {  // Reject events older than 7 days
        return model.ListenEvent{}, fmt.Errorf("timestamp too old: %d", req.Timestamp)
    }
    
    // Validate position (non-negative)
    if req.Position < 0 {
        return model.ListenEvent{}, fmt.Errorf("negative position: %d", req.Position)
    }
    
    // Validate media file exists
    _, err := api.ds.MediaFile(ctx).Get(req.MediaFileID)
    if err != nil {
        return model.ListenEvent{}, fmt.Errorf("media file not found: %s", req.MediaFileID)
    }
    
    // Convert metadata
    meta := model.ListenEventMeta{}
    if sessionID, ok := req.Metadata["sessionId"].(string); ok {
        meta.SessionID = sessionID
    }
    if clientVersion, ok := req.Metadata["clientVersion"].(string); ok {
        meta.ClientVersion = clientVersion
    }
    if incognito, ok := req.Metadata["incognito"].(bool); ok {
        meta.Incognito = incognito
    }
    if prevPos, ok := req.Metadata["previousPosition"].(float64); ok {
        meta.PreviousPosition = int(prevPos)
    }
    if errMsg, ok := req.Metadata["errorMessage"].(string); ok {
        meta.ErrorMessage = errMsg
    }
    
    return model.ListenEvent{
        UserID:      userID,
        PlayerID:    playerID,
        MediaFileID: req.MediaFileID,
        EventType:   eventType,
        Timestamp:   time.UnixMilli(req.Timestamp),
        Position:    req.Position,
        Metadata:    meta,
    }, nil
}

func (api *Router) isIncognitoMode(ctx context.Context, user *model.User, player *model.Player) bool {
    // Check user-level preference
    userProps, err := api.ds.UserProps(ctx).Get(user.ID, "incognito_mode")
    if err == nil && userProps == "true" {
        return true
    }
    
    // Check player-level preference
    playerProps, err := api.ds.UserProps(ctx).Get(user.ID, "tracking_enabled_"+player.ID)
    if err == nil && playerProps == "false" {
        return true
    }
    
    return false
}
```

**Add to Router:** `server/nativeapi/native_api.go`

```go
func (api *Router) routes() http.Handler {
    r := chi.NewRouter()
    
    // ... existing routes
    
    r.Group(func(r chi.Router) {
        r.Use(server.Authenticator(api.ds))
        r.Use(server.JWTRefresher)
        r.Use(server.UpdateLastAccessMiddleware(api.ds))
        
        // ... existing routes
        
        // Telemetry endpoints
        if conf.Server.EnableTelemetry {
            r.Post("/events/listen", rateLimiter(api.RecordListenEvents))
        }
    })
    
    return r
}

func rateLimiter(next http.HandlerFunc) http.HandlerFunc {
    limiter := httprate.NewRateLimiter(maxRequestsPerMinute, time.Minute)
    return limiter.Limit(next)
}
```

**Configuration:** Add to `conf/configuration.go`

```go
type configOptions struct {
    // ... existing fields
    EnableTelemetry bool   `mapstructure:"EnableTelemetry"`
}

// Default: enabled
func init() {
    viper.SetDefault("EnableTelemetry", true)
}
```

**Tasks:**
- [ ] Create `server/nativeapi/listen_events.go`
- [ ] Add route in `server/nativeapi/native_api.go`
- [ ] Add configuration option
- [ ] Write unit tests (valid requests, invalid requests, rate limiting)
- [ ] Write integration tests (end-to-end with client)

**Deliverable:** API endpoint + tests

**Acceptance Criteria:**
- Endpoint accepts valid events (201 Created)
- Endpoint rejects invalid events (400 Bad Request with error details)
- Endpoint enforces rate limiting (429 Too Many Requests)
- Endpoint respects incognito mode (204 No Content)
- Bulk insert handles 100 events in <100ms

---

### 4. Privacy Controls

**Goal:** Implement user-facing privacy controls (incognito mode, device tracking opt-out).

**User Preferences Schema:**

Store in `user_props` table:
```
user_id: "user-123"
key: "incognito_mode"
value: "true" | "false"

user_id: "user-123"
key: "tracking_enabled_player-456"
value: "true" | "false"
```

**UI Components:**

**Settings Page:** Add toggle for incognito mode

```tsx
// ui/src/user/UserPreferences.jsx

<FormControlLabel
    control={
        <Switch
            checked={incognitoMode}
            onChange={handleIncognitoToggle}
            color="primary"
        />
    }
    label="Incognito Mode (don't track my listening)"
/>
<FormHelperText>
    When enabled, your playback events won't be recorded. Existing data remains.
</FormHelperText>
```

**Player Settings:** Add per-device opt-out

```tsx
// ui/src/player/PlayerSettings.jsx

<FormControlLabel
    control={
        <Switch
            checked={trackingEnabled}
            onChange={handleTrackingToggle}
            color="primary"
        />
    }
    label={`Track listening on this device (${playerName})`}
/>
```

**Client-Side Incognito Session:**

```typescript
// ui/src/audioplayer/telemetry.ts

class TelemetryClient {
    private incognitoSession = false;
    
    startIncognitoSession() {
        this.incognitoSession = true;
        localStorage.setItem('incognitoSession', 'true');
    }
    
    endIncognitoSession() {
        this.incognitoSession = false;
        localStorage.removeItem('incognitoSession');
    }
    
    isIncognito() {
        return this.incognitoSession || localStorage.getItem('incognitoMode') === 'true';
    }
    
    recordEvent(event: ListenEvent) {
        if (this.isIncognito()) {
            return;  // Don't send events
        }
        this.eventBuffer.push(event);
    }
}
```

**Tasks:**
- [ ] Add privacy settings UI (user preferences page)
- [ ] Add API endpoints for privacy settings (GET/PUT /api/user/:id/preferences)
- [ ] Implement client-side incognito mode (localStorage flag)
- [ ] Add "Incognito Session" button to player UI (temporary mode)
- [ ] Document privacy features in user guide

**Deliverable:** Privacy UI + API + documentation

**Acceptance Criteria:**
- User can toggle incognito mode (saved to user_props)
- User can disable tracking per device (saved to user_props)
- Client respects incognito mode (no events sent)
- Server respects privacy settings (returns 204 if disabled)

---

### 5. Custom Web Player (Event Emission)

**Goal:** Fork `navidrome-music-player` package and add event emission.

**Strategy:**
- Fork npm package: `navidrome-music-player` → `navidrome-music-player-telemetry`
- Add event hooks to player state machine
- Emit events to buffer (don't call API directly from player)
- Buffer flushes periodically (every 10s) or on batch size (50 events)

**Event Hooks:**

```typescript
// ui/src/audioplayer/TelemetryPlayer.ts

import { NavidromePlayer } from 'navidrome-music-player';

export class TelemetryPlayer extends NavidromePlayer {
    private telemetry: TelemetryClient;
    private currentSession: string;
    
    constructor(options) {
        super(options);
        this.telemetry = new TelemetryClient(options.apiClient);
        this.currentSession = generateSessionId();
        
        // Hook into player events
        this.on('play', this.handlePlay.bind(this));
        this.on('pause', this.handlePause.bind(this));
        this.on('resume', this.handleResume.bind(this));
        this.on('seek', this.handleSeek.bind(this));
        this.on('ended', this.handleEnded.bind(this));
        this.on('error', this.handleError.bind(this));
    }
    
    private handlePlay(track: Track) {
        this.telemetry.recordEvent({
            mediaFileId: track.id,
            eventType: 'play',
            timestamp: Date.now(),
            position: 0,
            metadata: {
                sessionId: this.currentSession,
                clientVersion: APP_VERSION,
            }
        });
    }
    
    private handlePause() {
        this.telemetry.recordEvent({
            mediaFileId: this.currentTrack.id,
            eventType: 'pause',
            timestamp: Date.now(),
            position: this.getCurrentPosition() * 1000,  // Convert to ms
            metadata: {
                sessionId: this.currentSession,
            }
        });
    }
    
    // ... similar for resume, seek, ended, error
}
```

**Event Buffer:**

```typescript
// ui/src/audioplayer/EventBuffer.ts

export class EventBuffer {
    private events: ListenEvent[] = [];
    private flushInterval = 10000;  // 10 seconds
    private maxBatchSize = 50;
    private flushTimer: number;
    
    constructor(private apiClient: ApiClient) {
        this.startFlushTimer();
        
        // Flush on page unload
        window.addEventListener('beforeunload', () => this.flush());
    }
    
    push(event: ListenEvent) {
        this.events.push(event);
        
        if (this.events.length >= this.maxBatchSize) {
            this.flush();
        }
    }
    
    private startFlushTimer() {
        this.flushTimer = window.setInterval(() => {
            this.flush();
        }, this.flushInterval);
    }
    
    private async flush() {
        if (this.events.length === 0) return;
        
        const batch = this.events.splice(0, this.maxBatchSize);
        
        try {
            await this.apiClient.post('/api/events/listen', batch);
        } catch (error) {
            console.error('Failed to send listen events', error);
            
            // Re-queue events (with limit to prevent memory leak)
            if (this.events.length < 500) {
                this.events.unshift(...batch);
            }
        }
    }
    
    destroy() {
        clearInterval(this.flushTimer);
        this.flush();  // Final flush
    }
}
```

**Tasks:**
- [ ] Fork `navidrome-music-player` package (create `ui/src/audioplayer/TelemetryPlayer.ts`)
- [ ] Implement event hooks (play, pause, resume, seek, ended, error)
- [ ] Implement `EventBuffer` class
- [ ] Integrate into main App component
- [ ] Add debug mode (log events to console)
- [ ] Test with real playback (verify events are sent)

**Deliverable:** Custom player with telemetry + buffer

**Acceptance Criteria:**
- Player emits events on all playback actions
- Events are buffered and sent in batches
- Events are not sent in incognito mode
- No performance degradation (buffering is async)

---

### 6. Basic Telemetry Dashboard

**Goal:** Add simple UI to view raw events (for debugging and user transparency).

**Dashboard Components:**

**My Listening History Page:**
```tsx
// ui/src/personal/ListeningHistory.jsx

export const ListeningHistory = () => {
    const [events, setEvents] = useState([]);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        fetchEvents();
    }, []);
    
    const fetchEvents = async () => {
        const response = await apiClient.get('/api/events/listen/me?limit=100');
        setEvents(response.data);
        setLoading(false);
    };
    
    return (
        <Card>
            <CardContent>
                <Typography variant="h5">My Listening History</Typography>
                <List>
                    {events.map(event => (
                        <ListItem key={event.id}>
                            <ListItemText
                                primary={event.mediaFileTitle}
                                secondary={`${event.eventType} at ${formatTime(event.timestamp)}`}
                            />
                        </ListItem>
                    ))}
                </List>
            </CardContent>
        </Card>
    );
};
```

**API Endpoint:** `GET /api/events/listen/me`

```go
// server/nativeapi/listen_events.go

func (api *Router) GetMyListenEvents(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    user, _ := request.UserFrom(ctx)
    
    // Parse query params
    limit := getIntParam(r, "limit", 100)
    since := getTimeParam(r, "since", time.Time{})
    until := getTimeParam(r, "until", time.Now())
    
    // Fetch events
    events, err := api.ds.ListenEvent(ctx).GetByUser(user.ID, since, until, limit)
    if err != nil {
        log.Error(ctx, "Failed to fetch listen events", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    // Hydrate with media file info (join)
    enrichedEvents := api.enrichEvents(ctx, events)
    
    // Return JSON
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(enrichedEvents)
}

func (api *Router) enrichEvents(ctx context.Context, events model.ListenEvents) []map[string]interface{} {
    var enriched []map[string]interface{}
    
    for _, event := range events {
        mf, err := api.ds.MediaFile(ctx).Get(event.MediaFileID)
        if err != nil {
            continue  // Skip if media file was deleted
        }
        
        enriched = append(enriched, map[string]interface{}{
            "id":              event.ID,
            "eventType":       event.EventType,
            "timestamp":       event.Timestamp.Unix(),
            "position":        event.Position,
            "mediaFileId":     event.MediaFileID,
            "mediaFileTitle":  mf.Title,
            "mediaFileArtist": mf.Artist,
            "mediaFileAlbum":  mf.Album,
        })
    }
    
    return enriched
}
```

**Add Route:**
```go
r.Get("/events/listen/me", api.GetMyListenEvents)
```

**Tasks:**
- [ ] Create API endpoint for fetching user's events
- [ ] Create React component for listening history
- [ ] Add route to main navigation (under "Personal")
- [ ] Add date range filter (last 7 days, last 30 days, custom)
- [ ] Add event type filter (show only play events, show only skips)

**Deliverable:** Listening history page + API

**Acceptance Criteria:**
- User can view their recent listen events
- Events are paginated (100 per page)
- Events show track info (title, artist, album)
- Page loads in <2 seconds for 100 events

---

### 7. Deployment & Validation

**Goal:** Deploy to staging, validate telemetry pipeline end-to-end.

**Deployment Steps:**
1. Build Navidrome with telemetry changes: `make build`
2. Build UI with telemetry player: `cd ui && npm run build`
3. Deploy to staging: `docker build -t navidrome-telemetry:staging .`
4. Run migrations: `navidrome migrate up`
5. Start server: `docker run -p 4533:4533 navidrome-telemetry:staging`
6. Test with real users (internal team first)

**Validation Tests:**
- [ ] Play a track, verify event is recorded (check DB)
- [ ] Pause/resume, verify events are recorded
- [ ] Seek, verify event is recorded with previous position
- [ ] End track, verify event is recorded
- [ ] Enable incognito mode, verify no events are recorded
- [ ] Disable incognito mode, verify events are recorded again
- [ ] Flush browser (page reload), verify buffered events are sent
- [ ] High-frequency playback (skip through tracks), verify all events are captured
- [ ] Check DB size (should grow slowly, ~150 bytes/event)
- [ ] Check API latency (POST /api/events/listen should be <50ms for 50 events)

**Monitoring:**
- [ ] Add Prometheus metrics:
  - `navidrome_telemetry_events_ingested_total` (counter)
  - `navidrome_telemetry_events_rejected_total` (counter)
  - `navidrome_telemetry_db_size_bytes` (gauge)
- [ ] Create Grafana dashboard:
  - Events ingested per minute
  - Event rejection rate
  - DB size growth
  - API latency (P50, P95, P99)

**Tasks:**
- [ ] Deploy to staging environment
- [ ] Run end-to-end tests (automated + manual)
- [ ] Monitor for 3 days (ensure stability)
- [ ] Collect feedback from internal users
- [ ] Fix bugs (if any)
- [ ] Document deployment process

**Deliverable:** Staging deployment + validation report

**Acceptance Criteria:**
- Staging environment is stable (no crashes for 3 days)
- All validation tests pass
- Internal users can see their listening history
- No performance degradation (compared to Phase 1 baseline)

---

## Interfaces Introduced/Modified

### New Interfaces

**ListenEventRepository** (in `model/listen_event.go`):
```go
type ListenEventRepository interface {
    Insert(events []ListenEvent) error
    GetByUser(userID string, since time.Time, until time.Time, limit int) (ListenEvents, error)
    GetByMediaFile(mediaFileID string, since time.Time, limit int) (ListenEvents, error)
    GetBySession(userID string, playerID string, sessionID string) (ListenEvents, error)
    DeleteOlderThan(timestamp time.Time) (int64, error)
    Count(userID string, since time.Time) (int64, error)
}
```

### Modified Interfaces

**DataStore** (in `model/datastore.go`):
```go
type DataStore interface {
    // ... existing methods
    ListenEvent(ctx context.Context) ListenEventRepository  // NEW
}
```

---

## Data Models Introduced/Modified

### New Tables

**listen_event:**
```sql
CREATE TABLE listen_event (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         VARCHAR(255) NOT NULL,
    player_id       VARCHAR(255) NOT NULL,
    media_file_id   VARCHAR(255) NOT NULL,
    event_type      VARCHAR(50)  NOT NULL,
    timestamp       INTEGER      NOT NULL,
    position        INTEGER      NOT NULL,
    metadata        TEXT,
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE,
    FOREIGN KEY (player_id) REFERENCES player(id) ON DELETE CASCADE,
    FOREIGN KEY (media_file_id) REFERENCES media_file(id) ON DELETE CASCADE
);
```

### New Indexes

```sql
CREATE INDEX idx_listen_event_user_ts ON listen_event(user_id, timestamp DESC);
CREATE INDEX idx_listen_event_media_file ON listen_event(media_file_id, timestamp DESC);
CREATE INDEX idx_listen_event_timestamp ON listen_event(timestamp DESC);
CREATE INDEX idx_listen_event_user_player ON listen_event(user_id, player_id, timestamp DESC);
```

---

## Operational Needs

### Infrastructure
- Staging environment (updated with Phase 2 build)
- Database backup before migration (SQLite .backup command)

### Monitoring
- Prometheus metrics for telemetry ingestion
- Grafana dashboard for telemetry health
- Alerts for:
  - High event rejection rate (>5%)
  - High API latency (P95 >100ms)
  - Rapid DB growth (>1GB/day, indicates bug)

### Data Retention
- Plan for event purging (delete events older than 1 year)
- Scheduled job (runs daily): `DELETE FROM listen_event WHERE timestamp < (NOW() - INTERVAL 365 DAYS)`
- Configuration option: `ND_TELEMETRY_RETENTION_DAYS=365`

---

## Risks & Mitigations

### Risk: DB size grows too quickly
**Likelihood:** Medium  
**Impact:** High (disk full, performance degradation)  
**Mitigation:**
- Monitor DB size daily (Prometheus metric + alert)
- Implement purge policy (delete old events automatically)
- Consider compression (SQLite VACUUM)
- If still an issue, migrate to TimescaleDB (Phase 5)

### Risk: Event ingestion impacts streaming performance
**Likelihood:** Low  
**Impact:** High  
**Mitigation:**
- Use async event insertion (don't block stream requests)
- Use bulk inserts (reduce transaction overhead)
- Add separate DB connection pool for telemetry writes
- If still an issue, move ingestion to separate service (queue-based)

### Risk: Users disable telemetry (poor adoption)
**Likelihood:** Medium  
**Impact:** Medium (less data for analytics)  
**Mitigation:**
- Make value proposition clear (show users their own stats)
- Default to enabled (but allow opt-out, GDPR-compliant)
- Educate users on privacy controls (incognito mode, device opt-out)

### Risk: Client event buffer loses events (browser crash)
**Likelihood:** Medium  
**Impact:** Low (some events lost, but acceptable)  
**Mitigation:**
- Persist buffer to localStorage (survive page reloads)
- Send events on beforeunload event (best-effort)
- Accept that some events will be lost (buffered telemetry trade-off)

---

## Definition of Done

### Must Have
- ✅ `listen_event` table created (migration tested)
- ✅ `ListenEventRepository` implemented and tested
- ✅ `POST /api/events/listen` endpoint working (tested end-to-end)
- ✅ Privacy controls implemented (incognito mode, device opt-out)
- ✅ Custom web player emits events (play, pause, resume, seek, ended)
- ✅ Event buffer flushes periodically (10s or 50 events)
- ✅ Listening history page shows user's events
- ✅ Deployed to staging and validated (3 days stable)

### Should Have
- ✅ Rate limiting on ingestion endpoint
- ✅ Event validation (reject invalid events)
- ✅ Prometheus metrics for telemetry
- ✅ Grafana dashboard for monitoring
- ✅ Data retention policy documented

### Could Have
- Message queue integration (defer to Phase 5 if needed)
- Mobile client telemetry (defer to later phase)
- Export events to CSV (user data export)

---

## Success Metrics

- **Event Capture Rate:** >95% of playback actions result in recorded events
- **Ingestion Latency:** P95 <50ms for bulk insert (50 events)
- **Rejection Rate:** <1% (due to validation errors)
- **DB Growth:** ~274 MB/year for 100 users (within estimate)
- **User Adoption:** >80% of users leave telemetry enabled (default on)

---

## Next Steps (Transition to Phase 3)

After Phase 2 is complete:
1. Monitor staging for 1 week (ensure stability)
2. Deploy to production (small user cohort first)
3. Create Phase 3 branch: `feature/social-core-v1`
4. Begin work on social graph (follows, likes, comments)

---

**Estimated Effort:**
- Senior Backend Engineer: 80 hours (repository, API, privacy)
- Mid-Level Backend Engineer: 60 hours (validation, testing, deployment)
- Senior Frontend Engineer: 80 hours (custom player, event buffer, UI)
- Mid-Level Frontend Engineer: 40 hours (listening history page, settings UI)
- **Total: 260 hours (~6.5 person-weeks with 40-hour weeks)**

---

**End of Phase 2**
