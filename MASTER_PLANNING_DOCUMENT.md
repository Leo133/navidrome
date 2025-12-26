# Social-First Music Player + Deep Listening Analytics
## Planning & Design Document (Implementation Roadmap)

**Document Version:** 1.0  
**Date:** January 2025  
**Project:** Navidrome Fork - Social & Analytics Platform  
**Status:** Planning Complete - Ready for Implementation

---

## Executive Summary

This document provides a **comprehensive implementation roadmap** for building a **social-first music player with deep listening analytics** on top of Navidrome (open-source music streaming server). The platform will enable:

- **Social Discovery:** Follow friends, see what they're listening to, share favorite tracks
- **Deep Analytics:** Understand how users listen (completion %, skip patterns, replay hotspots)
- **Timestamped Engagement:** Add comments/reactions at specific moments in tracks (SoundCloud-style)
- **Intelligent Recommendations:** Personalized music discovery using collaborative filtering and ML

**Key Decisions:**
- **Architecture:** Hybrid approach - minimal Navidrome changes (~500-800 LoC) for telemetry ingestion + external services for heavy computation
- **Tech Stack:** Go (Navidrome/core), Python (ML/recommendations), PostgreSQL (social), TimescaleDB (analytics), React (UI)
- **Implementation:** 8 phases over ~6-8 months with clear deliverables and Definition of Done for each phase

---

## Table of Contents

1. [Project Goals & Vision](#project-goals--vision)
2. [Repository Architecture Snapshot](#repository-architecture-snapshot)
3. [Integration Pattern Decision](#integration-pattern-decision)
4. [Target System Architecture](#target-system-architecture)
5. [Implementation Phases (0-7)](#implementation-phases-0-7)
6. [Technology Stack](#technology-stack)
7. [Risk Assessment & Mitigations](#risk-assessment--mitigations)
8. [Success Metrics](#success-metrics)
9. [Resource Requirements](#resource-requirements)
10. [Timeline & Milestones](#timeline--milestones)

---

## Project Goals & Vision

### Vision Statement

> "Create a self-hosted music platform that combines the **personal ownership of Navidrome** with the **social discovery of SoundCloud** and the **intelligent recommendations of Spotify**, while providing **deep listening analytics** that help users understand their musical journey."

### Core Objectives

1. **Social-First Experience**
   - Enable organic music discovery through social connections
   - Replicate SoundCloud's timestamped commenting ("Moments")
   - Build social graph (follows, likes, activity feeds)

2. **Deep Listening Analytics**
   - Capture detailed playback telemetry (not just "played" timestamps)
   - Compute derived metrics (completion %, skip patterns, replay hotspots)
   - Provide personal listening insights dashboard

3. **Intelligent Recommendations**
   - Collaborative filtering (find users with similar taste)
   - Content-based filtering (similar tracks by metadata)
   - Friend-based discovery (what are friends listening to)
   - Weekly personalized playlists ("Discover Weekly")

4. **Self-Hosted & Privacy-Focused**
   - Run on personal infrastructure (not SaaS)
   - GDPR-compliant (data export, deletion, consent management)
   - Transparent telemetry (users control what's tracked)

---

## Repository Architecture Snapshot

### Navidrome Codebase Analysis

**Repository:** [navidrome/navidrome](https://github.com/navidrome/navidrome) (Go 1.25+)

**Key Components Identified:**

1. **Authentication & Session Management**
   - JWT-based authentication ([server/auth/auth.go](server/auth/auth.go))
   - Context propagation for user/player ([server/subsonic/middlewares.go](server/subsonic/middlewares.go))
   - Password hashing with bcrypt

2. **Play Tracking System**
   - **PlayTracker singleton** ([core/scrobbler/play_tracker.go](core/scrobbler/play_tracker.go:58-76))
     - Central telemetry point for all playback events
     - Handles now-playing state and scrobble submission
     - Currently only records completion timestamps (no interval data)
   - **Scrobbles table** (added Dec 2024): stores completion events only
   - **External scrobbler integration** (Last.fm, ListenBrainz)

3. **API Surfaces**
   - **Subsonic API** ([server/subsonic/api.go](server/subsonic/api.go)): XML/JSON, 95% compatible
   - **Native REST API** ([server/nativeapi/native_api.go](server/nativeapi/native_api.go)): JSON-only, modern
   - **Public share endpoints** ([server/public/public.go](server/public/public.go)): Unauthenticated access
   - **Server-Sent Events** ([server/events/broker.go](server/events/broker.go)): Real-time updates

4. **Database Layer**
   - **SQLite** (single-file DB, simple deployment)
   - **80+ migrations** ([db/migrations/](db/migrations/)): Schema evolution
   - **Key tables:**
     - `media_file`: Track metadata (title, artist, album, duration, genre, BPM)
     - `album`, `artist`: Aggregated views
     - `annotation`: User interactions (starred, ratings, play counts)
     - `scrobbles`: New table (Dec 2024) for completion timestamps
     - `user`, `player`, `playlist`, `share`: Core entities
   - **Repositories** ([persistence/](persistence/)): SQL access layer (squirrel query builder)

5. **Streaming Pipeline**
   - **Media Streamer** ([core/media_streamer.go](core/media_streamer.go)): Direct file serving or transcoding
   - **FFmpeg integration** ([core/ffmpeg/ffmpeg.go](core/ffmpeg/ffmpeg.go)): On-the-fly transcoding
   - **Range request support** (seek to arbitrary positions)

6. **Plugin System**
   - **WASM-based** ([plugins/](plugins/)): Sandboxed extensions
   - **Capabilities:** MetadataAgent, Scrobbler, PlaybackTracker
   - **Limitation:** Performance overhead for high-frequency telemetry

7. **UI Layer**
   - **React 17** ([ui/src/](ui/src/)): Material-UI components
   - **Redux state management**
   - **Custom audio player** ([ui/src/audioplayer/](ui/src/audioplayer/)): HTML5 Audio API

### Extension Points for Telemetry

**Primary Extension Point:** PlayTracker ([core/scrobbler/play_tracker.go](core/scrobbler/play_tracker.go))

```go
// Current implementation (simplified)
type playTracker struct {
    nowPlaying   *NowPlayingInfo
    lastScrobble time.Time
}

func (p *playTracker) NowPlaying(ctx context.Context, info NowPlayingInfo) {
    p.nowPlaying = &info
    // No detailed telemetry, just stores current state
}

func (p *playTracker) Scrobble(ctx context.Context, scrobble Scrobble) {
    // Only records completion timestamp (no intervals)
    p.ds.Scrobble(ctx).Insert(scrobble)
}
```

**Proposed Enhancement:**
- Add event emission to NATS on every state change (play, pause, seek)
- Keep changes minimal (~50 LoC in PlayTracker)
- External services consume events for heavy computation

---

## Integration Pattern Decision

### Three Patterns Evaluated

#### Option A: Tight Integration (In-Process)
- **Approach:** Embed all features in Navidrome codebase
- **Pros:** Simple deployment, no inter-service communication
- **Cons:** Navidrome bloat, SQLite limitations for analytics, harder to maintain fork
- **Verdict:** ❌ Rejected (maintenance burden, SQLite not suited for analytics)

#### Option B: Overlay (Sidecar Services)
- **Approach:** Navidrome unchanged, sidecar services read SQLite directly
- **Pros:** Zero Navidrome changes, easy to disable features
- **Cons:** Tight coupling to SQLite schema, can't capture detailed telemetry, polling overhead
- **Verdict:** ❌ Rejected (can't get interval data, inefficient polling)

#### Option C: Hybrid (Minimal Core + External Services)
- **Approach:** Minimal Navidrome changes for telemetry ingestion (~500-800 LoC) + external services for analytics/social
- **Pros:** Clean separation of concerns, optimal databases per service, maintainable fork
- **Cons:** More infrastructure (PostgreSQL, TimescaleDB, NATS)
- **Verdict:** ✅ **Selected** (best balance of maintainability and functionality)

### Hybrid Architecture Details

**Navidrome Modifications (~500-800 LoC):**
1. Add `listen_event` table to SQLite (schema migration)
2. Add `ListenEventRepository` interface in `model/datastore.go`
3. Implement repository in `persistence/listen_event_repository.go` (~150 LoC)
4. Add `POST /api/events/listen` endpoint in `server/nativeapi/native_api.go` (~100 LoC)
5. Add NATS publishing in PlayTracker (~50 LoC)
6. Add privacy controls (incognito mode) in user preferences (~50 LoC)
7. Update UI player to emit events (~100 LoC)

**External Services:**
- **Social Service:** PostgreSQL + GraphQL (social graph, follows, likes, moments)
- **Analytics Service:** TimescaleDB + GraphQL (interval calculation, derived metrics)
- **Recommendation Service:** Python + FastAPI (ML recommendations, collaborative filtering)

**Communication:**
- **Navidrome → External Services:** NATS (event-driven, async)
- **UI → External Services:** GraphQL (direct queries for social/analytics data)

---

## Target System Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            User's Browser (UI)                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │   Player     │  │ Social Feed  │  │  Analytics   │  │ Recommended │ │
│  │   (Moments)  │  │  (Activity)  │  │  Dashboard   │  │   For You   │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘ │
└─────────┼──────────────────┼──────────────────┼──────────────────┼────────┘
          │                  │                  │                  │
          ▼                  ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Navidrome (Core)                                │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Native API (REST)                                              │    │
│  │  • POST /api/events/listen  (telemetry ingestion)              │    │
│  │  • GET  /api/user/:id/preferences  (privacy settings)          │    │
│  │  • GET  /api/stream/:id  (media streaming)                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  PlayTracker (core/scrobbler/play_tracker.go)                  │    │
│  │  • Emit events to NATS on play/pause/seek                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  SQLite Database                                                │    │
│  │  • media_file, album, artist (core catalog)                    │    │
│  │  • listen_event (raw events, ~150 bytes each)                  │    │
│  │  • user, player, playlist (core entities)                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │
                            │ NATS (message queue)
                            │ • navidrome.listen.event
                            │ • navidrome.user.created/updated/deleted
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        External Services                                 │
│  ┌───────────────────┐  ┌───────────────────┐  ┌────────────────────┐  │
│  │  Social Service   │  │ Analytics Service │  │ Recommendation Svc │  │
│  │  (Go + GraphQL)   │  │ (Go + GraphQL)    │  │ (Python + FastAPI) │  │
│  │                   │  │                   │  │                    │  │
│  │  Features:        │  │  Features:        │  │  Features:         │  │
│  │  • Follows        │  │  • Intervals      │  │  • Collab Filter   │  │
│  │  • Likes          │  │  • Completion %   │  │  • Content Filter  │  │
│  │  • Moments        │  │  • Skip cliffs    │  │  • Taste Vectors   │  │
│  │  • Activity Feed  │  │  • Replay hotspots│  │  • Discover Weekly │  │
│  │                   │  │  • Session analyze│  │                    │  │
│  └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬──────────┘  │
│            │                      │                        │             │
│            ▼                      ▼                        ▼             │
│  ┌───────────────────┐  ┌───────────────────┐  ┌────────────────────┐  │
│  │   PostgreSQL      │  │   TimescaleDB     │  │      Qdrant        │  │
│  │   (Social Data)   │  │   (Time-Series)   │  │  (Vector Search)   │  │
│  │                   │  │                   │  │                    │  │
│  │  • users          │  │  • raw_events     │  │  • taste_vectors   │  │
│  │  • follows        │  │  • intervals      │  │                    │  │
│  │  • likes          │  │  • track_stats    │  │                    │  │
│  │  • moments        │  │  • sessions       │  │                    │  │
│  │  • activities     │  │                   │  │                    │  │
│  └───────────────────┘  └───────────────────┘  └────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Redis (Caching + Job Queue)                                      │ │
│  │  • Feed cache (5-min TTL)                                         │ │
│  │  • Recommendation cache (7-day TTL)                               │ │
│  │  • Celery task queue (background jobs)                            │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                      Observability Stack                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ Prometheus   │  │   Grafana    │  │    Jaeger    │  │  Loki/ELK  │  │
│  │  (Metrics)   │  │ (Dashboards) │  │   (Traces)   │  │   (Logs)   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Examples

**Example 1: User Listens to Track**

```
1. User clicks "Play" on track
   ↓
2. UI emits event to buffer:
   {
     mediaFileId: "track-123",
     eventType: "play",
     timestamp: 1704067200000,
     position: 0,
     metadata: {sessionId: "session-abc"}
   }
   ↓
3. Buffer flushes to Navidrome (every 10s or 50 events):
   POST /api/events/listen
   ↓
4. Navidrome ingests events:
   • Validates events (timestamp, position)
   • Inserts into SQLite listen_event table
   • Publishes to NATS (navidrome.listen.event)
   ↓
5. External services consume NATS events:
   • Social Service → creates activity (for friends' feeds)
   • Analytics Service → stores raw event for later processing
   ↓
6. Background jobs (Celery, cron):
   • Analytics Service: Calculate intervals (every 5 min)
   • Analytics Service: Compute stats (every 1 hour)
   • Recommendation Service: Update taste vectors (daily)
   • Recommendation Service: Generate Discover Weekly (weekly)
```

**Example 2: User Views Friend's Profile**

```
1. User navigates to /profile/alice
   ↓
2. UI queries Social Service (GraphQL):
   query GetUser($userId: ID!) {
     user(id: $userId) {
       username, displayName, followerCount, followingCount
     }
   }
   ↓
3. Social Service checks privacy:
   • Can viewer see this profile? (public/friends/private)
   • If yes, return user data
   ↓
4. UI renders profile page with follow button
   ↓
5. User clicks "Follow"
   ↓
6. UI calls mutation:
   mutation Follow($userId: ID!) {
     follow(userId: $userId) { id }
   }
   ↓
7. Social Service:
   • Inserts into follows table
   • Creates activity (for activity feed)
   • Invalidates feed cache
```

---

## Implementation Phases (0-7)

### Phase Overview

| Phase | Name | Duration | Team Size | Key Deliverables |
|-------|------|----------|-----------|------------------|
| **Phase 0** | Repo Audit & Upstream Alignment | 2-3 weeks | 3-4 engineers | Development environment, code audit, fork management strategy, CI/CD |
| **Phase 1** | Stable Backend Baseline | 3-4 weeks | 4-5 engineers | Performance baseline, query optimization, observability (Prometheus) |
| **Phase 2** | Telemetry v1 Foundations | 3-4 weeks | 4-5 engineers | listen_event table, event ingestion API, custom player with telemetry |
| **Phase 3** | Social Core v1 | 4-5 weeks | 5-6 engineers | Social Service, social graph (follows/likes), activity feed, privacy controls |
| **Phase 4** | Moment Layer | 3-4 weeks | 4-5 engineers | Timestamped comments, heatmap visualization, reactions |
| **Phase 5** | Analytics Expansion | 4-5 weeks | 4-5 engineers | Analytics Service, interval calculation, derived metrics, insights dashboard |
| **Phase 6** | Recommendations & Discovery | 4-5 weeks | 5-6 engineers | Recommendation Service, taste vectors, collaborative filtering, Discover Weekly |
| **Phase 7** | Hardening & Production | 3-4 weeks | 4-5 engineers | Security audit, GDPR compliance, performance optimization, load testing |

**Total Duration:** ~6-8 months (26-34 weeks)  
**Total Effort:** ~2,000 person-hours (~50 person-weeks)

### Phase 0: Repo Audit & Upstream Alignment

**See:** [PHASE_0_REPO_AUDIT.md](PHASE_0_REPO_AUDIT.md)

**Objectives:**
- Establish development foundation
- Audit Navidrome codebase (identify extension points)
- Setup fork management strategy (rebase workflow, upstream sync)
- Create CI/CD pipeline (test, build, deploy)

**Key Deliverables:**
- ✅ DEVELOPMENT_SETUP.md (Docker Compose, Go, React)
- ✅ ARCHITECTURE_MAP.md (component diagram with file paths)
- ✅ TEST_COVERAGE_REPORT.md (baseline coverage %)
- ✅ FORK_MANAGEMENT_STRATEGY.md (rebase workflow)
- ✅ CI/CD workflows (GitHub Actions: test, build, deploy to staging)
- ✅ DEPENDENCY_ANALYSIS.md (Go modules, npm packages)

**Definition of Done:**
- Dev environment is reproducible (Docker Compose up)
- All tests pass (make test)
- Code coverage baseline documented (>60%)
- Fork can sync with upstream (rebase tested)
- CI/CD pipeline runs on every push

**Estimated Effort:** 120 person-hours (~3 person-weeks)

---

### Phase 1: Stable Backend Baseline

**See:** [PHASE_1_STABLE_BACKEND.md](PHASE_1_STABLE_BACKEND.md)

**Objectives:**
- Establish performance baselines before adding features
- Optimize database queries (indexes, query plans)
- Add observability (Prometheus metrics, Grafana dashboards)
- Harden core system (error handling, config validation)

**Key Deliverables:**
- ✅ PERFORMANCE_BASELINE.md (latency, throughput, resource usage)
- ✅ Query optimization migration (new indexes on annotation, media_file)
- ✅ Prometheus metrics (HTTP requests, DB queries, streaming)
- ✅ Grafana dashboards (system health, API performance)
- ✅ Load test scripts (k6 scenarios)
- ✅ SECURITY_AUDIT_REPORT.md (initial scan)

**Definition of Done:**
- Performance baseline documented (P50/P95/P99 latencies)
- All slow queries optimized (<100ms)
- Prometheus metrics exported
- Grafana dashboard shows key metrics
- Load test validates 100 concurrent users

**Estimated Effort:** 180 person-hours (~4.5 person-weeks)

---

### Phase 2: Telemetry v1 Foundations

**See:** [PHASE_2_TELEMETRY_FOUNDATIONS.md](PHASE_2_TELEMETRY_FOUNDATIONS.md)

**Objectives:**
- Build core telemetry infrastructure to capture detailed playback events
- Add listen_event table (SQLite migration)
- Implement ListenEventRepository (bulk ingestion)
- Create POST /api/events/listen endpoint
- Build custom web player with event emission
- Add privacy controls (incognito mode, device opt-out)

**Key Deliverables:**
- ✅ listen_event table (schema, migration, indexes)
- ✅ ListenEventRepository implementation
- ✅ POST /api/events/listen endpoint (bulk ingestion, validation, rate limiting)
- ✅ Custom web player (event buffer, batch sends)
- ✅ Privacy UI (incognito mode, device opt-out)
- ✅ Listening history page (view raw events)

**Definition of Done:**
- Migration runs successfully
- Events are ingested (<50ms for 50 events)
- Custom player emits events on play/pause/seek/ended
- Privacy controls work (incognito mode suppresses events)
- Staging deployment validated (3 days stable)

**Estimated Effort:** 260 person-hours (~6.5 person-weeks)

---

### Phase 3: Social Core v1

**See:** [PHASE_3_SOCIAL_CORE.md](PHASE_3_SOCIAL_CORE.md)

**Objectives:**
- Build foundational social layer (follows, likes, activity feed)
- Deploy Social Service (PostgreSQL + GraphQL)
- Implement social graph (follow/unfollow, followers/following)
- Add likes on tracks/albums/playlists
- Build activity feed (timeline of friends' listens)
- Add privacy controls (profile visibility, activity visibility)

**Key Deliverables:**
- ✅ Social Service (Go + GraphQL + PostgreSQL)
- ✅ Social graph (users, follows, likes, activities tables)
- ✅ NATS event sync (Navidrome → Social Service)
- ✅ GraphQL API (queries: feed, followers, following; mutations: follow, like)
- ✅ UI integration (profile pages, follow buttons, activity feed)

**Definition of Done:**
- Social Service deployed and running
- Can follow/unfollow users (UI works end-to-end)
- Can like tracks/albums (like count updates)
- Activity feed shows friends' listens
- Privacy controls work (can't see private profiles)

**Estimated Effort:** 340 person-hours (~8.5 person-weeks)

---

### Phase 4: Moment Layer

**See:** [PHASE_4_MOMENT_LAYER.md](PHASE_4_MOMENT_LAYER.md)

**Objectives:**
- Build Moments layer (timestamped comments/reactions on tracks)
- Add moments table to Social Service (PostgreSQL)
- Visualize moments as heatmap on progress bar
- Allow users to add comments at specific timestamps
- Add reactions (emoji) on moments
- Detect hot spots (most-reacted sections)

**Key Deliverables:**
- ✅ Moments table (comments with timestamp_ms, reactions)
- ✅ GraphQL API (createMoment, trackMoments, trackMomentHeatmap, reactToMoment)
- ✅ Moment heatmap visualization (progress bar overlay)
- ✅ Moment creation UI (add moment at current playback position)
- ✅ Moment browser UI (view all moments, seek to moments)
- ✅ Privacy controls (public/friends/private moments)

**Definition of Done:**
- Can create moments with timestamps
- Heatmap shows colored regions for moment clusters
- Can react to moments (emoji reactions)
- Privacy controls work (can't see private moments)

**Estimated Effort:** 240 person-hours (~6 person-weeks)

---

### Phase 5: Analytics Expansion

**See:** [PHASE_5_ANALYTICS_EXPANSION.md](PHASE_5_ANALYTICS_EXPANSION.md)

**Objectives:**
- Build Analytics Service to compute derived metrics from raw events
- Deploy TimescaleDB (time-series database)
- Calculate listened intervals (which portions of tracks were played)
- Compute derived metrics (completion %, skip cliffs, replay hotspots)
- Generate session analytics (binge sessions, skip-heavy sessions)
- Build listening insights dashboard

**Key Deliverables:**
- ✅ Analytics Service (Go + GraphQL + TimescaleDB)
- ✅ TimescaleDB schema (raw_events, listened_intervals, track_stats, listening_sessions)
- ✅ Interval calculation algorithm (background job, every 5 min)
- ✅ Stats calculation (background job, every 1 hour)
- ✅ GraphQL API (myListeningStats, trackAnalytics, popularTracks, listeningTrend)
- ✅ Listening insights dashboard (personal stats, favorite genres, top tracks, listening trend chart)

**Definition of Done:**
- Analytics Service deployed and running
- Intervals are calculated correctly (>95% accuracy)
- Stats are computed hourly (track completion %, skip cliffs)
- Dashboard shows personal stats (accurate data)
- Queries are performant (<500ms)

**Estimated Effort:** 260 person-hours (~6.5 person-weeks)

---

### Phase 6: Recommendations & Discovery

**See:** [PHASE_6_RECOMMENDATIONS.md](PHASE_6_RECOMMENDATIONS.md)

**Objectives:**
- Build intelligent recommendation engine for music discovery
- Deploy Recommendation Service (Python + FastAPI)
- Generate taste vectors (user preference embeddings)
- Implement collaborative filtering (find similar users)
- Implement content-based filtering (similar tracks by metadata)
- Implement friend-based recommendations (what friends are listening to)
- Auto-generate Discover Weekly playlists

**Key Deliverables:**
- ✅ Recommendation Service (Python + FastAPI + Qdrant + Celery)
- ✅ Taste vector generation (background job, daily)
- ✅ Collaborative filtering (similar users, recommend their favorites)
- ✅ Content-based filtering (similar tracks by genre/tempo/year)
- ✅ Friend-based recommendations (friends' recent listens)
- ✅ Hybrid recommender (combine all methods)
- ✅ Discover Weekly playlist generation (background job, weekly)
- ✅ API endpoints (/recommendations/tracks, /similar-tracks)
- ✅ UI integration (Recommended For You, Similar Tracks, Discover Weekly)

**Definition of Done:**
- Recommendation Service deployed and running
- Taste vectors generated for all users
- Recommendations are diverse and high-quality (>60% user satisfaction)
- Discover Weekly playlists generated every Monday
- UI shows recommendations (Recommended For You page)

**Estimated Effort:** 300 person-hours (~7.5 person-weeks)

---

### Phase 7: Hardening & Production Readiness

**See:** [PHASE_7_HARDENING.md](PHASE_7_HARDENING.md)

**Objectives:**
- Prepare system for production deployment at scale
- Security hardening (rate limiting, CSRF protection, input validation)
- Privacy enforcement (GDPR compliance, data export/delete, consent)
- Performance optimization (caching, query tuning, horizontal scaling)
- Reliability improvements (error handling, retries, circuit breakers)
- Observability enhancement (structured logging, distributed tracing)
- Moderation tools (flag/review moments, block users)

**Key Deliverables:**
- ✅ Security audit complete (no critical/high vulnerabilities)
- ✅ Rate limiting (login, API endpoints)
- ✅ CSRF protection
- ✅ GDPR compliance (consent management, data export/delete CLI commands)
- ✅ Data retention policies (auto-purge old events)
- ✅ Performance optimization (caching, query optimization, horizontal scaling)
- ✅ Observability (structured logging, distributed tracing with Jaeger, comprehensive Prometheus metrics)
- ✅ Moderation tools (admin dashboard for flagged moments)
- ✅ Load testing (validated for 100-500 concurrent users)
- ✅ Documentation (DEPLOYMENT.md, OPERATIONS.md, API_REFERENCE.md)

**Definition of Done:**
- Security scan shows no critical/high vulnerabilities
- GDPR compliance validated (data export/delete work)
- System handles 100 concurrent users (P95 <500ms)
- Horizontal scaling tested (3 instances handle 3x load)
- Documentation complete (deployment, operations, API)
- Production deployment successful

**Estimated Effort:** 260 person-hours (~6.5 person-weeks)

---

## Technology Stack

### Core Platform (Navidrome)
- **Language:** Go 1.25+
- **Database:** SQLite 3.x
- **API:** REST (chi router)
- **Authentication:** JWT (HS256)
- **UI:** React 17 + Material-UI + Redux

### External Services

#### Social Service
- **Language:** Go 1.25+
- **Database:** PostgreSQL 15+
- **API:** GraphQL (gqlgen)
- **Cache:** Redis 7+
- **Message Queue:** NATS 2.10+

#### Analytics Service
- **Language:** Go 1.25+
- **Database:** TimescaleDB (PostgreSQL + time-series extensions)
- **API:** GraphQL (gqlgen)
- **Background Jobs:** Cron (robfig/cron)

#### Recommendation Service
- **Language:** Python 3.11+
- **Framework:** FastAPI
- **Vector Database:** Qdrant (similarity search)
- **Database:** PostgreSQL (recommendation cache)
- **Job Queue:** Celery + Redis
- **ML Libraries:** scikit-learn, pandas, numpy

### Infrastructure & Operations
- **Containerization:** Docker + Docker Compose
- **Orchestration:** Kubernetes (optional, for production)
- **Monitoring:** Prometheus + Grafana
- **Logging:** Loki (or Elasticsearch + Kibana)
- **Tracing:** Jaeger (OpenTracing)
- **Load Balancer:** Nginx
- **CI/CD:** GitHub Actions

---

## Risk Assessment & Mitigations

### Critical Risks

#### Risk: Fork Maintenance Burden
**Likelihood:** High  
**Impact:** High  
**Description:** Keeping fork in sync with upstream Navidrome becomes too time-consuming

**Mitigations:**
- Minimize Navidrome changes (~500-800 LoC, isolated to telemetry)
- Use feature flags (can disable telemetry if upstream conflicts)
- Automated rebase workflow (GitHub Actions test upstream merge weekly)
- Clear documentation of modified files (easier to resolve conflicts)
- Active monitoring of upstream releases (review changelogs, plan rebases)

---

#### Risk: Data Loss (Telemetry/Analytics)
**Likelihood:** Low  
**Impact:** High  
**Description:** Database corruption, backup failure, or retention policy bug causes permanent data loss

**Mitigations:**
- Automated daily backups (all databases: SQLite, PostgreSQL, TimescaleDB)
- Backup validation (restore test monthly)
- Retention policies tested in staging (verify purge doesn't delete wrong data)
- Disaster recovery plan (RTO: 4 hours, RPO: 24 hours)
- Data export capability (users can download their data before deletion)

---

#### Risk: Privacy Violations (GDPR Non-Compliance)
**Likelihood:** Low  
**Impact:** Critical  
**Description:** System violates GDPR (no consent, can't export/delete data, unauthorized data sharing)

**Mitigations:**
- Consent management (users opt-in to telemetry/analytics/social)
- Privacy-by-default (incognito mode, private profiles)
- Data export/delete (CLI commands tested)
- Security audit (external review before production)
- Clear privacy policy (what data is collected, how it's used, who can see it)
- Privacy checks in every resolver (can't query private data without permission)

---

### High Risks

#### Risk: Performance Degradation
**Likelihood:** Medium  
**Impact:** High  
**Description:** Telemetry ingestion or analytics queries slow down core streaming

**Mitigations:**
- Async event ingestion (don't block streaming requests)
- Separate DB connection pools (telemetry writes don't impact catalog reads)
- Bulk inserts (reduce transaction overhead)
- Caching (Redis for hot data: feeds, recommendations)
- Query optimization (indexes, covering indexes, continuous aggregates)
- Load testing (validate performance under expected load before production)

---

#### Risk: Recommendation Quality
**Likelihood:** Medium  
**Impact:** Medium  
**Description:** Recommendations are poor (low diversity, irrelevant tracks)

**Mitigations:**
- Hybrid approach (combine collaborative + content + friend-based)
- Diversity penalty (penalize too many tracks from same artist)
- A/B testing (test different recommendation weights)
- User feedback loop ("Was this helpful?" thumbs up/down)
- Fallback to popular tracks (if user has insufficient data)

---

## Success Metrics

### Phase-Specific Metrics

**Phase 2 (Telemetry):**
- Event capture rate: >95% (of playback actions result in recorded events)
- Ingestion latency: P95 <50ms (bulk insert of 50 events)
- User adoption: >80% leave telemetry enabled (default on)

**Phase 3 (Social):**
- Social adoption: >40% follow at least 1 person
- Engagement: >20% like at least 1 track per week
- Feed usage: >30% view feed at least once per week

**Phase 4 (Moments):**
- Moment adoption: >15% create at least 1 moment
- Engagement: >25% view moments on tracks
- Reaction rate: >30% of moments receive at least 1 reaction

**Phase 5 (Analytics):**
- Analytics accuracy: >95% (interval calculation validated manually)
- Query performance: P95 <500ms (analytics queries)
- Dashboard engagement: >40% view analytics dashboard at least once per month

**Phase 6 (Recommendations):**
- Recommendation adoption: >50% click on at least 1 recommendation per week
- Discover Weekly engagement: >30% play Discover Weekly playlist
- Recommendation quality: >60% user satisfaction (based on feedback)

**Phase 7 (Production Readiness):**
- Uptime: >99.5% (production)
- Security: 0 critical/high vulnerabilities
- Performance: P95 latency <500ms (under normal load)
- Data loss: 0% (backups + disaster recovery)

### Overall Platform Metrics (Post-Launch)

**User Engagement:**
- Monthly Active Users (MAU): Target 80% of registered users
- Daily Active Users (DAU): Target 40% of MAU
- Session duration: Target >30 minutes per session
- Return rate: Target >60% return within 7 days

**Feature Adoption:**
- Social features: >50% of users follow at least 1 person
- Moments: >20% of users create at least 1 moment
- Recommendations: >60% of users click on recommendations
- Analytics: >50% of users view analytics dashboard

**Technical Health:**
- Uptime: >99.9% (excluding planned maintenance)
- API latency: P95 <500ms, P99 <1s
- Error rate: <0.1% (HTTP 5xx)
- Data accuracy: >99% (telemetry, intervals, stats)

---

## Resource Requirements

### Team Composition

**Core Team (6-8 engineers):**
- 2x Senior Backend Engineers (Go): Navidrome modifications, Social/Analytics Services
- 1x Mid-Level Backend Engineer (Go): Repositories, testing, deployment
- 1x Senior Frontend Engineer (React): Player, social UI, analytics dashboard
- 1x Mid-Level Frontend Engineer (React): UI components, polish, responsiveness
- 1x ML Engineer (Python): Recommendation engine, taste vectors, collaborative filtering
- 1x DevOps Engineer: Infrastructure, CI/CD, monitoring, deployment
- 1x Technical Writer (part-time): Documentation (deployment, operations, API)

**Advisors/Reviewers (part-time):**
- 1x Security Engineer: Security audits, penetration testing (Phase 7)
- 1x Legal/Privacy Expert: GDPR compliance review (Phase 7)

### Infrastructure

**Development/Staging:**
- 3x VMs (4 CPU, 16GB RAM each): Navidrome, Social, Analytics
- 1x VM (2 CPU, 8GB RAM): PostgreSQL + TimescaleDB + Redis + NATS
- 1x VM (2 CPU, 8GB RAM): Monitoring (Prometheus, Grafana, Jaeger)

**Production (Target: 1000 users):**
- 3x VMs (8 CPU, 32GB RAM each): Navidrome (load balanced)
- 2x VMs (4 CPU, 16GB RAM each): Social Service (load balanced)
- 2x VMs (4 CPU, 16GB RAM each): Analytics Service (load balanced)
- 1x VM (2 CPU, 8GB RAM): Recommendation Service (single instance, cron jobs)
- 1x Managed PostgreSQL (4 CPU, 16GB RAM): Social data
- 1x Managed TimescaleDB (4 CPU, 16GB RAM): Analytics data
- 1x Managed Redis (2GB memory): Caching + job queue
- 1x NATS cluster (3 nodes, 2 CPU, 4GB RAM each): Message queue
- 1x Load Balancer: Nginx (or managed service like AWS ALB)
- 1x Monitoring VM (4 CPU, 8GB RAM): Prometheus, Grafana, Jaeger, Loki

**Estimated Cloud Costs (AWS):**
- Development/Staging: ~$300/month
- Production (1000 users): ~$1,200/month
- **Note:** Can reduce costs with reserved instances, spot instances, or self-hosting

---

## Timeline & Milestones

### Gantt Chart (High-Level)

```
Month 1   Month 2   Month 3   Month 4   Month 5   Month 6   Month 7   Month 8
|---------|---------|---------|---------|---------|---------|---------|---------|
Phase 0: Repo Audit & Upstream Alignment
[██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]

Phase 1: Stable Backend Baseline
[░░░░░░░░░░██████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]

Phase 2: Telemetry v1 Foundations
[░░░░░░░░░░░░░░░░░░░░░░░░██████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░]

Phase 3: Social Core v1
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████████████░░░░░░░░░░]

Phase 4: Moment Layer
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████░░]

Phase 5: Analytics Expansion
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████████░░]

Phase 6: Recommendations & Discovery
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████████░░]

Phase 7: Hardening & Production
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████░░]

Production Launch                                                                 ↑
                                                                              Month 7-8
```

### Key Milestones

**M1: Development Foundation Complete (Week 3)**
- Deliverable: Phase 0 complete
- Success Criteria: Dev environment reproducible, CI/CD pipeline running, code audit complete

**M2: Performance Baseline Established (Week 7)**
- Deliverable: Phase 1 complete
- Success Criteria: Performance baseline documented, Prometheus metrics exported, load testing validated

**M3: Telemetry Operational (Week 11)**
- Deliverable: Phase 2 complete
- Success Criteria: Events are ingested (<50ms), custom player emits events, staging deployment stable

**M4: Social Features Live (Week 16)**
- Deliverable: Phase 3 complete
- Success Criteria: Can follow/unfollow users, activity feed works, privacy controls enforced

**M5: Moments Layer Deployed (Week 20)**
- Deliverable: Phase 4 complete
- Success Criteria: Can create moments, heatmap shows moment clusters, reactions work

**M6: Analytics Dashboard Live (Week 25)**
- Deliverable: Phase 5 complete
- Success Criteria: Intervals calculated correctly, stats computed hourly, dashboard shows personal insights

**M7: Recommendations Available (Week 30)**
- Deliverable: Phase 6 complete
- Success Criteria: Taste vectors generated, recommendations are high-quality (>60% satisfaction), Discover Weekly playlists created

**M8: Production Ready (Week 34)**
- Deliverable: Phase 7 complete
- Success Criteria: Security audit passed, GDPR compliance validated, load testing successful, documentation complete

**M9: Production Launch (Week 34+)**
- Deliverable: System deployed to production
- Success Criteria: Uptime >99.5%, no critical issues, user feedback positive

---

## Appendices

### Appendix A: Event Schema (Listen Events)

**ListenEvent:**

```json
{
  "id": 12345,
  "userId": "user-abc123",
  "playerId": "player-xyz789",
  "mediaFileId": "track-123456",
  "eventType": "play",  // 'play', 'pause', 'resume', 'seek', 'ended', 'error'
  "timestamp": 1704067200000,  // Unix timestamp (milliseconds)
  "position": 0,  // Playback position (milliseconds)
  "metadata": {
    "sessionId": "session-abc123",
    "clientVersion": "1.0.0",
    "incognito": false,
    "previousPosition": 0,  // For seek events
    "errorMessage": null    // For error events
  },
  "createdAt": "2025-01-01T00:00:00Z"
}
```

**Event Types:**
- **play:** User started playback (position reset to 0 or specific position)
- **pause:** User paused playback (position captured)
- **resume:** User resumed playback (position captured)
- **seek:** User seeked to different position (previous and new positions captured)
- **ended:** Track ended (position = track duration)
- **error:** Playback error occurred (error message captured)

**Size Estimation:**
- Event size: ~150 bytes (JSON compressed)
- Assumptions:
  - 100 users, 10 tracks/day/user, 5 events/track
  - Total: 5,000 events/day = 750 KB/day = 23 MB/month = 274 MB/year
- For 1,000 users: 2.74 GB/year (manageable for SQLite)
- For 10,000 users: 27.4 GB/year (approaching SQLite limits, consider archival)

---

### Appendix B: Analytics Storage Approach

**Raw Events → Derived Metrics Pipeline:**

```
Raw Events (listen_event table in Navidrome SQLite)
  ↓ [NATS: navidrome.listen.event]
Analytics Service (TimescaleDB)
  ↓ [Insert into raw_events hypertable]
Background Job (every 5 minutes)
  ↓ [Calculate listened intervals]
  ↓ [Merge overlapping intervals]
  ↓ [Insert into listened_intervals hypertable]
Background Job (every 1 hour)
  ↓ [Aggregate intervals by track]
  ↓ [Compute stats: listen_count, completion_count, skip_count, avg_completion_pct]
  ↓ [Insert into track_stats hypertable]
Continuous Aggregate (TimescaleDB)
  ↓ [Daily rollup of track_stats]
  ↓ [Materialized view: track_stats_daily]
GraphQL API
  ↓ [Query materialized views for fast dashboard queries]
```

**TimescaleDB Schema:**

```sql
-- Raw events (1:1 copy from Navidrome)
CREATE TABLE raw_events (
    id              BIGSERIAL,
    user_id         VARCHAR(255) NOT NULL,
    player_id       VARCHAR(255) NOT NULL,
    media_file_id   VARCHAR(255) NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    position        INTEGER NOT NULL,  -- milliseconds
    metadata        JSONB,
    PRIMARY KEY (timestamp, id)
);
SELECT create_hypertable('raw_events', 'timestamp', chunk_time_interval => INTERVAL '1 month');

-- Derived: Listened intervals (computed from events)
CREATE TABLE listened_intervals (
    id              BIGSERIAL,
    user_id         VARCHAR(255) NOT NULL,
    media_file_id   VARCHAR(255) NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    start_position  INTEGER NOT NULL,  -- milliseconds
    end_position    INTEGER NOT NULL,  -- milliseconds
    duration_ms     INTEGER NOT NULL,
    PRIMARY KEY (start_time, id)
);
SELECT create_hypertable('listened_intervals', 'start_time', chunk_time_interval => INTERVAL '1 month');

-- Derived: Track stats (aggregated from intervals)
CREATE TABLE track_stats (
    id                  BIGSERIAL,
    media_file_id       VARCHAR(255) NOT NULL,
    time_bucket         TIMESTAMPTZ NOT NULL,  -- Hourly bucket
    listen_count        INTEGER DEFAULT 0,
    completion_count    INTEGER DEFAULT 0,
    skip_count          INTEGER DEFAULT 0,
    avg_completion_pct  FLOAT DEFAULT 0,
    total_listen_time   INTEGER DEFAULT 0,
    unique_listeners    INTEGER DEFAULT 0,
    PRIMARY KEY (time_bucket, media_file_id)
);
SELECT create_hypertable('track_stats', 'time_bucket', chunk_time_interval => INTERVAL '1 week');

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
    SUM(total_listen_time) AS total_listen_time
FROM track_stats
GROUP BY media_file_id, day;
```

**Retention & Compression:**
- **Raw events:** Compressed after 7 days, deleted after 2 years
- **Intervals:** Compressed after 7 days, deleted after 2 years
- **Track stats:** Compressed after 30 days, kept for 5 years
- **Daily aggregates:** Never deleted (small data size)

**Query Performance:**
- **Dashboard query (user's stats for last 30 days):** <100ms (query intervals, not raw events)
- **Track analytics (completion % for track):** <50ms (query track_stats_daily materialized view)
- **Popular tracks (top 50 tracks this week):** <200ms (query track_stats with time filter)

---

### Appendix C: Recommendation Algorithm Details

**Taste Vector Dimensions (100 total):**

```python
Taste Vector (100 dimensions):
  [0-19]:   Genre preferences (rock=0.8, jazz=0.2, ...)
  [20-39]:  Artist preferences (Queen=0.9, Beatles=0.7, ...) [hashed]
  [40-59]:  Tempo preferences (fast=0.6, slow=0.4, ...)
  [60-79]:  Year preferences (70s=0.7, 80s=0.5, ...)
  [80-99]:  Mood preferences (energetic=0.8, chill=0.3, ...)
```

**Collaborative Filtering Algorithm:**

```python
def recommend_tracks_collaborative(user_id, limit=50):
    # 1. Get user's taste vector
    user_vector = qdrant.get_vector(user_id)
    
    # 2. Find similar users (cosine similarity)
    similar_users = qdrant.search(
        vector=user_vector,
        limit=50,
        score_threshold=0.5  # Minimum similarity
    )
    
    # 3. Get user's already-listened tracks (don't recommend duplicates)
    user_listened = analytics.get_user_listened_tracks(user_id)
    
    # 4. Aggregate recommendations from similar users (weighted by similarity)
    track_scores = {}
    for similar_user_id, similarity in similar_users:
        similar_user_tracks = analytics.get_user_top_tracks(similar_user_id, limit=100)
        
        for track_id, listen_count in similar_user_tracks.items():
            if track_id not in user_listened:
                score = similarity * listen_count
                track_scores[track_id] = track_scores.get(track_id, 0) + score
    
    # 5. Sort by score and return top N
    recommended = sorted(track_scores.items(), key=lambda x: x[1], reverse=True)
    return [track_id for track_id, score in recommended[:limit]]
```

**Hybrid Recommendation Algorithm:**

```python
def recommend_tracks_hybrid(user_id, limit=50):
    # Get recommendations from each method
    collab_recs = recommend_tracks_collaborative(user_id, limit=100)
    content_recs = recommend_tracks_content_based(user_id, limit=100)
    friend_recs = recommend_tracks_from_friends(user_id, limit=100)
    
    # Assign weights (tunable via A/B testing)
    weights = {
        "collaborative": 0.4,
        "content": 0.3,
        "friend": 0.3
    }
    
    # Combine scores (higher rank = higher score)
    track_scores = {}
    
    for i, track_id in enumerate(collab_recs):
        score = weights["collaborative"] * (100 - i)
        track_scores[track_id] = track_scores.get(track_id, 0) + score
    
    for i, track_id in enumerate(content_recs):
        score = weights["content"] * (100 - i)
        track_scores[track_id] = track_scores.get(track_id, 0) + score
    
    for i, track_id in enumerate(friend_recs):
        score = weights["friend"] * (100 - i)
        track_scores[track_id] = track_scores.get(track_id, 0) + score
    
    # Sort by final score
    recommended = sorted(track_scores.items(), key=lambda x: x[1], reverse=True)
    
    # Apply diversity penalty (penalize too many tracks from same artist)
    diversified = apply_diversity_penalty(recommended, max_per_artist=3)
    
    return [track_id for track_id, score in diversified[:limit]]
```

**Cold Start Problem Mitigation:**

```python
def recommend_tracks_for_new_user(user_id, limit=50):
    # New user with no listening history
    
    # Option 1: Popular tracks (globally most listened)
    popular = analytics.get_popular_tracks(time_range='week', limit=50)
    
    # Option 2: Genre-based (if user specified genres during signup)
    user_genres = get_user_preferred_genres(user_id)
    if user_genres:
        genre_tracks = analytics.get_popular_tracks_by_genres(user_genres, limit=50)
        return genre_tracks
    
    # Fallback to popular tracks
    return popular
```

---

## Conclusion

This document provides a **comprehensive roadmap** for building a social-first music player with deep listening analytics on top of Navidrome. The 8-phase implementation plan balances **incremental feature delivery** with **production-ready quality**, ensuring each phase has clear deliverables, Definition of Done, and success metrics.

**Key Takeaways:**
1. **Hybrid architecture** minimizes Navidrome changes (~500-800 LoC) while enabling powerful external services
2. **8 phases over 6-8 months** provide clear milestones and checkpoints
3. **~2,000 person-hours** (~50 person-weeks) estimated with 6-8 engineer team
4. **Comprehensive risk mitigations** for fork maintenance, data loss, privacy violations, and performance
5. **Clear success metrics** for each phase (telemetry capture rate, social adoption, recommendation quality, etc.)

**Next Steps:**
1. Review planning documents with stakeholders
2. Assemble team (6-8 engineers + advisors)
3. Setup infrastructure (development/staging environments)
4. Begin Phase 0 (Repo Audit & Upstream Alignment)

---

**End of Planning Document**

**Status:** ✅ Planning Complete - Ready for Implementation

---

## Related Documents

- [ARCHITECTURE_SNAPSHOT.md](ARCHITECTURE_SNAPSHOT.md) - Detailed Navidrome codebase analysis
- [INTEGRATION_PATTERN_DECISION.md](INTEGRATION_PATTERN_DECISION.md) - Architecture pattern comparison and selection
- [PHASE_0_REPO_AUDIT.md](PHASE_0_REPO_AUDIT.md) - Phase 0 implementation details
- [PHASE_1_STABLE_BACKEND.md](PHASE_1_STABLE_BACKEND.md) - Phase 1 implementation details
- [PHASE_2_TELEMETRY_FOUNDATIONS.md](PHASE_2_TELEMETRY_FOUNDATIONS.md) - Phase 2 implementation details
- [PHASE_3_SOCIAL_CORE.md](PHASE_3_SOCIAL_CORE.md) - Phase 3 implementation details
- [PHASE_4_MOMENT_LAYER.md](PHASE_4_MOMENT_LAYER.md) - Phase 4 implementation details
- [PHASE_5_ANALYTICS_EXPANSION.md](PHASE_5_ANALYTICS_EXPANSION.md) - Phase 5 implementation details
- [PHASE_6_RECOMMENDATIONS.md](PHASE_6_RECOMMENDATIONS.md) - Phase 6 implementation details
- [PHASE_7_HARDENING.md](PHASE_7_HARDENING.md) - Phase 7 implementation details
