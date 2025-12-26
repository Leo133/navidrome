# Phase 3: Social Core v1
**Duration:** 4-5 weeks  
**Team Size:** 5-6 engineers  
**Dependencies:** Phase 2 complete (telemetry pipeline operational)

---

## Objective

Build the **foundational social layer** to enable user-to-user interactions:
1. Implement social graph (follow/unfollow, followers/following lists)
2. Build activity feed (see what friends are listening to)
3. Add likes on tracks/albums/playlists
4. Implement privacy controls (private/friends/public profiles)
5. Deploy Social Service as external microservice (PostgreSQL + GraphQL API)
6. Integrate social features into Navidrome UI

**No complex analytics yet** - this phase focuses on social primitives. Phase 5 will add derived metrics and recommendations.

---

## Scope

### In Scope
- ✅ Social Service (external microservice)
- ✅ PostgreSQL database for social data
- ✅ GraphQL API for social queries
- ✅ Social graph (follow/unfollow, followers/following)
- ✅ Activity feed (timeline of friends' listens)
- ✅ Likes on tracks/albums/playlists
- ✅ Privacy controls (profile visibility, activity visibility)
- ✅ Navidrome API integration (sync user data, player data)
- ✅ UI for social features (profile pages, follow buttons, feed)

### Out of Scope
- ❌ Timestamped comments (Moments) - covered in Phase 4
- ❌ Complex analytics (skip patterns, replay hotspots) - covered in Phase 5
- ❌ Recommendations - covered in Phase 6
- ❌ Group listening (shared sessions) - covered in later phase
- ❌ Direct messaging - covered in later phase

---

## Architecture

### Social Service Overview

**Technology Stack:**
- **Language:** Go 1.25+ (consistent with Navidrome)
- **Database:** PostgreSQL 15+ (better for relational data than SQLite)
- **API:** GraphQL (flexible queries, efficient for social use cases)
- **Cache:** Redis (activity feed caching, hot data)
- **Message Queue:** NATS (event-driven sync from Navidrome)

**Why External Service:**
- **Isolation:** Social features don't impact core Navidrome performance
- **Scalability:** PostgreSQL handles relational queries (follows, likes) better than SQLite
- **Flexibility:** GraphQL allows UI to fetch exactly what it needs (no over-fetching)
- **Independent deployment:** Can update social features without Navidrome rebuild

**Communication Pattern:**
```
Navidrome → NATS → Social Service  (event-driven sync)
UI → Social Service GraphQL API    (direct queries)
```

---

## Key Technical Work Items

### 1. Social Service Setup

**Goal:** Bootstrap external service with PostgreSQL, GraphQL, and NATS integration.

**Project Structure:**
```
social-service/
├── cmd/
│   └── server/
│       └── main.go                  # Service entry point
├── internal/
│   ├── domain/                      # Domain models
│   │   ├── user.go
│   │   ├── follow.go
│   │   ├── like.go
│   │   └── activity.go
│   ├── repository/                  # Data access layer
│   │   ├── postgres/
│   │   │   ├── user_repository.go
│   │   │   ├── follow_repository.go
│   │   │   ├── like_repository.go
│   │   │   └── activity_repository.go
│   │   └── redis/
│   │       └── feed_cache.go
│   ├── service/                     # Business logic
│   │   ├── follow_service.go
│   │   ├── like_service.go
│   │   ├── feed_service.go
│   │   └── privacy_service.go
│   ├── graphql/                     # GraphQL schema & resolvers
│   │   ├── schema.graphql
│   │   ├── resolver.go
│   │   └── generated/               # gqlgen generated code
│   └── nats/                        # NATS event consumers
│       ├── user_sync.go
│       ├── listen_event_consumer.go
│       └── consumer.go
├── migrations/                      # PostgreSQL migrations
│   ├── 001_initial_schema.sql
│   ├── 002_add_privacy_controls.sql
│   └── ...
├── docker-compose.yml               # Local development
├── Dockerfile
├── go.mod
└── README.md
```

**Dependencies:**
```go
// go.mod
module github.com/navidrome/social-service

go 1.25

require (
    github.com/99designs/gqlgen v0.17.42       // GraphQL server
    github.com/jackc/pgx/v5 v5.5.0             // PostgreSQL driver
    github.com/nats-io/nats.go v1.31.0         // NATS client
    github.com/redis/go-redis/v9 v9.3.0        // Redis client
    github.com/pressly/goose/v3 v3.16.0        // Database migrations
    github.com/spf13/viper v1.18.2             // Configuration
    github.com/rs/zerolog v1.31.0              // Logging
)
```

**Configuration:** `config.yaml`

```yaml
server:
  port: 8080
  host: 0.0.0.0

database:
  host: localhost
  port: 5432
  database: social
  user: social_user
  password: ${DB_PASSWORD}  # From env var
  sslmode: disable          # Enable in production
  maxConnections: 20

redis:
  host: localhost
  port: 6379
  db: 0
  password: ${REDIS_PASSWORD}

nats:
  url: nats://localhost:4222
  subjects:
    userCreated: navidrome.user.created
    userUpdated: navidrome.user.updated
    userDeleted: navidrome.user.deleted
    listenEvent: navidrome.listen.event

navidrome:
  apiUrl: http://localhost:4533
  apiToken: ${NAVIDROME_API_TOKEN}  # Service-to-service auth

graphql:
  playground: true  # Disable in production
  introspection: true

cache:
  feedTTL: 300  # 5 minutes
  maxFeedSize: 100
```

**Main Entry Point:** `cmd/server/main.go`

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/playground"
    "github.com/navidrome/social-service/internal/config"
    "github.com/navidrome/social-service/internal/graphql"
    "github.com/navidrome/social-service/internal/nats"
    "github.com/navidrome/social-service/internal/repository/postgres"
    "github.com/navidrome/social-service/internal/repository/redis"
    "github.com/navidrome/social-service/internal/service"
    "github.com/rs/zerolog/log"
)

func main() {
    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        log.Fatal().Err(err).Msg("Failed to load configuration")
    }

    // Initialize PostgreSQL
    db, err := postgres.NewConnection(cfg.Database)
    if err != nil {
        log.Fatal().Err(err).Msg("Failed to connect to PostgreSQL")
    }
    defer db.Close()

    // Initialize Redis
    redisClient, err := redis.NewClient(cfg.Redis)
    if err != nil {
        log.Fatal().Err(err).Msg("Failed to connect to Redis")
    }
    defer redisClient.Close()

    // Initialize repositories
    userRepo := postgres.NewUserRepository(db)
    followRepo := postgres.NewFollowRepository(db)
    likeRepo := postgres.NewLikeRepository(db)
    activityRepo := postgres.NewActivityRepository(db)
    feedCache := redis.NewFeedCache(redisClient)

    // Initialize services
    followService := service.NewFollowService(followRepo, userRepo)
    likeService := service.NewLikeService(likeRepo)
    feedService := service.NewFeedService(activityRepo, followRepo, feedCache)
    privacyService := service.NewPrivacyService(userRepo)

    // Initialize GraphQL resolver
    resolver := graphql.NewResolver(followService, likeService, feedService, privacyService)
    schema := graphql.NewExecutableSchema(graphql.Config{Resolvers: resolver})
    graphqlHandler := handler.NewDefaultServer(schema)

    // Setup HTTP server
    mux := http.NewServeMux()
    mux.Handle("/graphql", graphqlHandler)
    if cfg.GraphQL.Playground {
        mux.Handle("/playground", playground.Handler("GraphQL Playground", "/graphql"))
    }

    server := &http.Server{
        Addr:    fmt.Sprintf("%s:%d", cfg.Server.Host, cfg.Server.Port),
        Handler: mux,
    }

    // Initialize NATS consumer (event-driven sync)
    natsConsumer, err := nats.NewConsumer(cfg.NATS, userRepo, activityRepo)
    if err != nil {
        log.Fatal().Err(err).Msg("Failed to initialize NATS consumer")
    }
    defer natsConsumer.Close()

    // Start server
    go func() {
        log.Info().Msgf("Starting Social Service on %s", server.Addr)
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal().Err(err).Msg("Server failed")
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Info().Msg("Shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatal().Err(err).Msg("Server forced to shutdown")
    }

    log.Info().Msg("Server exited")
}
```

**Tasks:**
- [ ] Create project structure (`social-service/` directory)
- [ ] Initialize Go module (`go mod init`)
- [ ] Add dependencies (`go get` for gqlgen, pgx, NATS, Redis)
- [ ] Create `main.go` and basic HTTP server
- [ ] Create `config.yaml` and config loader
- [ ] Setup Docker Compose for local dev (PostgreSQL + Redis + NATS)
- [ ] Test service startup (should listen on port 8080)

**Deliverable:** Runnable Social Service skeleton

**Acceptance Criteria:**
- Service starts without errors
- Health check endpoint `/health` returns 200 OK
- Can connect to PostgreSQL, Redis, NATS

---

### 2. Database Schema: Social Graph

**Goal:** Create PostgreSQL tables for social data.

**Schema Design:**

```sql
-- Users (synced from Navidrome)
CREATE TABLE users (
    id              VARCHAR(255) PRIMARY KEY,      -- Navidrome user ID
    username        VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    avatar_url      TEXT,
    bio             TEXT,
    profile_visibility VARCHAR(50) DEFAULT 'public', -- 'public', 'friends', 'private'
    activity_visibility VARCHAR(50) DEFAULT 'friends', -- 'public', 'friends', 'private'
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMP  -- Soft delete
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_deleted_at ON users(deleted_at) WHERE deleted_at IS NULL;

-- Follows (social graph)
CREATE TABLE follows (
    id              SERIAL PRIMARY KEY,
    follower_id     VARCHAR(255) NOT NULL,         -- User doing the following
    following_id    VARCHAR(255) NOT NULL,         -- User being followed
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (follower_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE(follower_id, following_id)             -- Prevent duplicate follows
);

CREATE INDEX idx_follows_follower ON follows(follower_id, created_at DESC);
CREATE INDEX idx_follows_following ON follows(following_id, created_at DESC);

-- Likes (tracks, albums, playlists)
CREATE TABLE likes (
    id              SERIAL PRIMARY KEY,
    user_id         VARCHAR(255) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,          -- 'track', 'album', 'playlist'
    entity_id       VARCHAR(255) NOT NULL,         -- Navidrome media_file.id, album.id, playlist.id
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE(user_id, entity_type, entity_id)       -- Prevent duplicate likes
);

CREATE INDEX idx_likes_user ON likes(user_id, created_at DESC);
CREATE INDEX idx_likes_entity ON likes(entity_type, entity_id, created_at DESC);

-- Activities (feed items)
CREATE TABLE activities (
    id              SERIAL PRIMARY KEY,
    user_id         VARCHAR(255) NOT NULL,
    activity_type   VARCHAR(50) NOT NULL,          -- 'listen', 'like', 'follow', 'playlist_create'
    entity_type     VARCHAR(50),                   -- 'track', 'album', 'playlist', 'user'
    entity_id       VARCHAR(255),
    metadata        JSONB,                         -- Additional context (track title, duration, etc.)
    timestamp       TIMESTAMP NOT NULL,            -- When the activity occurred
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_activities_user_timestamp ON activities(user_id, timestamp DESC);
CREATE INDEX idx_activities_timestamp ON activities(timestamp DESC);
CREATE INDEX idx_activities_entity ON activities(entity_type, entity_id, timestamp DESC);

-- Feed materialized view (for performance)
CREATE MATERIALIZED VIEW user_feed AS
SELECT 
    a.id AS activity_id,
    a.user_id AS activity_user_id,
    a.activity_type,
    a.entity_type,
    a.entity_id,
    a.metadata,
    a.timestamp,
    f.follower_id AS viewer_id
FROM activities a
JOIN follows f ON a.user_id = f.following_id
ORDER BY a.timestamp DESC;

CREATE INDEX idx_user_feed_viewer ON user_feed(viewer_id, timestamp DESC);

-- Refresh feed (run periodically or on-demand)
-- REFRESH MATERIALIZED VIEW CONCURRENTLY user_feed;
```

**Design Decisions:**
- **users table synced from Navidrome:** Avoid dual authentication (single source of truth)
- **soft delete (deleted_at):** Keep referential integrity when users are deleted
- **profile_visibility & activity_visibility:** Granular privacy controls
- **follows with UNIQUE constraint:** Prevent duplicate follow relationships
- **likes with UNIQUE constraint:** Prevent duplicate likes (one like per user per entity)
- **activities table:** Log all social actions for feed generation
- **Materialized view for feed:** Pre-compute feed for performance (refresh every 5 minutes)
- **JSONB metadata:** Flexible storage for entity details (avoid JOIN to Navidrome DB)

**Migration File:** `migrations/001_initial_schema.sql`

```sql
-- Run with: goose up
-- (migrations are in social-service/migrations/)
```

**Tasks:**
- [ ] Create migration files (`001_initial_schema.sql`)
- [ ] Test migrations locally (`goose up`, `goose down`)
- [ ] Document schema in `DATABASE_SCHEMA.md`
- [ ] Add seed data for testing (test users, follows, likes)

**Deliverable:** PostgreSQL schema + migrations

**Acceptance Criteria:**
- Migrations run successfully (up and down)
- Schema matches design (all tables, indexes, constraints created)
- Seed data can be inserted without errors

---

### 3. NATS Event Sync (Navidrome → Social Service)

**Goal:** Sync user data and listen events from Navidrome to Social Service.

**Event Schema:**

**User Created:**
```json
{
  "subject": "navidrome.user.created",
  "data": {
    "id": "user-123",
    "username": "alice",
    "displayName": "Alice Johnson",
    "email": "alice@example.com",
    "createdAt": "2025-01-10T12:00:00Z"
  }
}
```

**User Updated:**
```json
{
  "subject": "navidrome.user.updated",
  "data": {
    "id": "user-123",
    "username": "alice",
    "displayName": "Alice J.",
    "updatedAt": "2025-01-11T14:00:00Z"
  }
}
```

**User Deleted:**
```json
{
  "subject": "navidrome.user.deleted",
  "data": {
    "id": "user-123",
    "deletedAt": "2025-01-12T16:00:00Z"
  }
}
```

**Listen Event:**
```json
{
  "subject": "navidrome.listen.event",
  "data": {
    "userId": "user-123",
    "playerId": "player-456",
    "mediaFileId": "track-789",
    "eventType": "play",
    "timestamp": 1704067200000,
    "position": 0,
    "metadata": {
      "sessionId": "session-abc",
      "trackTitle": "Bohemian Rhapsody",
      "trackArtist": "Queen",
      "trackDuration": 354000
    }
  }
}
```

**Navidrome Event Publisher:**

Add NATS publishing to Navidrome:

**File:** `core/events/publisher.go`

```go
package events

import (
    "context"
    "encoding/json"
    
    "github.com/nats-io/nats.go"
    "github.com/navidrome/navidrome/conf"
    "github.com/navidrome/navidrome/log"
)

type Publisher struct {
    nc *nats.Conn
}

func NewPublisher() (*Publisher, error) {
    nc, err := nats.Connect(conf.Server.NatsURL)
    if err != nil {
        return nil, err
    }
    return &Publisher{nc: nc}, nil
}

func (p *Publisher) PublishUserCreated(ctx context.Context, user User) error {
    data, _ := json.Marshal(user)
    return p.nc.Publish("navidrome.user.created", data)
}

func (p *Publisher) PublishUserUpdated(ctx context.Context, user User) error {
    data, _ := json.Marshal(user)
    return p.nc.Publish("navidrome.user.updated", data)
}

func (p *Publisher) PublishUserDeleted(ctx context.Context, userID string) error {
    data, _ := json.Marshal(map[string]string{"id": userID})
    return p.nc.Publish("navidrome.user.deleted", data)
}

func (p *Publisher) PublishListenEvent(ctx context.Context, event ListenEvent) error {
    data, _ := json.Marshal(event)
    return p.nc.Publish("navidrome.listen.event", data)
}

func (p *Publisher) Close() {
    p.nc.Close()
}
```

**Hook into existing code:**

**User creation:** `persistence/user_repository.go`

```go
func (r *userRepository) Put(user *model.User) error {
    // ... existing code
    
    // Publish event
    if r.eventPublisher != nil {
        r.eventPublisher.PublishUserCreated(r.ctx, user)
    }
    
    return nil
}
```

**Listen event ingestion:** `server/nativeapi/listen_events.go`

```go
func (api *Router) RecordListenEvents(w http.ResponseWriter, r *http.Request) {
    // ... existing validation and insertion
    
    // Publish events to NATS
    for _, event := range events {
        api.eventPublisher.PublishListenEvent(ctx, event)
    }
    
    // ... rest of code
}
```

**Social Service Consumer:**

**File:** `social-service/internal/nats/consumer.go`

```go
package nats

import (
    "context"
    "encoding/json"
    
    "github.com/nats-io/nats.go"
    "github.com/navidrome/social-service/internal/domain"
    "github.com/navidrome/social-service/internal/repository/postgres"
    "github.com/rs/zerolog/log"
)

type Consumer struct {
    nc           *nats.Conn
    subs         []*nats.Subscription
    userRepo     *postgres.UserRepository
    activityRepo *postgres.ActivityRepository
}

func NewConsumer(cfg NATSConfig, userRepo *postgres.UserRepository, activityRepo *postgres.ActivityRepository) (*Consumer, error) {
    nc, err := nats.Connect(cfg.URL)
    if err != nil {
        return nil, err
    }
    
    c := &Consumer{
        nc:           nc,
        userRepo:     userRepo,
        activityRepo: activityRepo,
    }
    
    // Subscribe to user events
    c.subscribeUserCreated(cfg.Subjects.UserCreated)
    c.subscribeUserUpdated(cfg.Subjects.UserUpdated)
    c.subscribeUserDeleted(cfg.Subjects.UserDeleted)
    
    // Subscribe to listen events
    c.subscribeListenEvent(cfg.Subjects.ListenEvent)
    
    return c, nil
}

func (c *Consumer) subscribeUserCreated(subject string) {
    sub, err := c.nc.Subscribe(subject, func(msg *nats.Msg) {
        var user domain.User
        if err := json.Unmarshal(msg.Data, &user); err != nil {
            log.Error().Err(err).Msg("Failed to unmarshal user created event")
            return
        }
        
        // Insert user into social DB
        if err := c.userRepo.Create(&user); err != nil {
            log.Error().Err(err).Msg("Failed to create user in social DB")
            return
        }
        
        log.Info().Str("userId", user.ID).Msg("User synced to social DB")
    })
    
    if err != nil {
        log.Fatal().Err(err).Msg("Failed to subscribe to user.created")
    }
    c.subs = append(c.subs, sub)
}

func (c *Consumer) subscribeListenEvent(subject string) {
    sub, err := c.nc.Subscribe(subject, func(msg *nats.Msg) {
        var event domain.ListenEvent
        if err := json.Unmarshal(msg.Data, &event); err != nil {
            log.Error().Err(err).Msg("Failed to unmarshal listen event")
            return
        }
        
        // Create activity from listen event (if public or friends-only)
        activity := domain.Activity{
            UserID:       event.UserID,
            ActivityType: "listen",
            EntityType:   "track",
            EntityID:     event.MediaFileID,
            Metadata: map[string]interface{}{
                "trackTitle":  event.Metadata["trackTitle"],
                "trackArtist": event.Metadata["trackArtist"],
                "duration":    event.Metadata["trackDuration"],
            },
            Timestamp: event.Timestamp,
        }
        
        // Check privacy before inserting
        user, err := c.userRepo.Get(event.UserID)
        if err != nil || user.ActivityVisibility == "private" {
            return  // Don't create activity for private users
        }
        
        if err := c.activityRepo.Create(&activity); err != nil {
            log.Error().Err(err).Msg("Failed to create activity")
            return
        }
    })
    
    if err != nil {
        log.Fatal().Err(err).Msg("Failed to subscribe to listen.event")
    }
    c.subs = append(c.subs, sub)
}

func (c *Consumer) Close() {
    for _, sub := range c.subs {
        sub.Unsubscribe()
    }
    c.nc.Close()
}
```

**Tasks:**
- [ ] Add NATS client to Navidrome (`go get nats.go`)
- [ ] Create event publisher in Navidrome (`core/events/publisher.go`)
- [ ] Hook publisher into user repository (create/update/delete)
- [ ] Hook publisher into listen event ingestion
- [ ] Create NATS consumer in Social Service
- [ ] Test event flow end-to-end (create user in Navidrome → see in Social DB)
- [ ] Add error handling and retries (dead letter queue for failed events)

**Deliverable:** Event-driven sync (Navidrome → Social Service)

**Acceptance Criteria:**
- User created in Navidrome appears in Social Service within 1 second
- Listen event in Navidrome appears as activity in Social Service
- Failed events are retried (exponential backoff)
- NATS connection is resilient (auto-reconnect on failure)

---

### 4. GraphQL Schema & Resolvers

**Goal:** Define GraphQL schema for social queries and mutations.

**Schema:** `social-service/internal/graphql/schema.graphql`

```graphql
schema {
  query: Query
  mutation: Mutation
}

# ===== User Types =====

type User {
  id: ID!
  username: String!
  displayName: String
  avatarUrl: String
  bio: String
  profileVisibility: Visibility!
  activityVisibility: Visibility!
  followerCount: Int!
  followingCount: Int!
  isFollowing: Boolean!    # Current user following this user?
  isFollower: Boolean!     # This user following current user?
  createdAt: String!
}

enum Visibility {
  PUBLIC
  FRIENDS
  PRIVATE
}

# ===== Follow Types =====

type Follow {
  id: ID!
  follower: User!
  following: User!
  createdAt: String!
}

type FollowConnection {
  edges: [FollowEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type FollowEdge {
  node: User!
  cursor: String!
}

# ===== Like Types =====

type Like {
  id: ID!
  user: User!
  entityType: EntityType!
  entityId: ID!
  entity: Entity          # Resolved entity (Track, Album, Playlist)
  createdAt: String!
}

enum EntityType {
  TRACK
  ALBUM
  PLAYLIST
}

union Entity = Track | Album | Playlist

type Track {
  id: ID!
  title: String!
  artist: String!
  album: String!
  duration: Int!
  likeCount: Int!
  isLiked: Boolean!       # Current user liked this track?
}

type Album {
  id: ID!
  name: String!
  artist: String!
  year: Int
  trackCount: Int!
  likeCount: Int!
  isLiked: Boolean!
}

type Playlist {
  id: ID!
  name: String!
  owner: User!
  trackCount: Int!
  likeCount: Int!
  isLiked: Boolean!
}

# ===== Activity Feed Types =====

type Activity {
  id: ID!
  user: User!
  activityType: ActivityType!
  entityType: EntityType
  entityId: ID
  entity: Entity
  metadata: JSON
  timestamp: String!
}

enum ActivityType {
  LISTEN
  LIKE
  FOLLOW
  PLAYLIST_CREATE
  PLAYLIST_UPDATE
}

type ActivityConnection {
  edges: [ActivityEdge!]!
  pageInfo: PageInfo!
}

type ActivityEdge {
  node: Activity!
  cursor: String!
}

# ===== Pagination =====

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# ===== Queries =====

type Query {
  # Current user
  me: User!
  
  # User lookup
  user(id: ID!): User
  userByUsername(username: String!): User
  searchUsers(query: String!, limit: Int = 20): [User!]!
  
  # Followers / Following
  followers(userId: ID!, after: String, limit: Int = 50): FollowConnection!
  following(userId: ID!, after: String, limit: Int = 50): FollowConnection!
  
  # Activity Feed
  feed(after: String, limit: Int = 50): ActivityConnection!
  userActivity(userId: ID!, after: String, limit: Int = 50): ActivityConnection!
  
  # Likes
  userLikes(userId: ID!, entityType: EntityType, after: String, limit: Int = 50): [Like!]!
  entityLikes(entityType: EntityType!, entityId: ID!, limit: Int = 50): [Like!]!
  
  # Stats
  stats: Stats!
}

type Stats {
  totalUsers: Int!
  totalFollows: Int!
  totalLikes: Int!
  totalActivities: Int!
}

# ===== Mutations =====

type Mutation {
  # Follow / Unfollow
  follow(userId: ID!): Follow!
  unfollow(userId: ID!): Boolean!
  
  # Like / Unlike
  like(entityType: EntityType!, entityId: ID!): Like!
  unlike(entityType: EntityType!, entityId: ID!): Boolean!
  
  # Profile updates
  updateProfile(input: UpdateProfileInput!): User!
  updatePrivacy(input: UpdatePrivacyInput!): User!
}

input UpdateProfileInput {
  displayName: String
  bio: String
  avatarUrl: String
}

input UpdatePrivacyInput {
  profileVisibility: Visibility
  activityVisibility: Visibility
}

# ===== Scalars =====

scalar JSON
```

**Resolver Implementation:** `social-service/internal/graphql/resolver.go`

```go
package graphql

import (
    "context"
    
    "github.com/navidrome/social-service/internal/domain"
    "github.com/navidrome/social-service/internal/service"
)

type Resolver struct {
    followService  *service.FollowService
    likeService    *service.LikeService
    feedService    *service.FeedService
    privacyService *service.PrivacyService
}

func NewResolver(followService *service.FollowService, likeService *service.LikeService, feedService *service.FeedService, privacyService *service.PrivacyService) *Resolver {
    return &Resolver{
        followService:  followService,
        likeService:    likeService,
        feedService:    feedService,
        privacyService: privacyService,
    }
}

// ===== Query Resolvers =====

func (r *Resolver) Me(ctx context.Context) (*domain.User, error) {
    userID := getUserIDFromContext(ctx)
    return r.privacyService.GetUser(ctx, userID)
}

func (r *Resolver) User(ctx context.Context, args struct{ ID string }) (*domain.User, error) {
    currentUserID := getUserIDFromContext(ctx)
    
    // Check privacy (can current user view this profile?)
    canView, err := r.privacyService.CanViewProfile(ctx, currentUserID, args.ID)
    if err != nil || !canView {
        return nil, fmt.Errorf("profile not accessible")
    }
    
    return r.privacyService.GetUser(ctx, args.ID)
}

func (r *Resolver) Feed(ctx context.Context, args struct {
    After *string
    Limit *int
}) (*ActivityConnection, error) {
    userID := getUserIDFromContext(ctx)
    limit := getLimit(args.Limit, 50)
    
    // Get feed for user (activities from people they follow)
    activities, pageInfo, err := r.feedService.GetFeed(ctx, userID, args.After, limit)
    if err != nil {
        return nil, err
    }
    
    return &ActivityConnection{
        Edges:    activities,
        PageInfo: pageInfo,
    }, nil
}

// ===== Mutation Resolvers =====

func (r *Resolver) Follow(ctx context.Context, args struct{ UserID string }) (*domain.Follow, error) {
    followerID := getUserIDFromContext(ctx)
    
    // Check if target user allows follows (public or friends)
    canFollow, err := r.privacyService.CanFollow(ctx, followerID, args.UserID)
    if err != nil || !canFollow {
        return nil, fmt.Errorf("cannot follow this user")
    }
    
    return r.followService.Follow(ctx, followerID, args.UserID)
}

func (r *Resolver) Unfollow(ctx context.Context, args struct{ UserID string }) (bool, error) {
    followerID := getUserIDFromContext(ctx)
    return r.followService.Unfollow(ctx, followerID, args.UserID)
}

func (r *Resolver) Like(ctx context.Context, args struct {
    EntityType string
    EntityID   string
}) (*domain.Like, error) {
    userID := getUserIDFromContext(ctx)
    return r.likeService.Like(ctx, userID, args.EntityType, args.EntityID)
}

func (r *Resolver) Unlike(ctx context.Context, args struct {
    EntityType string
    EntityID   string
}) (bool, error) {
    userID := getUserIDFromContext(ctx)
    return r.likeService.Unlike(ctx, userID, args.EntityType, args.EntityID)
}

// ===== Helper Functions =====

func getUserIDFromContext(ctx context.Context) string {
    // Extract from JWT (passed via Authorization header)
    // Navidrome JWT includes user_id claim
    return ctx.Value("user_id").(string)
}

func getLimit(limit *int, defaultLimit int) int {
    if limit == nil || *limit <= 0 || *limit > 100 {
        return defaultLimit
    }
    return *limit
}
```

**Generate GraphQL Code:**

```bash
cd social-service
go run github.com/99designs/gqlgen generate
```

**Tasks:**
- [ ] Create `schema.graphql`
- [ ] Implement resolver methods
- [ ] Generate GraphQL code (`gqlgen generate`)
- [ ] Add authentication middleware (extract user ID from JWT)
- [ ] Test queries and mutations (GraphQL Playground)

**Deliverable:** Working GraphQL API

**Acceptance Criteria:**
- Can query `me` (current user info)
- Can query `feed` (activity feed)
- Can mutate `follow` (follow a user)
- Can mutate `like` (like a track)
- Queries respect privacy settings (can't see private profiles)

---

### 5. UI Integration (Navidrome UI)

**Goal:** Add social features to Navidrome web UI.

**Components to Build:**

**1. Profile Page** (`ui/src/social/ProfilePage.jsx`)

```tsx
import React, { useState, useEffect } from 'react';
import { useParams } from 'react-router-dom';
import { Avatar, Button, Typography, Card, CardContent, Tabs, Tab } from '@material-ui/core';
import { useGraphQL } from '../hooks/useGraphQL';

export const ProfilePage = () => {
    const { userId } = useParams();
    const [user, setUser] = useState(null);
    const [activeTab, setActiveTab] = useState(0);
    
    const { data, loading } = useGraphQL(`
        query GetUser($userId: ID!) {
            user(id: $userId) {
                id
                username
                displayName
                avatarUrl
                bio
                followerCount
                followingCount
                isFollowing
                isFollower
            }
        }
    `, { userId });
    
    useEffect(() => {
        if (data) setUser(data.user);
    }, [data]);
    
    const handleFollow = async () => {
        await client.mutate({
            mutation: FOLLOW_MUTATION,
            variables: { userId: user.id },
        });
        setUser({ ...user, isFollowing: true, followerCount: user.followerCount + 1 });
    };
    
    const handleUnfollow = async () => {
        await client.mutate({
            mutation: UNFOLLOW_MUTATION,
            variables: { userId: user.id },
        });
        setUser({ ...user, isFollowing: false, followerCount: user.followerCount - 1 });
    };
    
    if (loading) return <div>Loading...</div>;
    if (!user) return <div>User not found</div>;
    
    return (
        <Card>
            <CardContent>
                <Avatar src={user.avatarUrl} style={{ width: 100, height: 100 }} />
                <Typography variant="h4">{user.displayName || user.username}</Typography>
                <Typography variant="body2" color="textSecondary">@{user.username}</Typography>
                <Typography variant="body1" style={{ marginTop: 16 }}>{user.bio}</Typography>
                
                <div style={{ marginTop: 16 }}>
                    <Typography variant="caption">
                        <strong>{user.followerCount}</strong> followers · 
                        <strong>{user.followingCount}</strong> following
                    </Typography>
                </div>
                
                <div style={{ marginTop: 16 }}>
                    {user.isFollowing ? (
                        <Button variant="outlined" onClick={handleUnfollow}>Unfollow</Button>
                    ) : (
                        <Button variant="contained" color="primary" onClick={handleFollow}>Follow</Button>
                    )}
                </div>
                
                <Tabs value={activeTab} onChange={(e, val) => setActiveTab(val)}>
                    <Tab label="Activity" />
                    <Tab label="Likes" />
                    <Tab label="Followers" />
                    <Tab label="Following" />
                </Tabs>
                
                {activeTab === 0 && <UserActivityList userId={user.id} />}
                {activeTab === 1 && <UserLikesList userId={user.id} />}
                {activeTab === 2 && <FollowersList userId={user.id} />}
                {activeTab === 3 && <FollowingList userId={user.id} />}
            </CardContent>
        </Card>
    );
};
```

**2. Activity Feed** (`ui/src/social/ActivityFeed.jsx`)

```tsx
export const ActivityFeed = () => {
    const { data, loading, fetchMore } = useGraphQL(`
        query GetFeed($after: String, $limit: Int) {
            feed(after: $after, limit: $limit) {
                edges {
                    node {
                        id
                        user {
                            username
                            displayName
                            avatarUrl
                        }
                        activityType
                        entity {
                            ... on Track {
                                id
                                title
                                artist
                            }
                        }
                        timestamp
                    }
                    cursor
                }
                pageInfo {
                    hasNextPage
                    endCursor
                }
            }
        }
    `);
    
    const activities = data?.feed?.edges?.map(edge => edge.node) || [];
    
    return (
        <div>
            <Typography variant="h5">Activity Feed</Typography>
            {activities.map(activity => (
                <ActivityCard key={activity.id} activity={activity} />
            ))}
            {data?.feed?.pageInfo?.hasNextPage && (
                <Button onClick={() => fetchMore({ after: data.feed.pageInfo.endCursor })}>
                    Load More
                </Button>
            )}
        </div>
    );
};

const ActivityCard = ({ activity }) => {
    return (
        <Card style={{ marginBottom: 16 }}>
            <CardContent>
                <div style={{ display: 'flex', alignItems: 'center' }}>
                    <Avatar src={activity.user.avatarUrl} />
                    <div style={{ marginLeft: 12 }}>
                        <Typography variant="body1">
                            <strong>{activity.user.displayName}</strong> {getActivityText(activity)}
                        </Typography>
                        <Typography variant="caption" color="textSecondary">
                            {formatTimestamp(activity.timestamp)}
                        </Typography>
                    </div>
                </div>
            </CardContent>
        </Card>
    );
};

function getActivityText(activity) {
    switch (activity.activityType) {
        case 'LISTEN':
            return `listened to "${activity.entity.title}" by ${activity.entity.artist}`;
        case 'LIKE':
            return `liked "${activity.entity.title}"`;
        case 'FOLLOW':
            return `followed ${activity.entity.username}`;
        default:
            return 'did something';
    }
}
```

**3. Like Button** (`ui/src/song/LikeButton.jsx`)

```tsx
export const LikeButton = ({ trackId }) => {
    const [isLiked, setIsLiked] = useState(false);
    const [likeCount, setLikeCount] = useState(0);
    
    const handleLike = async () => {
        if (isLiked) {
            await client.mutate({
                mutation: UNLIKE_MUTATION,
                variables: { entityType: 'TRACK', entityId: trackId },
            });
            setIsLiked(false);
            setLikeCount(likeCount - 1);
        } else {
            await client.mutate({
                mutation: LIKE_MUTATION,
                variables: { entityType: 'TRACK', entityId: trackId },
            });
            setIsLiked(true);
            setLikeCount(likeCount + 1);
        }
    };
    
    return (
        <IconButton onClick={handleLike} color={isLiked ? 'secondary' : 'default'}>
            {isLiked ? <FavoriteIcon /> : <FavoriteBorderIcon />}
            {likeCount > 0 && <Typography variant="caption">{likeCount}</Typography>}
        </IconButton>
    );
};
```

**Tasks:**
- [ ] Create GraphQL client hook (`useGraphQL`)
- [ ] Create Profile Page component
- [ ] Create Activity Feed component
- [ ] Create Like Button component
- [ ] Add routes to React Router (`/profile/:userId`, `/feed`)
- [ ] Add navigation links (sidebar)
- [ ] Test end-to-end (follow user, like track, see in feed)

**Deliverable:** Social UI integrated into Navidrome

**Acceptance Criteria:**
- Can view user profiles (follower/following counts)
- Can follow/unfollow users (button works, counts update)
- Can like tracks/albums/playlists (button works, like count updates)
- Can see activity feed (friends' listens and likes)
- UI is responsive (works on mobile)

---

## Operational Needs

### Infrastructure
- PostgreSQL 15+ (managed service recommended: AWS RDS, Google Cloud SQL)
- Redis 7+ (managed service: ElastiCache, MemoryStore)
- NATS Server 2.10+ (can run on Kubernetes or separate VM)
- Social Service deployment (Docker container, scaled horizontally)

### Monitoring
- Prometheus metrics for Social Service:
  - `social_graphql_requests_total` (counter)
  - `social_graphql_latency_seconds` (histogram)
  - `social_db_connections_active` (gauge)
  - `social_feed_cache_hits_total` (counter)
  - `social_nats_messages_received_total` (counter)
- Grafana dashboard for Social Service health
- Alerts for:
  - GraphQL P95 latency >500ms
  - NATS message processing lag >10s
  - Redis cache hit rate <80%

### Data Retention
- Activities older than 1 year (soft delete, archive to S3)
- Deleted users (soft delete for 30 days, then hard delete)

---

## Risks & Mitigations

### Risk: Social Service downtime impacts core streaming
**Likelihood:** Low  
**Impact:** Medium  
**Mitigation:**
- Social features are non-blocking (core streaming works without social)
- UI gracefully degrades (hide social features if API unavailable)
- NATS buffer events (up to 1 hour) if Social Service is down

### Risk: Feed generation is slow (N+1 queries)
**Likelihood:** Medium  
**Impact:** High  
**Mitigation:**
- Use materialized view for feed (pre-computed)
- Cache feeds in Redis (5-minute TTL)
- Use GraphQL dataloaders (batch entity fetches)

### Risk: Privacy violations (see private activity)
**Likelihood:** Low  
**Impact:** High  
**Mitigation:**
- Privacy checks in every resolver (can't query private profiles)
- Privacy checks in NATS consumer (don't create activities for private users)
- Security audit before launch
- Add integration tests for privacy (private user shouldn't appear in feed)

---

## Definition of Done

### Must Have
- ✅ Social Service deployed and running
- ✅ PostgreSQL schema created (users, follows, likes, activities)
- ✅ NATS event sync working (Navidrome → Social Service)
- ✅ GraphQL API implemented (queries and mutations)
- ✅ UI components (profile page, activity feed, like button)
- ✅ Privacy controls (profile visibility, activity visibility)
- ✅ Follow/unfollow working end-to-end
- ✅ Like/unlike working end-to-end
- ✅ Activity feed shows friends' listens

### Should Have
- ✅ Feed caching in Redis
- ✅ Privacy checks in all resolvers
- ✅ Graceful degradation (UI works if social API is down)
- ✅ Monitoring and alerts

### Could Have
- Push notifications (new follower, new like)
- User search (autocomplete for @mentions)
- Block/mute users

---

## Success Metrics

- **Social Adoption:** >40% of users follow at least 1 person
- **Engagement:** >20% of users like at least 1 track per week
- **Feed Usage:** >30% of users view feed at least once per week
- **API Latency:** P95 <300ms for GraphQL queries
- **Uptime:** 99.5% (social features available)

---

## Next Steps (Transition to Phase 4)

After Phase 3 is complete:
1. Monitor social engagement metrics (2 weeks)
2. Collect user feedback on social features
3. Create Phase 4 branch: `feature/moments-layer`
4. Begin work on timestamped comments (Moments)

---

**Estimated Effort:**
- Senior Backend Engineer (Go): 100 hours (Social Service, NATS, GraphQL)
- Mid-Level Backend Engineer (Go): 60 hours (repositories, privacy logic)
- Senior Frontend Engineer (React): 80 hours (profile page, feed, like button)
- Mid-Level Frontend Engineer (React): 60 hours (UI polish, responsiveness)
- DevOps Engineer: 40 hours (PostgreSQL, Redis, NATS, deployment)
- **Total: 340 hours (~8.5 person-weeks with 40-hour weeks)**

---

**End of Phase 3**
