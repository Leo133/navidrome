# Phase 1: Stable Backend Baseline
**Duration:** 2-3 weeks  
**Team Size:** 3-4 engineers  
**Dependencies:** Phase 0 complete

---

## Objective

Establish a **rock-solid backend foundation** with:
1. Performance baselines (streaming, scanning, query latency)
2. Reliability improvements (error handling, retry logic, graceful degradation)
3. Configuration validation (schema validation, sensible defaults)
4. Observability infrastructure (structured logging, metrics, tracing)
5. Database optimization (indexes, query profiling, migration testing)

This phase ensures that adding new features (telemetry, social) won't degrade existing functionality.

---

## Scope

### In Scope
- ✅ Performance profiling and baseline metrics
- ✅ Error handling audit and improvements
- ✅ Database query optimization
- ✅ Observability setup (Prometheus metrics, structured logs)
- ✅ Configuration validation and documentation
- ✅ Load testing and stress testing
- ✅ Migration testing (forward and backward compatibility)
- ✅ Security audit (basic: SQL injection, XSS, CSRF)

### Out of Scope
- ❌ New features (telemetry, social)
- ❌ UI changes (unless fixing bugs discovered during testing)
- ❌ External services (Analytics, Social)
- ❌ Client development

---

## Key Technical Work Items

### 1. Performance Baseline Establishment

**Goal:** Measure current performance to detect regressions in future phases.

**Tasks:**
- [ ] Set up performance testing environment:
  - Large test library: 50,000 tracks, 5,000 albums, 500 artists
  - Generate using `scanner/testdata/` or real music collection
  - Script to create 100 test users with varied play history
- [ ] Profile critical paths with pprof:
  - Library scan: `go tool pprof -http=:8080 http://localhost:4533/debug/pprof/profile?seconds=30`
  - Streaming: concurrent stream requests (simulate 50 simultaneous streams)
  - Search: full-text search queries with varying complexity
  - Scrobbling: bulk scrobble submission (100 scrobbles/sec)
- [ ] Identify bottlenecks:
  - CPU hotspots (use `go tool pprof`)
  - Memory allocations (use `go tool pprof -alloc_space`)
  - Database locks (use `PRAGMA wal_checkpoint(FULL)` and analyze)
  - Disk I/O (use `iostat` during scan)
- [ ] Document baseline metrics:
  - Library scan time: X seconds for 50K tracks
  - Stream request latency: P50, P95, P99 (with/without transcoding)
  - Search query latency: P50, P95, P99
  - Scrobble insertion throughput: X writes/sec
  - Memory usage: idle, during scan, under load
  - CPU usage: idle, during scan, during concurrent streams

**Deliverable:** `PERFORMANCE_BASELINE.md` with metrics + pprof profiles

**Acceptance Criteria:**
- Baseline metrics are documented for comparison in future phases
- No performance regressions from upstream Navidrome (if any, document and justify)
- Bottlenecks are identified (even if not fixed in this phase)

---

### 2. Database Query Optimization

**Goal:** Ensure all queries are indexed and performant, especially annotation queries (will be extended in Phase 2).

**Tasks:**
- [ ] Enable query logging:
  - Set `ND_LOGLEVEL=trace` to log all SQL queries
  - Run test workload (scan, search, stream, scrobble)
  - Capture logs to file: `navidrome 2>&1 | tee queries.log`
- [ ] Analyze slow queries:
  - Extract queries >100ms from logs
  - Use `EXPLAIN QUERY PLAN` in SQLite to check index usage
  - Focus on:
    - Annotation queries (play counts, ratings, starred items)
    - Search queries (full-text search on `media_file`, `album`, `artist`)
    - Playlist queries (tracks join, smart playlist evaluation)
    - Library browsing (album list with filters, artist list with filters)
- [ ] Add missing indexes:
  - `annotation`: already has indexes on `(play_count)`, `(play_date)`, `(starred)`
  - Check if compound indexes would help: `(user_id, item_type, play_count DESC)`
  - `media_file`: check index on `(album_id, track_number)` for album track lists
  - `media_file`: check index on `(genre)` for genre browsing
- [ ] Optimize joins:
  - Review `withAnnotation()` in `persistence/sql_annotations.go`
  - Ensure LEFT JOIN on annotation is efficient
  - Consider materializing frequently accessed data (denormalization)
- [ ] Test index impact:
  - Re-run baseline queries after adding indexes
  - Measure improvement (should be >2x faster for filtered queries)
- [ ] Document query patterns:
  - Common queries and their execution plans
  - Index usage guide for future developers

**Deliverable:** New migration `20260105000001_add_performance_indexes.sql` + `QUERY_OPTIMIZATION.md`

**Acceptance Criteria:**
- All queries <100ms (P95) for test dataset
- No table scans on queries with WHERE clauses
- Index coverage >90% (measured by `EXPLAIN QUERY PLAN`)

---

### 3. Error Handling Audit & Improvements

**Goal:** Ensure all error paths are handled gracefully (log, return error, don't panic).

**Tasks:**
- [ ] Search for `panic()` calls in codebase:
  - `grep -r "panic(" --include="*.go" .`
  - Review each: is it justified (e.g., programmer error) or should it be an error return?
  - Replace unnecessary panics with error returns
- [ ] Search for unchecked errors:
  - Use `errcheck` tool: `go install github.com/kisielk/errcheck@latest && errcheck ./...`
  - Review output, fix critical paths (ignore defer Close() errors if acceptable)
- [ ] Add context to errors:
  - Review error returns in `persistence/`, `core/`, `server/`
  - Wrap errors with `fmt.Errorf("context: %w", err)` for better stack traces
  - Example: `return fmt.Errorf("failed to load media file %s: %w", id, err)`
- [ ] Add retry logic for external calls:
  - Review `core/agents/` (Last.fm, Spotify API calls)
  - Add exponential backoff with max retries (3 attempts, 1s/2s/4s delays)
  - Use `github.com/cenkalti/backoff/v4` or similar
- [ ] Improve error responses to clients:
  - Review `server/subsonic/api.go` error handling
  - Ensure errors return appropriate HTTP status codes (400, 404, 500, 503)
  - Don't leak internal details (e.g., SQL errors) to clients
- [ ] Add error logging:
  - Ensure all errors are logged before returning
  - Use `log.Error(ctx, "message", err)` consistently
  - Include context (user ID, track ID, etc.) for debugging

**Deliverable:** Pull request with error handling improvements + `ERROR_HANDLING_GUIDE.md`

**Acceptance Criteria:**
- `errcheck` tool reports <10 unchecked errors (only safe to ignore)
- No `panic()` calls in production code (only in tests or init functions)
- All errors are logged with context
- External API calls have retry logic

---

### 4. Observability Infrastructure

**Goal:** Add Prometheus metrics and improve structured logging for production monitoring.

**Tasks:**
- [ ] Review existing Prometheus metrics:
  - Navidrome already has `core/metrics/prometheus.go`
  - Current metrics: scan_duration, scan_errors, HTTP request durations, etc.
  - Check if metrics are exposed: `curl http://localhost:4533/metrics`
- [ ] Add new metrics for baseline monitoring:
  - `navidrome_stream_requests_total` (counter, labeled by: user, transcoding_format)
  - `navidrome_stream_duration_seconds` (histogram, labeled by: format, cached)
  - `navidrome_scrobble_submissions_total` (counter, labeled by: user, service)
  - `navidrome_annotation_updates_total` (counter, labeled by: item_type)
  - `navidrome_db_query_duration_seconds` (histogram, labeled by: query_type)
  - `navidrome_db_size_bytes` (gauge, updated every 5 minutes)
- [ ] Add metrics to critical paths:
  - `core/media_streamer.go`: increment stream counter, observe duration
  - `core/scrobbler/play_tracker.go`: increment scrobble counter
  - `persistence/sql_base_repository.go`: observe query duration (generic wrapper)
- [ ] Improve structured logging:
  - Review log statements in `core/`, `server/`, `persistence/`
  - Ensure consistent use of key-value pairs: `log.Info(ctx, "msg", "key", value)`
  - Add important context: userID, trackID, playerID, requestID
  - Avoid PII in logs (don't log passwords, IP addresses, etc.)
- [ ] Configure log levels:
  - Default: INFO (production)
  - Development: DEBUG
  - Critical errors: ERROR (page ops team)
- [ ] Set up log aggregation (optional, but recommended):
  - Export logs to Loki or Elasticsearch
  - Create Grafana dashboards for log queries
- [ ] Create Prometheus alert rules:
  - Alert if error rate >1% of requests
  - Alert if stream latency P95 >500ms
  - Alert if DB size >10GB (indicates large library or bloat)
  - Alert if scan duration >2x baseline

**Deliverable:** New metrics in `core/metrics/`, updated `server/subsonic/api.go`, `prometheus_alerts.yml`, `OBSERVABILITY_GUIDE.md`

**Acceptance Criteria:**
- Prometheus metrics endpoint exposes 15+ useful metrics
- All critical paths emit metrics
- Logs include context (user, track, request ID)
- Alert rules are defined (even if not deployed to alerting system yet)

---

### 5. Configuration Validation

**Goal:** Ensure config file is validated on startup (prevent runtime errors from misconfig).

**Tasks:**
- [ ] Review `conf/configuration.go`:
  - Current config options: 90+ settings
  - Some have defaults, some are required (e.g., MusicFolder)
- [ ] Add config schema validation:
  - Use `github.com/go-playground/validator/v10` for struct validation
  - Add validation tags to `configOptions` struct:
    ```go
    type configOptions struct {
        MusicFolder string `validate:"required,dirpath"`  // must exist, must be directory
        Port        int    `validate:"min=1,max=65535"`   // valid port range
        BaseURL     string `validate:"omitempty,url"`     // if set, must be valid URL
        // ... etc.
    }
    ```
- [ ] Validate on startup:
  - Call `validator.Struct(conf.Server)` in `conf/configuration.go`
  - If validation fails, log errors and exit (don't start with invalid config)
- [ ] Add validation for common mistakes:
  - `MusicFolder` must be an absolute path
  - `FFmpegPath` must be executable (run `ffmpeg -version` to check)
  - `TranscodingCacheSize` must be valid size string (e.g., "1GB", not "1 GB" or "1G")
  - `SessionTimeout` must be >0
- [ ] Improve config documentation:
  - Update `conf/configuration.go` godoc comments
  - Create `CONFIG_OPTIONS.md` with descriptions, defaults, examples for every option
  - Include environment variable mapping (e.g., `ND_MUSICFOLDER`)
- [ ] Add config file examples:
  - `navidrome.toml.example` (comprehensive example with all options)
  - `navidrome.minimal.toml` (minimal config for quick start)

**Deliverable:** Config validation code + `CONFIG_OPTIONS.md` + example config files

**Acceptance Criteria:**
- Navidrome exits with clear error if config is invalid (doesn't crash later)
- All config options are documented (name, type, default, description)
- Example config files are tested (actually work)

---

### 6. Migration Testing

**Goal:** Ensure database migrations are safe (can upgrade and downgrade without data loss).

**Tasks:**
- [ ] Review migration files in `db/migrations/`:
  - 80+ migrations from v0.12.0 to v0.52.x
  - Check if all have `Up` and `Down` functions
  - Check if all are idempotent (can run multiple times safely)
- [ ] Test forward migration path:
  - Start with empty DB (Navidrome v0.50.0)
  - Scan library (populate tables)
  - Run all migrations up to latest (v0.52.5+fork)
  - Verify data integrity (no lost tracks, albums, play counts)
- [ ] Test backward migration path:
  - Start with latest DB (v0.52.5+fork)
  - Run migrations down to v0.50.0
  - Verify data integrity (no errors, no orphaned records)
  - Note: Some down migrations may lose data (e.g., dropping columns) - document this
- [ ] Test schema changes for Phase 2:
  - Create new migration: `20260110000001_create_listen_event_table.go`
  - Run Up: verify table is created with correct schema
  - Insert test data: verify inserts work
  - Run Down: verify table is dropped cleanly
  - Run Up again: verify idempotence
- [ ] Add migration tests:
  - Unit tests for each new migration
  - Test Up and Down separately
  - Use in-memory SQLite DB for fast tests
- [ ] Document migration best practices:
  - Always add `Up` and `Down`
  - Always add indexes in same migration that creates table
  - Use transactions (`goose.WithTransaction()`) for safety
  - Test on copy of production DB before deploying

**Deliverable:** Migration tests + `MIGRATION_GUIDE.md`

**Acceptance Criteria:**
- All existing migrations have been tested (up and down)
- New migration for `listen_event` table is created and tested
- Migration guide documents best practices

---

### 7. Load Testing & Stress Testing

**Goal:** Ensure Navidrome can handle production load (simulate 100 concurrent users).

**Tasks:**
- [ ] Set up load testing tool:
  - Use `k6` (https://k6.io/) or `vegeta` (https://github.com/tsenart/vegeta)
  - Install: `brew install k6` (macOS) or `apt install k6` (Linux)
- [ ] Create load test scripts:
  - **Scenario 1: Streaming load**
    - 50 virtual users
    - Each user streams 10 tracks sequentially (mix of raw and transcoded)
    - Duration: 10 minutes
    - Expected: P95 latency <500ms, 0 errors
  - **Scenario 2: Search load**
    - 20 virtual users
    - Each user performs search queries (10 queries/minute)
    - Queries: artist search, album search, track search, genre filter
    - Duration: 5 minutes
    - Expected: P95 latency <200ms, 0 errors
  - **Scenario 3: Scrobble load**
    - 10 virtual users
    - Each user submits scrobbles (1 scrobble every 30 seconds)
    - Duration: 10 minutes
    - Expected: P95 latency <100ms, 0 errors
  - **Scenario 4: Mixed workload**
    - 100 virtual users
    - Mix of streaming (60%), search (30%), scrobble (10%)
    - Duration: 30 minutes
    - Expected: System remains stable, no crashes, acceptable latency
- [ ] Run load tests:
  - Baseline: current Navidrome (no fork changes)
  - Monitor: CPU, memory, disk I/O, DB locks, HTTP response times
  - Capture results: `k6 run --out json=results.json load_test.js`
- [ ] Analyze results:
  - Identify breaking points (at what load does system start failing?)
  - Identify bottlenecks (CPU? memory? DB locks?)
  - Compare to baseline metrics (are we within tolerance?)
- [ ] Document load test results:
  - Graphs: latency over time, error rate over time, throughput over time
  - System resources: CPU%, memory%, disk I/O
  - Recommendations: horizontal scaling, vertical scaling, caching, etc.

**Deliverable:** `load_tests/` directory with k6 scripts + `LOAD_TEST_RESULTS.md`

**Acceptance Criteria:**
- Load tests can be run automatically (`make load-test`)
- System handles 100 concurrent users without crashes
- Latency remains within acceptable bounds (P95 <500ms for streams)

---

### 8. Security Audit (Basic)

**Goal:** Identify and fix common security vulnerabilities (SQL injection, XSS, CSRF, secrets in logs).

**Tasks:**
- [ ] SQL injection audit:
  - Search for raw SQL queries: `grep -r "Query(" --include="*.go" .`
  - Ensure all use parameterized queries (Squirrel already handles this)
  - Check custom SQL in migrations (should be safe, but verify)
- [ ] XSS audit:
  - Review `server/subsonic/responses/` (XML responses)
  - Ensure user input is escaped (e.g., playlist names, comments)
  - Review `ui/src/` (React automatically escapes, but check `dangerouslySetInnerHTML`)
- [ ] CSRF protection:
  - Check if Subsonic API has CSRF tokens (not required for Subsonic spec)
  - Native API uses JWT (no cookies), so CSRF is not applicable
  - Verify `SameSite` cookie attribute if cookies are used
- [ ] Secrets in logs:
  - Search for password logging: `grep -r "Password" --include="*.go" log/`
  - Ensure `log/redactrus.go` is redacting sensitive fields
  - Add test: log user struct, verify password is `[REDACTED]`
- [ ] JWT security:
  - Verify JWT secret is stored encrypted in DB (already done in `core/auth/auth.go`)
  - Verify JWT has expiration (`exp` claim is set)
  - Verify JWT signature is validated on every request
- [ ] Rate limiting:
  - Verify login endpoint has rate limiting (already implemented in `server/server.go`)
  - Add rate limiting to scrobble endpoint (optional, but recommended)
  - Configuration: `ND_AUTHREQUESTLIMIT=5` (5 login attempts per minute per IP)
- [ ] Dependency vulnerabilities:
  - Run `go list -json -m all | nancy sleuth` (check for known CVEs)
  - Update vulnerable dependencies (if any)
- [ ] Add security headers:
  - `X-Content-Type-Options: nosniff` (already in `server/middlewares.go`)
  - `X-Frame-Options: DENY` (prevent clickjacking)
  - `Content-Security-Policy` (prevent inline scripts)
  - Verify with: `curl -I http://localhost:4533`

**Deliverable:** Security fixes (pull request) + `SECURITY_AUDIT_REPORT.md`

**Acceptance Criteria:**
- No SQL injection vulnerabilities
- No XSS vulnerabilities in common flows
- Secrets are not logged
- JWT is secure (encrypted secret, expiration, signature validation)
- Security headers are present

---

## Interfaces Introduced/Modified

**None** (Phase 1 focuses on stability, no new interfaces)

---

## Data Models Introduced/Modified

### Modified: Add indexes for performance

**Migration:** `20260105000001_add_performance_indexes.sql`

```sql
-- +goose Up
-- Improve annotation queries (used heavily for play counts, ratings)
CREATE INDEX IF NOT EXISTS idx_annotation_user_type_playcount 
    ON annotation(user_id, item_type, play_count DESC);

-- Improve album track list queries
CREATE INDEX IF NOT EXISTS idx_mediafile_album_track 
    ON media_file(album_id, disc_number, track_number);

-- Improve genre browsing
CREATE INDEX IF NOT EXISTS idx_mediafile_genre 
    ON media_file(genre);

-- Improve artist queries with play counts
CREATE INDEX IF NOT EXISTS idx_annotation_user_artist 
    ON annotation(user_id, item_type, item_id) 
    WHERE item_type = 'artist';

-- +goose Down
DROP INDEX IF EXISTS idx_annotation_user_type_playcount;
DROP INDEX IF EXISTS idx_mediafile_album_track;
DROP INDEX IF EXISTS idx_mediafile_genre;
DROP INDEX IF EXISTS idx_annotation_user_artist;
```

---

## Operational Needs

### Infrastructure
- Load testing environment: dedicated VM or cloud instance (2 vCPU, 4GB RAM)
- Prometheus server (for metrics collection)
- Grafana (for dashboards)

### Monitoring
- Set up Prometheus scraping: `http://navidrome:4533/metrics`
- Create Grafana dashboards:
  - Dashboard 1: Request latency (P50, P95, P99 for each endpoint)
  - Dashboard 2: System resources (CPU, memory, disk, DB size)
  - Dashboard 3: Error rates (by endpoint, by error type)
  - Dashboard 4: Business metrics (streams/hour, scrobbles/hour, active users)

### Alerting
- Configure Alertmanager (or PagerDuty, Opsgenie, etc.)
- Route alerts to Slack/Discord channel
- Define on-call rotation (who responds to alerts)

---

## Risks & Mitigations

### Risk: Performance regressions discovered during profiling
**Likelihood:** Medium  
**Impact:** High  
**Mitigation:**
- If regression is from upstream Navidrome: report to upstream, consider local patch
- If regression is from our changes: fix before proceeding to Phase 2
- Document all regressions in `PERFORMANCE_BASELINE.md`

### Risk: Database migrations fail on large datasets
**Likelihood:** Low  
**Impact:** High (data loss)  
**Mitigation:**
- Test migrations on copy of production DB (if available)
- Always create DB backup before migration: `sqlite3 navidrome.db ".backup navidrome.db.backup"`
- Add migration timeout (default 1 hour, configurable)
- Document rollback procedure in `MIGRATION_GUIDE.md`

### Risk: Load tests reveal system cannot handle 100 users
**Likelihood:** Low (Navidrome is already production-tested)  
**Impact:** Medium  
**Mitigation:**
- Identify bottleneck (CPU, memory, DB locks, network)
- Optimize query or add cache (depends on bottleneck)
- If cannot be fixed easily, document as known limitation
- Plan for horizontal scaling (multiple Navidrome instances behind load balancer)

### Risk: Security vulnerabilities discovered
**Likelihood:** Low (Navidrome is mature)  
**Impact:** High  
**Mitigation:**
- Fix immediately (critical vulnerabilities)
- Contribute fix upstream (if generic)
- Document in `SECURITY_AUDIT_REPORT.md`

---

## Definition of Done

### Must Have
- ✅ Performance baseline documented (latency, throughput, resource usage)
- ✅ All queries <100ms P95 (with indexes added)
- ✅ No unchecked errors in critical paths (`errcheck` tool passes)
- ✅ Prometheus metrics are exposed and documented
- ✅ Config validation works (invalid config causes startup error)
- ✅ Migration tests pass (up and down)
- ✅ Load tests pass (100 concurrent users, 0 crashes)
- ✅ Security audit complete (no critical vulnerabilities)

### Should Have
- ✅ Error handling guide documented
- ✅ Observability guide documented
- ✅ Grafana dashboards created
- ✅ Alert rules defined

### Could Have
- Log aggregation (Loki/Elasticsearch) set up
- Distributed tracing (Jaeger/Zipkin) added
- Performance profiling automation (continuous profiling)

---

## Success Metrics

- **Stability:** No crashes during 30-minute load test
- **Performance:** P95 latency <500ms for streams, <200ms for searches, <100ms for scrobbles
- **Observability:** 15+ Prometheus metrics, logs include context
- **Security:** No critical vulnerabilities, security headers present

---

## Next Steps (Transition to Phase 2)

After Phase 1 is complete:
1. Merge performance improvements to `develop` branch
2. Deploy to staging environment
3. Monitor for 1 week (ensure stability)
4. Create Phase 2 branch: `feature/telemetry-v1-foundations`
5. Begin work on `listen_event` table and ingestion endpoint

---

**Estimated Effort:**
- Senior Engineer (backend): 80 hours (profiling, optimization, security)
- Mid-Level Engineer (backend): 60 hours (error handling, observability, testing)
- DevOps Engineer: 40 hours (load testing, monitoring, alerts)
- **Total: 180 hours (~4.5 person-weeks with 40-hour weeks)**

---

**End of Phase 1**
