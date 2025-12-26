# Phase 5: Analytics Expansion (Deep Listening Metrics)
**Duration:** 4-5 weeks  
**Team Size:** 4-5 engineers  
**Dependencies:** Phase 2 complete (telemetry ingestion operational)

---

## Objective

Build the **Analytics Service** to compute derived metrics from raw listen events:
1. Calculate listened intervals (which portions of tracks were actually played)
2. Compute completion percentage (what % of track was heard)
3. Detect skip patterns (where users skip tracks)
4. Identify replay hotspots (sections users replay)
5. Generate session analytics (binge sessions, skip-heavy sessions)
6. Build listening insights dashboard (personal stats, trends)

This enables **deep listening analytics** - understanding *how* users listen, not just *what* they listen to.

---

## Scope

### In Scope
- ✅ Analytics Service (external microservice)
- ✅ TimescaleDB for time-series data (efficient for interval queries)
- ✅ Interval calculation (derive listened intervals from events)
- ✅ Derived metrics (completion %, skip cliffs, replay hotspots)
- ✅ Session analytics (detect listening sessions, analyze patterns)
- ✅ GraphQL API for analytics queries
- ✅ Listening insights dashboard (personal stats)
- ✅ Admin analytics (aggregate stats, popular tracks)

### Out of Scope
- ❌ Recommendations (defer to Phase 6)
- ❌ ML-based predictions (defer to Phase 6)
- ❌ Real-time analytics (batch processing for v1)
- ❌ Export to BI tools (Looker, Tableau) - defer to later phase

---

## Architecture

### Why TimescaleDB?

**TimescaleDB = PostgreSQL + time-series extensions**

Benefits for analytics:
- **Efficient time-range queries:** Optimized for `WHERE timestamp BETWEEN`
- **Automatic partitioning:** Data partitioned by time (monthly chunks)
- **Continuous aggregates:** Pre-computed rollups (hourly, daily, weekly)
- **Compression:** Old data compressed (90% storage reduction)
- **PostgreSQL compatibility:** Use existing SQL tools

**Comparison to alternatives:**

| Feature | TimescaleDB | InfluxDB | Prometheus | SQLite |
|---------|-------------|----------|------------|--------|
| SQL support | ✅ Full | ❌ InfluxQL only | ❌ PromQL only | ✅ Full |
| Relational joins | ✅ Native | ❌ Limited | ❌ None | ✅ Native |
| Time-series optimization | ✅ Excellent | ✅ Excellent | ✅ Excellent | ⚠️ Manual |
| Compression | ✅ Native | ✅ Native | ✅ Native | ⚠️ Manual |
| Retention policies | ✅ Native | ✅ Native | ✅ Native | ⚠️ Manual |
| Continuous aggregates | ✅ Native | ✅ Downsampling | ✅ Recording rules | ❌ None |

**Decision:** TimescaleDB (best fit for relational + time-series workloads)

### Data Flow

```
Navidrome (SQLite)
  └─> listen_event table (raw events)
       └─> NATS publish (navidrome.listen.event)
            └─> Analytics Service (consumer)
                 └─> TimescaleDB (raw_events → derived_metrics)
                      └─> GraphQL API (queries)
                           └─> UI (dashboard)
```

---

## Key Technical Work Items

### 1. Analytics Service Setup

**Goal:** Bootstrap Analytics Service with TimescaleDB, NATS consumer, GraphQL API.

**Project Structure:**

```
analytics-service/
├── cmd/
│   ├── server/
│   │   └── main.go                    # GraphQL API server
│   └── worker/
│       └── main.go                    # Background worker (compute metrics)
├── internal/
│   ├── domain/
│   │   ├── listen_event.go
│   │   ├── listened_interval.go
│   │   ├── track_stats.go
│   │   └── session.go
│   ├── repository/
│   │   └── timescale/
│   │       ├── event_repository.go
│   │       ├── interval_repository.go
│   │       ├── stats_repository.go
│   │       └── session_repository.go
│   ├── service/
│   │   ├── interval_calculator.go
│   │   ├── stats_calculator.go
│   │   └── session_analyzer.go
│   ├── graphql/
│   │   ├── schema.graphql
│   │   ├── resolver.go
│   │   └── generated/
│   └── nats/
│       └── event_consumer.go
├── migrations/
│   ├── 001_initial_schema.sql
│   └── ...
├── docker-compose.yml
├── Dockerfile
├── go.mod
└── README.md
```

**Dependencies:**

```go
// go.mod
module github.com/navidrome/analytics-service

go 1.25

require (
    github.com/99designs/gqlgen v0.17.42
    github.com/jackc/pgx/v5 v5.5.0
    github.com/nats-io/nats.go v1.31.0
    github.com/robfig/cron/v3 v3.0.1          // Background jobs
    github.com/spf13/viper v1.18.2
    github.com/rs/zerolog v1.31.0
)
```

**Configuration:**

```yaml
server:
  port: 8081
  host: 0.0.0.0

database:
  host: localhost
  port: 5432
  database: analytics
  user: analytics_user
  password: ${DB_PASSWORD}
  sslmode: disable

nats:
  url: nats://localhost:4222
  subjects:
    listenEvent: navidrome.listen.event

worker:
  enabled: true
  intervalCalculation:
    schedule: "*/5 * * * *"  # Every 5 minutes
  statsCalculation:
    schedule: "0 * * * *"    # Every hour
  sessionAnalysis:
    schedule: "*/10 * * * *" # Every 10 minutes
```

**Tasks:**
- [ ] Create project structure
- [ ] Initialize Go module
- [ ] Add dependencies
- [ ] Create `main.go` for API server
- [ ] Create `worker/main.go` for background jobs
- [ ] Setup Docker Compose (TimescaleDB + NATS)
- [ ] Test service startup

**Deliverable:** Runnable Analytics Service skeleton

---

### 2. TimescaleDB Schema

**Goal:** Create hypertables for raw events and derived metrics.

**Schema:**

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Raw events (synced from Navidrome)
CREATE TABLE raw_events (
    id              BIGSERIAL,
    user_id         VARCHAR(255) NOT NULL,
    player_id       VARCHAR(255) NOT NULL,
    media_file_id   VARCHAR(255) NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    position        INTEGER NOT NULL,  -- milliseconds
    metadata        JSONB,
    PRIMARY KEY (timestamp, id)        -- Composite PK for hypertable
);

-- Convert to hypertable (partitioned by time)
SELECT create_hypertable('raw_events', 'timestamp', chunk_time_interval => INTERVAL '1 month');

-- Indexes
CREATE INDEX idx_raw_events_user ON raw_events(user_id, timestamp DESC);
CREATE INDEX idx_raw_events_media_file ON raw_events(media_file_id, timestamp DESC);
CREATE INDEX idx_raw_events_event_type ON raw_events(event_type, timestamp DESC);

-- Compression policy (compress chunks older than 7 days)
ALTER TABLE raw_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'user_id,media_file_id',
    timescaledb.compress_orderby = 'timestamp DESC'
);

SELECT add_compression_policy('raw_events', INTERVAL '7 days');

-- Retention policy (delete chunks older than 2 years)
SELECT add_retention_policy('raw_events', INTERVAL '2 years');

-- ===== Derived Metrics =====

-- Listened intervals (which portions of tracks were played)
CREATE TABLE listened_intervals (
    id              BIGSERIAL,
    user_id         VARCHAR(255) NOT NULL,
    media_file_id   VARCHAR(255) NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,  -- When interval started
    start_position  INTEGER NOT NULL,      -- Position in track (ms)
    end_position    INTEGER NOT NULL,      -- Position in track (ms)
    duration_ms     INTEGER NOT NULL,      -- end_position - start_position
    PRIMARY KEY (start_time, id)
);

SELECT create_hypertable('listened_intervals', 'start_time', chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_listened_intervals_user ON listened_intervals(user_id, start_time DESC);
CREATE INDEX idx_listened_intervals_media_file ON listened_intervals(media_file_id, start_time DESC);

ALTER TABLE listened_intervals SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'user_id,media_file_id',
    timescaledb.compress_orderby = 'start_time DESC'
);

SELECT add_compression_policy('listened_intervals', INTERVAL '7 days');

-- Track stats (aggregated metrics per track)
CREATE TABLE track_stats (
    id                  BIGSERIAL,
    media_file_id       VARCHAR(255) NOT NULL,
    time_bucket         TIMESTAMPTZ NOT NULL,  -- Hourly bucket
    listen_count        INTEGER DEFAULT 0,
    completion_count    INTEGER DEFAULT 0,     -- Completed listens (>80%)
    skip_count          INTEGER DEFAULT 0,     -- Skipped early (<20%)
    avg_completion_pct  FLOAT DEFAULT 0,
    total_listen_time   INTEGER DEFAULT 0,     -- Total milliseconds listened
    unique_listeners    INTEGER DEFAULT 0,
    PRIMARY KEY (time_bucket, media_file_id)
);

SELECT create_hypertable('track_stats', 'time_bucket', chunk_time_interval => INTERVAL '1 week');

CREATE INDEX idx_track_stats_media_file ON track_stats(media_file_id, time_bucket DESC);

-- Continuous aggregate (daily rollup)
CREATE MATERIALIZED VIEW track_stats_daily
WITH (timescaledb.continuous) AS
SELECT 
    media_file_id,
    time_bucket('1 day', time_bucket) AS day,
    SUM(listen_count) AS total_listens,
    SUM(completion_count) AS total_completions,
    SUM(skip_count) AS total_skips,
    AVG(avg_completion_pct) AS avg_completion_pct,
    SUM(total_listen_time) AS total_listen_time,
    SUM(unique_listeners) AS total_unique_listeners
FROM track_stats
GROUP BY media_file_id, day;

-- Refresh policy (every hour)
SELECT add_continuous_aggregate_policy('track_stats_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Listening sessions (grouped events)
CREATE TABLE listening_sessions (
    id              BIGSERIAL,
    user_id         VARCHAR(255) NOT NULL,
    player_id       VARCHAR(255) NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    duration        INTEGER NOT NULL,  -- seconds
    track_count     INTEGER DEFAULT 0,
    skip_count      INTEGER DEFAULT 0,
    replay_count    INTEGER DEFAULT 0,
    session_type    VARCHAR(50),       -- 'binge', 'commute', 'workout', 'discovery'
    PRIMARY KEY (start_time, id)
);

SELECT create_hypertable('listening_sessions', 'start_time', chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_listening_sessions_user ON listening_sessions(user_id, start_time DESC);

ALTER TABLE listening_sessions SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'user_id,session_type',
    timescaledb.compress_orderby = 'start_time DESC'
);

SELECT add_compression_policy('listening_sessions', INTERVAL '30 days');
```

**Design Decisions:**
- **Hypertables:** All tables partitioned by time (automatic chunk management)
- **Composite PKs:** `(timestamp, id)` for optimal time-range queries
- **Compression:** Old data compressed automatically (7-30 days)
- **Continuous aggregates:** Daily rollups for fast dashboard queries
- **Retention policies:** Auto-delete old data (2 years for raw events)

**Migration File:** `analytics-service/migrations/001_initial_schema.sql`

**Tasks:**
- [ ] Create migration files
- [ ] Test migrations (up/down)
- [ ] Verify hypertables created (`\d+ raw_events`)
- [ ] Verify compression/retention policies active

**Deliverable:** TimescaleDB schema + migrations

---

### 3. Interval Calculation (Core Algorithm)

**Goal:** Compute listened intervals from raw events.

**Algorithm:**

```
Input: Sequence of events for a track
  - play(position=0)
  - pause(position=30000)
  - resume(position=30000)
  - pause(position=90000)
  - seek(position=150000)
  - play(position=150000)
  - ended(position=180000)

Output: Listened intervals
  - [0ms, 30000ms]       (30s)
  - [30000ms, 90000ms]   (60s)
  - [150000ms, 180000ms] (30s)

Total listened: 120s
Gaps: [90000ms, 150000ms] (60s skipped)
```

**Implementation:** `analytics-service/internal/service/interval_calculator.go`

```go
package service

import (
    "sort"
    "time"
    
    "github.com/navidrome/analytics-service/internal/domain"
)

type IntervalCalculator struct {
    repo domain.IntervalRepository
}

func NewIntervalCalculator(repo domain.IntervalRepository) *IntervalCalculator {
    return &IntervalCalculator{repo: repo}
}

// CalculateIntervals computes listened intervals from raw events
func (c *IntervalCalculator) CalculateIntervals(events []domain.ListenEvent) ([]domain.ListenedInterval, error) {
    // Sort events by timestamp
    sort.Slice(events, func(i, j int) bool {
        return events[i].Timestamp.Before(events[j].Timestamp)
    })
    
    var intervals []domain.ListenedInterval
    var currentInterval *domain.ListenedInterval
    
    for i, event := range events {
        switch event.EventType {
        case "play":
            // Start new interval
            currentInterval = &domain.ListenedInterval{
                UserID:        event.UserID,
                MediaFileID:   event.MediaFileID,
                StartTime:     event.Timestamp,
                StartPosition: event.Position,
                EndPosition:   event.Position,  // Will be updated on pause/ended
            }
        
        case "pause", "ended":
            // Close current interval
            if currentInterval != nil {
                currentInterval.EndPosition = event.Position
                currentInterval.DurationMs = currentInterval.EndPosition - currentInterval.StartPosition
                
                // Only save if duration > 1s (filter noise)
                if currentInterval.DurationMs > 1000 {
                    intervals = append(intervals, *currentInterval)
                }
                currentInterval = nil
            }
        
        case "seek":
            // Close current interval (if active) and start new one
            if currentInterval != nil {
                currentInterval.EndPosition = events[i-1].Position  // Use previous position
                currentInterval.DurationMs = currentInterval.EndPosition - currentInterval.StartPosition
                
                if currentInterval.DurationMs > 1000 {
                    intervals = append(intervals, *currentInterval)
                }
            }
            
            // Start new interval at seek position (if playback continues)
            if i+1 < len(events) && (events[i+1].EventType == "play" || events[i+1].EventType == "resume") {
                currentInterval = &domain.ListenedInterval{
                    UserID:        event.UserID,
                    MediaFileID:   event.MediaFileID,
                    StartTime:     event.Timestamp,
                    StartPosition: event.Position,
                    EndPosition:   event.Position,
                }
            } else {
                currentInterval = nil
            }
        
        case "resume":
            // Resume previous interval (or start new if none)
            if currentInterval == nil {
                currentInterval = &domain.ListenedInterval{
                    UserID:        event.UserID,
                    MediaFileID:   event.MediaFileID,
                    StartTime:     event.Timestamp,
                    StartPosition: event.Position,
                    EndPosition:   event.Position,
                }
            }
        }
    }
    
    // Close any open interval (user left page without ending)
    if currentInterval != nil {
        // Estimate end position (assume playback continued for 5s max)
        estimatedDuration := 5000  // ms
        currentInterval.EndPosition = currentInterval.StartPosition + estimatedDuration
        currentInterval.DurationMs = estimatedDuration
        
        if currentInterval.DurationMs > 1000 {
            intervals = append(intervals, *currentInterval)
        }
    }
    
    return intervals, nil
}

// ComputeCompletionPercentage calculates what % of track was listened to
func (c *IntervalCalculator) ComputeCompletionPercentage(intervals []domain.ListenedInterval, trackDuration int) float64 {
    // Merge overlapping intervals
    merged := mergeIntervals(intervals)
    
    // Sum total listened time
    totalListened := 0
    for _, interval := range merged {
        totalListened += interval.DurationMs
    }
    
    // Return percentage
    return float64(totalListened) / float64(trackDuration*1000) * 100
}

// mergeIntervals combines overlapping intervals
func mergeIntervals(intervals []domain.ListenedInterval) []domain.ListenedInterval {
    if len(intervals) == 0 {
        return intervals
    }
    
    // Sort by start position
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i].StartPosition < intervals[j].StartPosition
    })
    
    merged := []domain.ListenedInterval{intervals[0]}
    
    for i := 1; i < len(intervals); i++ {
        current := intervals[i]
        last := &merged[len(merged)-1]
        
        // Overlapping or adjacent?
        if current.StartPosition <= last.EndPosition {
            // Merge (extend end position)
            if current.EndPosition > last.EndPosition {
                last.EndPosition = current.EndPosition
                last.DurationMs = last.EndPosition - last.StartPosition
            }
        } else {
            // Non-overlapping, add as new interval
            merged = append(merged, current)
        }
    }
    
    return merged
}
```

**Background Job:** Run every 5 minutes

```go
// cmd/worker/main.go

func main() {
    // ... setup
    
    c := cron.New()
    c.AddFunc("*/5 * * * *", func() {
        log.Info().Msg("Running interval calculation...")
        
        // Get unprocessed events (events without corresponding intervals)
        events, err := eventRepo.GetUnprocessed(ctx)
        if err != nil {
            log.Error().Err(err).Msg("Failed to get unprocessed events")
            return
        }
        
        // Group events by (user, media_file, session)
        grouped := groupEventsBySession(events)
        
        // Calculate intervals for each session
        for _, sessionEvents := range grouped {
            intervals, err := intervalCalc.CalculateIntervals(sessionEvents)
            if err != nil {
                log.Error().Err(err).Msg("Failed to calculate intervals")
                continue
            }
            
            // Save intervals
            if err := intervalRepo.InsertBatch(ctx, intervals); err != nil {
                log.Error().Err(err).Msg("Failed to insert intervals")
            }
        }
        
        log.Info().Msgf("Processed %d events into intervals", len(events))
    })
    
    c.Start()
    
    // Keep alive
    select {}
}
```

**Tasks:**
- [ ] Implement interval calculation algorithm
- [ ] Implement completion percentage calculation
- [ ] Implement interval merging (handle overlaps)
- [ ] Create background job (cron scheduler)
- [ ] Test with real event sequences (verify intervals are correct)

**Deliverable:** Interval calculation service

**Acceptance Criteria:**
- Intervals are calculated correctly from events
- Overlapping intervals are merged
- Completion percentage is accurate (±1%)
- Background job runs every 5 minutes

---

### 4. Derived Metrics (Stats Calculation)

**Goal:** Compute aggregate stats from intervals.

**Metrics:**

1. **Completion %:** What % of track was listened to
2. **Skip cliff:** Position where most users skip (early exit)
3. **Replay hotspot:** Section users replay most often
4. **Average listen time:** Mean duration users listen before skipping

**Implementation:** `analytics-service/internal/service/stats_calculator.go`

```go
func (s *StatsCalculator) CalculateTrackStats(mediaFileID string, timeBucket time.Time) (*domain.TrackStats, error) {
    // Get all intervals for track in time bucket (e.g., last hour)
    intervals, err := s.intervalRepo.GetByMediaFile(ctx, mediaFileID, timeBucket, timeBucket.Add(1*time.Hour))
    if err != nil {
        return nil, err
    }
    
    // Get track duration (from Navidrome API)
    track, err := s.navidromeClient.GetTrack(ctx, mediaFileID)
    if err != nil {
        return nil, err
    }
    trackDuration := track.Duration * 1000  // Convert to ms
    
    stats := &domain.TrackStats{
        MediaFileID:    mediaFileID,
        TimeBucket:     timeBucket,
        ListenCount:    0,
        CompletionCount: 0,
        SkipCount:      0,
        TotalListenTime: 0,
    }
    
    // Group intervals by user (one listen per user)
    userIntervals := groupIntervalsByUser(intervals)
    
    stats.UniqueListeners = len(userIntervals)
    
    for userID, userIntervalsForTrack := range userIntervals {
        stats.ListenCount++
        
        // Calculate completion %
        completionPct := s.intervalCalc.ComputeCompletionPercentage(userIntervalsForTrack, track.Duration)
        
        if completionPct >= 80.0 {
            stats.CompletionCount++
        } else if completionPct < 20.0 {
            stats.SkipCount++
        }
        
        // Sum listen time
        for _, interval := range userIntervalsForTrack {
            stats.TotalListenTime += interval.DurationMs
        }
        
        stats.AvgCompletionPct += completionPct
    }
    
    // Average completion %
    if stats.ListenCount > 0 {
        stats.AvgCompletionPct /= float64(stats.ListenCount)
    }
    
    return stats, nil
}

// DetectSkipCliff identifies position where users skip most often
func (s *StatsCalculator) DetectSkipCliff(mediaFileID string) (int, error) {
    // Get all intervals for track (last 30 days)
    intervals, err := s.intervalRepo.GetByMediaFile(ctx, mediaFileID, time.Now().Add(-30*24*time.Hour), time.Now())
    if err != nil {
        return 0, err
    }
    
    // Create histogram of end positions (where users stopped)
    histogram := make(map[int]int)  // Position (10s buckets) → count
    
    for _, interval := range intervals {
        bucket := (interval.EndPosition / 10000) * 10000  // Round to 10s
        histogram[bucket]++
    }
    
    // Find bucket with highest count (skip cliff)
    maxCount := 0
    skipCliff := 0
    for position, count := range histogram {
        if count > maxCount {
            maxCount = count
            skipCliff = position
        }
    }
    
    return skipCliff, nil
}

// DetectReplayHotspot identifies section users replay most often
func (s *StatsCalculator) DetectReplayHotspot(mediaFileID string) (int, int, error) {
    // Get all seek events for track (last 30 days)
    events, err := s.eventRepo.GetByMediaFile(ctx, mediaFileID, time.Now().Add(-30*24*time.Hour), time.Now())
    if err != nil {
        return 0, 0, err
    }
    
    // Filter seek events (backward seeks = replays)
    var replays []domain.ListenEvent
    for i, event := range events {
        if event.EventType == "seek" && i > 0 {
            prevPosition := events[i-1].Position
            if event.Position < prevPosition {
                // Backward seek = replay
                replays = append(replays, event)
            }
        }
    }
    
    // Create histogram of replay positions (10s buckets)
    histogram := make(map[int]int)
    for _, replay := range replays {
        bucket := (replay.Position / 10000) * 10000
        histogram[bucket]++
    }
    
    // Find bucket with highest count (replay hotspot)
    maxCount := 0
    hotspotStart := 0
    for position, count := range histogram {
        if count > maxCount {
            maxCount = count
            hotspotStart = position
        }
    }
    
    return hotspotStart, hotspotStart + 10000, nil
}
```

**Background Job:** Run every hour

```go
c.AddFunc("0 * * * *", func() {
    log.Info().Msg("Calculating track stats...")
    
    // Get all tracks with activity in last hour
    tracks, err := eventRepo.GetActiveTracksInLastHour(ctx)
    if err != nil {
        log.Error().Err(err).Msg("Failed to get active tracks")
        return
    }
    
    timeBucket := time.Now().Truncate(1 * time.Hour)
    
    for _, trackID := range tracks {
        stats, err := statsCalc.CalculateTrackStats(trackID, timeBucket)
        if err != nil {
            log.Error().Err(err).Str("trackId", trackID).Msg("Failed to calculate stats")
            continue
        }
        
        // Save stats
        if err := statsRepo.Insert(ctx, stats); err != nil {
            log.Error().Err(err).Msg("Failed to insert stats")
        }
    }
    
    log.Info().Msgf("Calculated stats for %d tracks", len(tracks))
})
```

**Tasks:**
- [ ] Implement track stats calculation
- [ ] Implement skip cliff detection
- [ ] Implement replay hotspot detection
- [ ] Create background job (hourly)
- [ ] Test with real data

**Deliverable:** Stats calculation service

**Acceptance Criteria:**
- Track stats are calculated hourly
- Skip cliffs are detected accurately
- Replay hotspots are identified
- Continuous aggregates refresh automatically

---

### 5. GraphQL Schema (Analytics Queries)

**Goal:** Expose analytics metrics via GraphQL.

**Schema:** `analytics-service/internal/graphql/schema.graphql`

```graphql
schema {
  query: Query
}

# ===== Analytics Types =====

type TrackAnalytics {
  trackId: ID!
  track: Track                       # Resolved from Navidrome
  listenCount: Int!
  completionCount: Int!
  skipCount: Int!
  avgCompletionPct: Float!
  totalListenTime: Int!              # Milliseconds
  uniqueListeners: Int!
  skipCliff: Int                     # Position (ms) where users skip most
  replayHotspot: TimeRange           # Section users replay most
}

type TimeRange {
  startMs: Int!
  endMs: Int!
}

type UserListeningStats {
  userId: ID!
  totalListens: Int!
  totalListenTime: Int!              # Milliseconds
  uniqueTracks: Int!
  avgCompletionPct: Float!
  favoriteGenres: [GenreStats!]!
  topTracks: [TrackAnalytics!]!
  listeningSessions: [ListeningSession!]!
}

type GenreStats {
  genre: String!
  listenCount: Int!
  percentage: Float!
}

type ListeningSession {
  id: ID!
  startTime: String!
  endTime: String!
  duration: Int!                     # Seconds
  trackCount: Int!
  skipCount: Int!
  replayCount: Int!
  sessionType: String                # 'binge', 'commute', 'workout', 'discovery'
}

type ListeningTrend {
  date: String!                      # YYYY-MM-DD
  listenCount: Int!
  listenTime: Int!                   # Minutes
}

# ===== Queries =====

type Query {
  # User analytics
  myListeningStats(
    since: String                    # ISO8601 date
    until: String
  ): UserListeningStats!
  
  # Track analytics
  trackAnalytics(
    trackId: ID!
    since: String
    until: String
  ): TrackAnalytics!
  
  # Popular tracks (admin/public)
  popularTracks(
    timeRange: TimeRange = WEEK      # DAY, WEEK, MONTH, YEAR, ALL_TIME
    limit: Int = 50
  ): [TrackAnalytics!]!
  
  # Listening trends (time series)
  listeningTrend(
    userId: ID
    since: String
    until: String
    granularity: Granularity = DAY   # HOUR, DAY, WEEK, MONTH
  ): [ListeningTrend!]!
  
  # Session analytics
  listeningSessions(
    userId: ID!
    since: String
    limit: Int = 50
  ): [ListeningSession!]!
}

enum TimeRange {
  DAY
  WEEK
  MONTH
  YEAR
  ALL_TIME
}

enum Granularity {
  HOUR
  DAY
  WEEK
  MONTH
}
```

**Resolver Example:**

```go
func (r *Resolver) MyListeningStats(ctx context.Context, args struct {
    Since *string
    Until *string
}) (*domain.UserListeningStats, error) {
    userID := getUserIDFromContext(ctx)
    
    since := parseTime(args.Since, time.Now().Add(-30*24*time.Hour))
    until := parseTime(args.Until, time.Now())
    
    // Query intervals for user
    intervals, err := r.intervalRepo.GetByUser(ctx, userID, since, until)
    if err != nil {
        return nil, err
    }
    
    stats := &domain.UserListeningStats{
        UserID:         userID,
        TotalListens:   len(intervals),
        TotalListenTime: 0,
        UniqueTracks:   0,
    }
    
    // Calculate stats
    trackSet := make(map[string]bool)
    var totalCompletion float64
    
    for _, interval := range intervals {
        stats.TotalListenTime += interval.DurationMs
        trackSet[interval.MediaFileID] = true
        
        // Get track info (for completion %)
        track, _ := r.navidromeClient.GetTrack(ctx, interval.MediaFileID)
        if track != nil {
            completion := float64(interval.DurationMs) / float64(track.Duration*1000) * 100
            totalCompletion += completion
        }
    }
    
    stats.UniqueTracks = len(trackSet)
    if stats.TotalListens > 0 {
        stats.AvgCompletionPct = totalCompletion / float64(stats.TotalListens)
    }
    
    // Get favorite genres (from track metadata)
    stats.FavoriteGenres = r.calculateFavoriteGenres(ctx, intervals)
    
    // Get top tracks (most listened)
    stats.TopTracks = r.getTopTracks(ctx, userID, since, until, 10)
    
    // Get sessions
    stats.ListeningSessions = r.getSessions(ctx, userID, since, until, 10)
    
    return stats, nil
}
```

**Tasks:**
- [ ] Create GraphQL schema
- [ ] Implement resolvers
- [ ] Add authentication (extract user ID from JWT)
- [ ] Test queries (verify data accuracy)

**Deliverable:** Analytics GraphQL API

**Acceptance Criteria:**
- Can query user's listening stats (total listens, completion %, top tracks)
- Can query track analytics (skip cliff, replay hotspot)
- Can query popular tracks (most listened)
- Queries are performant (<500ms for typical requests)

---

### 6. Listening Insights Dashboard (UI)

**Goal:** Build UI to display personal listening analytics.

**Components:**

**Dashboard Page:** `ui/src/analytics/AnalyticsDashboard.jsx`

```tsx
export const AnalyticsDashboard = () => {
    const [stats, setStats] = useState(null);
    const [timeRange, setTimeRange] = useState('WEEK');
    
    const { data, loading } = useGraphQL(`
        query GetMyStats($since: String, $until: String) {
            myListeningStats(since: $since, until: $until) {
                totalListens
                totalListenTime
                uniqueTracks
                avgCompletionPct
                favoriteGenres {
                    genre
                    listenCount
                    percentage
                }
                topTracks {
                    trackId
                    track {
                        title
                        artist
                    }
                    listenCount
                    avgCompletionPct
                }
            }
        }
    `, getTimeRangeParams(timeRange));
    
    useEffect(() => {
        if (data) setStats(data.myListeningStats);
    }, [data]);
    
    if (loading) return <CircularProgress />;
    if (!stats) return <div>No data available</div>;
    
    return (
        <div className="analytics-dashboard">
            <Typography variant="h4">Your Listening Stats</Typography>
            
            <TimeRangeSelector value={timeRange} onChange={setTimeRange} />
            
            <Grid container spacing={3}>
                {/* Summary cards */}
                <Grid item xs={12} md={3}>
                    <StatCard title="Total Listens" value={stats.totalListens} />
                </Grid>
                <Grid item xs={12} md={3}>
                    <StatCard title="Listen Time" value={formatDuration(stats.totalListenTime)} />
                </Grid>
                <Grid item xs={12} md={3}>
                    <StatCard title="Unique Tracks" value={stats.uniqueTracks} />
                </Grid>
                <Grid item xs={12} md={3}>
                    <StatCard title="Avg Completion" value={`${stats.avgCompletionPct.toFixed(1)}%`} />
                </Grid>
                
                {/* Charts */}
                <Grid item xs={12} md={6}>
                    <Card>
                        <CardContent>
                            <Typography variant="h6">Favorite Genres</Typography>
                            <PieChart data={stats.favoriteGenres} />
                        </CardContent>
                    </Card>
                </Grid>
                
                <Grid item xs={12} md={6}>
                    <Card>
                        <CardContent>
                            <Typography variant="h6">Top Tracks</Typography>
                            <List>
                                {stats.topTracks.map(track => (
                                    <ListItem key={track.trackId}>
                                        <ListItemText
                                            primary={track.track.title}
                                            secondary={`${track.track.artist} · ${track.listenCount} listens`}
                                        />
                                    </ListItem>
                                ))}
                            </List>
                        </CardContent>
                    </Card>
                </Grid>
                
                {/* Listening trend chart */}
                <Grid item xs={12}>
                    <ListeningTrendChart userId={currentUserId} timeRange={timeRange} />
                </Grid>
            </Grid>
        </div>
    );
};
```

**Listening Trend Chart:**

```tsx
export const ListeningTrendChart = ({ userId, timeRange }) => {
    const { data, loading } = useGraphQL(`
        query GetTrend($userId: ID, $since: String, $granularity: Granularity) {
            listeningTrend(userId: $userId, since: $since, granularity: $granularity) {
                date
                listenCount
                listenTime
            }
        }
    `, { userId, ...getTimeRangeParams(timeRange) });
    
    if (loading) return <CircularProgress />;
    
    const chartData = data?.listeningTrend || [];
    
    return (
        <Card>
            <CardContent>
                <Typography variant="h6">Listening Activity</Typography>
                <LineChart
                    data={chartData}
                    xKey="date"
                    yKey="listenTime"
                    yLabel="Minutes"
                />
            </CardContent>
        </Card>
    );
};
```

**Tasks:**
- [ ] Create dashboard page component
- [ ] Create stat cards (total listens, listen time, etc.)
- [ ] Create pie chart (favorite genres)
- [ ] Create line chart (listening trend)
- [ ] Add time range selector (last 7 days, last 30 days, etc.)
- [ ] Add route to navigation

**Deliverable:** Listening insights dashboard

**Acceptance Criteria:**
- Dashboard shows personal stats (accurate data)
- Charts are interactive (tooltips, zoom)
- Time range selector works (updates data)
- Dashboard loads in <2 seconds

---

## Operational Needs

### Infrastructure
- TimescaleDB (managed service: Timescale Cloud, AWS RDS with TimescaleDB extension)
- Analytics Service deployment (Docker container, scaled horizontally)
- Background worker deployment (single instance with cron jobs)

### Monitoring
- Prometheus metrics:
  - `analytics_intervals_calculated_total` (counter)
  - `analytics_stats_calculated_total` (counter)
  - `analytics_query_latency_seconds` (histogram)
- Alerts for:
  - Background job failures
  - Query latency >1s
  - TimescaleDB disk usage >80%

### Data Retention
- Raw events: 2 years (auto-deleted by retention policy)
- Intervals: 2 years
- Stats: 5 years (smaller data size)
- Continuous aggregates: Never expire (daily rollups)

---

## Risks & Mitigations

### Risk: Interval calculation is inaccurate (gaps, overlaps)
**Likelihood:** Medium  
**Impact:** High  
**Mitigation:**
- Comprehensive unit tests (edge cases: seeks, pauses, page reloads)
- Manual validation (compare calculated intervals with actual playback)
- Allow user feedback ("This doesn't look right")

### Risk: Background jobs lag (can't keep up with event volume)
**Likelihood:** Low  
**Impact:** Medium  
**Mitigation:**
- Monitor job execution time (alert if >5 minutes)
- Scale background workers horizontally (process events in parallel)
- Optimize queries (use indexes, covering indexes)

### Risk: TimescaleDB costs too much (large data volume)
**Likelihood:** Low  
**Impact:** Medium  
**Mitigation:**
- Use compression (90% reduction)
- Use retention policies (auto-delete old data)
- If still too expensive, migrate to ClickHouse (column-oriented DB, cheaper for analytics)

---

## Definition of Done

### Must Have
- ✅ Analytics Service deployed and running
- ✅ TimescaleDB schema created (raw_events, listened_intervals, track_stats)
- ✅ Interval calculation working (background job)
- ✅ Stats calculation working (background job)
- ✅ GraphQL API for analytics queries
- ✅ Listening insights dashboard (personal stats)

### Should Have
- ✅ Continuous aggregates (daily rollups)
- ✅ Compression and retention policies
- ✅ Skip cliff and replay hotspot detection

### Could Have
- Recommendations based on analytics (defer to Phase 6)
- Real-time analytics (WebSocket updates) - defer
- Export analytics to CSV/JSON - defer

---

## Success Metrics

- **Analytics Accuracy:** Interval calculation >95% accurate (manual validation)
- **Query Performance:** P95 <500ms for analytics queries
- **Background Job Performance:** Intervals calculated within 10 minutes of events
- **Dashboard Engagement:** >40% of users view analytics dashboard at least once per month

---

## Next Steps (Transition to Phase 6)

After Phase 5 is complete:
1. Monitor analytics service performance (1 week)
2. Collect user feedback on analytics dashboard
3. Create Phase 6 branch: `feature/recommendations`
4. Begin work on recommendation engine (collaborative filtering, taste matching)

---

**Estimated Effort:**
- Senior Backend Engineer (Go): 100 hours (Analytics Service, interval calculation, stats calculation)
- Mid-Level Backend Engineer (Go): 60 hours (TimescaleDB setup, background jobs)
- Senior Frontend Engineer (React): 60 hours (analytics dashboard, charts)
- Mid-Level Frontend Engineer (React): 40 hours (UI polish, responsiveness)
- **Total: 260 hours (~6.5 person-weeks with 40-hour weeks)**

---

**End of Phase 5**
