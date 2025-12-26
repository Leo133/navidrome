# Phase 7: Hardening & Production Readiness
**Duration:** 3-4 weeks  
**Team Size:** 4-5 engineers  
**Dependencies:** Phases 2-6 complete (all features implemented)

---

## Objective

Prepare system for **production deployment** at scale:
1. Security hardening (authentication, authorization, rate limiting, CSRF protection)
2. Privacy enforcement (GDPR compliance, data export/delete, consent management)
3. Performance optimization (caching, query optimization, horizontal scaling)
4. Reliability improvements (error handling, retries, circuit breakers)
5. Observability enhancement (logging, metrics, distributed tracing)
6. Data integrity (backups, disaster recovery, data validation)
7. Moderation tools (flag/review moments, block users)

This ensures the system is **secure, scalable, and reliable** for production use.

---

## Scope

### In Scope
- ✅ Security audit and hardening
- ✅ Privacy controls (GDPR compliance)
- ✅ Performance optimization (caching, query tuning)
- ✅ Error handling and resilience
- ✅ Observability (logging, metrics, tracing)
- ✅ Data backup and recovery
- ✅ Moderation tools
- ✅ Load testing and capacity planning
- ✅ Documentation (deployment, operations, API)

### Out of Scope
- ❌ Multi-tenancy (single-instance deployment for v1)
- ❌ CDN integration (defer to later phase)
- ❌ Mobile app optimization (defer to later phase)

---

## Key Technical Work Items

### 1. Security Hardening

**Goal:** Ensure system is secure against common attacks.

**Security Checklist:**

#### 1.1 Authentication & Authorization

**Navidrome (Core):**
- ✅ JWT authentication (already implemented)
- ✅ Token refresh (already implemented)
- ⚠️ Add JWT expiration (short-lived: 1 hour, refresh token: 30 days)
- ⚠️ Add token revocation (blacklist compromised tokens)
- ⚠️ Add brute-force protection (rate limit login attempts)

**Social/Analytics/Recommendation Services:**
- ⚠️ Add JWT verification middleware (validate Navidrome tokens)
- ⚠️ Add service-to-service auth (API keys for inter-service communication)
- ⚠️ Add role-based access control (admin vs. user permissions)

**Implementation:** `server/auth/jwt_middleware.go`

```go
package auth

import (
    "net/http"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
    "github.com/navidrome/navidrome/conf"
)

const (
    AccessTokenExpiration  = 1 * time.Hour
    RefreshTokenExpiration = 30 * 24 * time.Hour
)

type TokenBlacklist interface {
    Add(token string, expiration time.Time) error
    IsBlacklisted(token string) (bool, error)
}

func ValidateJWT(tokenString string, blacklist TokenBlacklist) (*jwt.Token, error) {
    // Parse token
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte(conf.Server.JWTSecret), nil
    })
    if err != nil {
        return nil, err
    }
    
    // Check if blacklisted
    isBlacklisted, err := blacklist.IsBlacklisted(tokenString)
    if err != nil || isBlacklisted {
        return nil, fmt.Errorf("token is revoked")
    }
    
    // Validate expiration
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token")
    }
    
    exp, ok := claims["exp"].(float64)
    if !ok || time.Unix(int64(exp), 0).Before(time.Now()) {
        return nil, fmt.Errorf("token expired")
    }
    
    return token, nil
}
```

**Rate Limiting:** `server/middleware/rate_limiter.go`

```go
package middleware

import (
    "net/http"
    "time"
    
    "github.com/ulule/limiter/v3"
    "github.com/ulule/limiter/v3/drivers/store/redis"
)

func RateLimiter(redisClient *redis.Client, rate limiter.Rate) func(http.Handler) http.Handler {
    store := redis.NewStoreWithOptions(redisClient, limiter.StoreOptions{
        Prefix: "navidrome_rate_limit",
    })
    
    rateLimiter := limiter.New(store, rate)
    
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Get client IP (or user ID if authenticated)
            key := r.RemoteAddr
            if userID := getUserIDFromRequest(r); userID != "" {
                key = userID
            }
            
            context, err := rateLimiter.Get(r.Context(), key)
            if err != nil {
                log.Error(r.Context(), "Rate limiter error", err)
                http.Error(w, "Internal error", http.StatusInternalServerError)
                return
            }
            
            // Check if limit exceeded
            if context.Reached {
                w.Header().Set("X-Rate-Limit-Limit", fmt.Sprint(context.Limit))
                w.Header().Set("X-Rate-Limit-Remaining", "0")
                w.Header().Set("X-Rate-Limit-Reset", fmt.Sprint(context.Reset))
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            // Set rate limit headers
            w.Header().Set("X-Rate-Limit-Limit", fmt.Sprint(context.Limit))
            w.Header().Set("X-Rate-Limit-Remaining", fmt.Sprint(context.Remaining))
            w.Header().Set("X-Rate-Limit-Reset", fmt.Sprint(context.Reset))
            
            next.ServeHTTP(w, r)
        })
    }
}

// Rate limits by endpoint
var RateLimits = map[string]limiter.Rate{
    "/api/auth/login":          limiter.Rate{Period: 1 * time.Minute, Limit: 5},   // 5 login attempts per minute
    "/api/events/listen":       limiter.Rate{Period: 1 * time.Minute, Limit: 60},  // 60 requests per minute (generous for telemetry)
    "/api/recommendations/*":   limiter.Rate{Period: 1 * time.Minute, Limit: 30},  // 30 requests per minute
    "default":                  limiter.Rate{Period: 1 * time.Minute, Limit: 100}, // 100 requests per minute (general API)
}
```

#### 1.2 Input Validation & Sanitization

**SQL Injection Prevention:**
- ✅ Use parameterized queries (squirrel already does this)
- ⚠️ Audit all raw SQL queries (no string concatenation)

**XSS Prevention:**
- ⚠️ Sanitize user input (strip HTML tags from comments, usernames)
- ⚠️ Set CSP headers (Content-Security-Policy)

**Implementation:** `server/middleware/security_headers.go`

```go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Content Security Policy
        w.Header().Set("Content-Security-Policy", 
            "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';")
        
        // XSS Protection
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        
        // Frame Options (prevent clickjacking)
        w.Header().Set("X-Frame-Options", "SAMEORIGIN")
        
        // Content Type Options (prevent MIME sniffing)
        w.Header().Set("X-Content-Type-Options", "nosniff")
        
        // HTTPS Redirect (in production)
        if conf.Server.TLSEnabled && r.URL.Scheme != "https" {
            http.Redirect(w, r, "https://"+r.Host+r.RequestURI, http.StatusMovedPermanently)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

#### 1.3 CSRF Protection

**Implementation:** `server/middleware/csrf.go`

```go
import "github.com/gorilla/csrf"

func CSRFProtection() func(http.Handler) http.Handler {
    return csrf.Protect(
        []byte(conf.Server.CSRFSecret),
        csrf.Secure(conf.Server.TLSEnabled),
        csrf.SameSite(csrf.SameSiteStrictMode),
    )
}
```

**Tasks:**
- [ ] Audit authentication (add token expiration, revocation)
- [ ] Add rate limiting (login, API endpoints)
- [ ] Add security headers middleware
- [ ] Add CSRF protection
- [ ] Sanitize user input (strip HTML tags)
- [ ] Run security scan (OWASP ZAP, Snyk)

**Deliverable:** Security hardening complete

**Acceptance Criteria:**
- No critical/high vulnerabilities (security scan)
- Rate limiting works (429 Too Many Requests after limit)
- CSRF protection works (requests without token are rejected)
- Security headers are set (verified with browser dev tools)

---

### 2. Privacy & GDPR Compliance

**Goal:** Ensure system complies with privacy regulations.

**GDPR Requirements:**

#### 2.1 Consent Management

**Implementation:** Add consent tracking to user preferences

```sql
ALTER TABLE user_props ADD COLUMN consent_telemetry BOOLEAN DEFAULT FALSE;
ALTER TABLE user_props ADD COLUMN consent_analytics BOOLEAN DEFAULT FALSE;
ALTER TABLE user_props ADD COLUMN consent_social BOOLEAN DEFAULT FALSE;
ALTER TABLE user_props ADD COLUMN consent_date TIMESTAMP;
```

**UI:** Add consent screen on first login

```tsx
export const ConsentDialog = ({ onAccept, onDecline }) => {
    const [consents, setConsents] = useState({
        telemetry: false,
        analytics: false,
        social: false,
    });
    
    return (
        <Dialog open={true}>
            <DialogTitle>Privacy Preferences</DialogTitle>
            <DialogContent>
                <Typography>We collect data to improve your experience. Please review and accept:</Typography>
                
                <FormControlLabel
                    control={<Checkbox checked={consents.telemetry} onChange={(e) => setConsents({...consents, telemetry: e.target.checked})} />}
                    label="Listening telemetry (required for analytics)"
                />
                <FormHelperText>We track what you listen to, when, and for how long.</FormHelperText>
                
                <FormControlLabel
                    control={<Checkbox checked={consents.analytics} onChange={(e) => setConsents({...consents, analytics: e.target.checked})} />}
                    label="Analytics (optional)"
                />
                <FormHelperText>We compute derived metrics like completion %, skip patterns.</FormHelperText>
                
                <FormControlLabel
                    control={<Checkbox checked={consents.social} onChange={(e) => setConsents({...consents, social: e.target.checked})} />}
                    label="Social features (optional)"
                />
                <FormHelperText>Your listening activity will be shared with friends (if you choose).</FormHelperText>
            </DialogContent>
            <DialogActions>
                <Button onClick={onDecline}>Decline All</Button>
                <Button onClick={() => onAccept(consents)} variant="contained" color="primary">Accept</Button>
            </DialogActions>
        </Dialog>
    );
};
```

#### 2.2 Data Export (GDPR Right to Data Portability)

**Implementation:** `cmd/export.go`

```go
package cmd

import (
    "encoding/json"
    "fmt"
    "os"
    
    "github.com/spf13/cobra"
)

var exportCmd = &cobra.Command{
    Use:   "export-user-data [user-id]",
    Short: "Export all user data (GDPR compliance)",
    Args:  cobra.ExactArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        userID := args[0]
        
        // Export listening history
        listenEvents := ds.ListenEvent(ctx).GetByUser(userID, time.Time{}, time.Now(), -1)
        
        // Export social data
        follows := socialClient.GetFollows(userID)
        likes := socialClient.GetLikes(userID)
        moments := socialClient.GetMoments(userID)
        
        // Export analytics data
        intervals := analyticsClient.GetIntervals(userID)
        sessions := analyticsClient.GetSessions(userID)
        
        // Combine into single JSON
        export := map[string]interface{}{
            "userId":        userID,
            "exportedAt":    time.Now(),
            "listenEvents":  listenEvents,
            "follows":       follows,
            "likes":         likes,
            "moments":       moments,
            "intervals":     intervals,
            "sessions":      sessions,
        }
        
        // Write to file
        filename := fmt.Sprintf("user_%s_export_%s.json", userID, time.Now().Format("20060102"))
        file, _ := os.Create(filename)
        defer file.Close()
        
        encoder := json.NewEncoder(file)
        encoder.SetIndent("", "  ")
        encoder.Encode(export)
        
        fmt.Printf("Exported user data to %s\n", filename)
    },
}
```

#### 2.3 Data Deletion (GDPR Right to Erasure)

**Implementation:** `cmd/delete.go`

```go
var deleteUserDataCmd = &cobra.Command{
    Use:   "delete-user-data [user-id]",
    Short: "Delete all user data (GDPR compliance)",
    Args:  cobra.ExactArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        userID := args[0]
        
        fmt.Printf("⚠️  WARNING: This will permanently delete ALL data for user %s\n", userID)
        fmt.Print("Type 'DELETE' to confirm: ")
        
        var confirmation string
        fmt.Scanln(&confirmation)
        
        if confirmation != "DELETE" {
            fmt.Println("Aborted.")
            return
        }
        
        // Delete from Navidrome
        ds.ListenEvent(ctx).DeleteByUser(userID)
        ds.User(ctx).Delete(userID)
        
        // Delete from Social Service
        socialClient.DeleteUser(userID)
        
        // Delete from Analytics Service
        analyticsClient.DeleteUser(userID)
        
        // Delete from Recommendation Service
        recommendationClient.DeleteUser(userID)
        
        fmt.Printf("✅ Deleted all data for user %s\n", userID)
    },
}
```

#### 2.4 Data Retention Policies

**Configuration:** `conf/configuration.go`

```go
type configOptions struct {
    // ... existing fields
    DataRetentionDays int `mapstructure:"DataRetentionDays"`
}

func init() {
    viper.SetDefault("DataRetentionDays", 365)  // 1 year default
}
```

**Background Job:** Run daily to purge old data

```go
// scheduler/retention_job.go

func PurgeOldData(ctx context.Context, ds model.DataStore) {
    retentionDays := conf.Server.DataRetentionDays
    cutoffDate := time.Now().Add(-time.Duration(retentionDays) * 24 * time.Hour)
    
    // Delete old listen events
    count, _ := ds.ListenEvent(ctx).DeleteOlderThan(cutoffDate)
    log.Info(ctx, "Purged old listen events", "count", count)
    
    // Delete old activities (social service)
    socialClient.DeleteActivitiesOlderThan(cutoffDate)
    
    // Delete old intervals (analytics service)
    analyticsClient.DeleteIntervalsOlderThan(cutoffDate)
}
```

**Tasks:**
- [ ] Add consent management (UI + backend)
- [ ] Implement data export (CLI command)
- [ ] Implement data deletion (CLI command)
- [ ] Add retention policy (auto-purge old data)
- [ ] Document privacy practices (privacy policy)
- [ ] Add privacy settings to user preferences page

**Deliverable:** GDPR compliance

**Acceptance Criteria:**
- Users can export their data (JSON format)
- Users can delete their data (permanent deletion)
- Consent is tracked and enforced
- Retention policy automatically purges old data

---

### 3. Performance Optimization

**Goal:** Optimize system for production workloads.

#### 3.1 Database Query Optimization

**Audit slow queries:**

```sql
-- PostgreSQL: Enable slow query logging
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- Log queries >1s
SELECT pg_reload_conf();

-- Identify slow queries
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Add missing indexes:**

```sql
-- Social Service
CREATE INDEX CONCURRENTLY idx_follows_created_at ON follows(created_at DESC);
CREATE INDEX CONCURRENTLY idx_likes_created_at ON likes(created_at DESC);
CREATE INDEX CONCURRENTLY idx_moments_visibility ON moments(visibility, timestamp_ms);

-- Analytics Service (TimescaleDB)
CREATE INDEX CONCURRENTLY idx_track_stats_time_bucket ON track_stats(time_bucket DESC);
```

**Optimize N+1 queries:** Use DataLoader pattern (batch fetching)

```go
// graphql/dataloader.go

type TrackLoader struct {
    navidromeClient *NavidromeClient
    cache           map[string]*model.MediaFile
}

func (l *TrackLoader) Load(trackID string) (*model.MediaFile, error) {
    // Check cache
    if track, ok := l.cache[trackID]; ok {
        return track, nil
    }
    
    // Fetch from Navidrome
    track, err := l.navidromeClient.GetTrack(trackID)
    if err != nil {
        return nil, err
    }
    
    l.cache[trackID] = track
    return track, nil
}

func (l *TrackLoader) LoadMany(trackIDs []string) ([]*model.MediaFile, error) {
    // Batch fetch (single HTTP request)
    tracks, err := l.navidrome Client.GetTracksBulk(trackIDs)
    if err != nil {
        return nil, err
    }
    
    // Populate cache
    for _, track := range tracks {
        l.cache[track.ID] = track
    }
    
    return tracks, nil
}
```

#### 3.2 Caching

**Redis caching for hot data:**

```go
// cache/redis_cache.go

type RedisCache struct {
    client *redis.Client
}

func (c *RedisCache) Get(key string) (string, error) {
    return c.client.Get(context.Background(), key).Result()
}

func (c *RedisCache) Set(key string, value string, expiration time.Duration) error {
    return c.client.Set(context.Background(), key, value, expiration).Err()
}

// Example: Cache activity feed
func (s *FeedService) GetFeed(userID string) ([]Activity, error) {
    cacheKey := fmt.Sprintf("feed:%s", userID)
    
    // Try cache first
    cached, err := cache.Get(cacheKey)
    if err == nil {
        var feed []Activity
        json.Unmarshal([]byte(cached), &feed)
        return feed, nil
    }
    
    // Cache miss, fetch from DB
    feed, err := s.repo.GetFeed(userID)
    if err != nil {
        return nil, err
    }
    
    // Store in cache (5-minute TTL)
    cacheData, _ := json.Marshal(feed)
    cache.Set(cacheKey, string(cacheData), 5*time.Minute)
    
    return feed, nil
}
```

**Cache invalidation:**

```go
// Invalidate feed cache when new activity is created
func (s *ActivityService) Create(activity *Activity) error {
    // Save to DB
    err := s.repo.Create(activity)
    if err != nil {
        return err
    }
    
    // Invalidate cache for user's followers (they should see new activity)
    followers, _ := s.followService.GetFollowers(activity.UserID)
    for _, followerID := range followers {
        cacheKey := fmt.Sprintf("feed:%s", followerID)
        cache.Delete(cacheKey)
    }
    
    return nil
}
```

#### 3.3 Horizontal Scaling

**Stateless services:** All services are already stateless (can scale horizontally)

**Load balancer configuration:** `nginx.conf`

```nginx
upstream navidrome {
    least_conn;  # Route to least-loaded instance
    server navidrome-1:4533;
    server navidrome-2:4533;
    server navidrome-3:4533;
}

upstream social-service {
    server social-1:8080;
    server social-2:8080;
}

upstream analytics-service {
    server analytics-1:8081;
    server analytics-2:8081;
}

server {
    listen 443 ssl http2;
    server_name music.example.com;
    
    ssl_certificate /etc/ssl/certs/music.example.com.crt;
    ssl_certificate_key /etc/ssl/private/music.example.com.key;
    
    location /api/social/ {
        proxy_pass http://social-service/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /api/analytics/ {
        proxy_pass http://analytics-service/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location / {
        proxy_pass http://navidrome/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Tasks:**
- [ ] Audit slow queries (identify bottlenecks)
- [ ] Add missing indexes
- [ ] Implement caching (Redis for hot data)
- [ ] Optimize N+1 queries (DataLoader pattern)
- [ ] Configure load balancer (Nginx)
- [ ] Test horizontal scaling (spin up 3 instances, verify load distribution)

**Deliverable:** Performance optimization complete

**Acceptance Criteria:**
- P95 query latency <100ms (improved from baseline)
- Cache hit rate >80% for hot data (feeds, recommendations)
- System scales horizontally (3 instances handle 3x load)

---

### 4. Observability

**Goal:** Comprehensive monitoring, logging, and tracing.

#### 4.1 Structured Logging

**Standardize log format (JSON):**

```go
// log/logger.go

import "github.com/rs/zerolog"

func InitLogger() {
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
    
    logger := zerolog.New(os.Stdout).With().
        Timestamp().
        Str("service", "navidrome").
        Logger()
    
    log.Logger = logger
}

// Usage
log.Info().
    Str("userId", userID).
    Str("trackId", trackID).
    Int("position", position).
    Msg("Listen event recorded")
```

#### 4.2 Distributed Tracing (Jaeger)

**Add tracing middleware:**

```go
import (
    "github.com/opentracing/opentracing-go"
    "github.com/uber/jaeger-client-go"
)

func InitTracing() {
    cfg := jaeger.Configuration{
        ServiceName: "navidrome",
        Sampler: &jaeger.SamplerConfig{
            Type:  "const",
            Param: 1,  // Sample 100% (adjust for production)
        },
        Reporter: &jaeger.ReporterConfig{
            LocalAgentHostPort: "localhost:6831",
        },
    }
    
    tracer, closer, _ := cfg.NewTracer()
    opentracing.SetGlobalTracer(tracer)
    
    return closer
}

// Middleware
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        span, ctx := opentracing.StartSpanFromContext(r.Context(), r.URL.Path)
        defer span.Finish()
        
        r = r.WithContext(ctx)
        next.ServeHTTP(w, r)
    })
}
```

#### 4.3 Prometheus Metrics

**Add comprehensive metrics:**

```go
// metrics/metrics.go

var (
    // HTTP metrics
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "navidrome_http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "navidrome_http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
    
    // Telemetry metrics
    telemetryEventsIngested = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "navidrome_telemetry_events_ingested_total",
            Help: "Total number of telemetry events ingested",
        },
    )
    
    // Database metrics
    dbConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "navidrome_db_connections_active",
            Help: "Number of active database connections",
        },
    )
)

func init() {
    prometheus.MustRegister(
        httpRequestsTotal,
        httpRequestDuration,
        telemetryEventsIngested,
        dbConnections,
    )
}
```

**Grafana Dashboard:** Create dashboards for:
- HTTP request rate, latency, error rate
- Telemetry ingestion rate
- Database connection pool usage
- Cache hit rate
- Service health (up/down)

**Tasks:**
- [ ] Add structured logging (JSON format)
- [ ] Add distributed tracing (Jaeger)
- [ ] Add Prometheus metrics (HTTP, DB, telemetry)
- [ ] Create Grafana dashboards
- [ ] Setup alerting (Prometheus Alertmanager)

**Deliverable:** Comprehensive observability

**Acceptance Criteria:**
- All logs are structured (JSON)
- Traces show end-to-end request flow
- Grafana dashboards show key metrics
- Alerts fire for critical issues (high error rate, high latency)

---

### 5. Moderation Tools

**Goal:** Tools for moderating user-generated content.

**Admin Dashboard:** `ui/src/admin/ModerationDashboard.jsx`

```tsx
export const ModerationDashboard = () => {
    const [flaggedMoments, setFlaggedMoments] = useState([]);
    
    useEffect(() => {
        fetchFlaggedMoments();
    }, []);
    
    const fetchFlaggedMoments = async () => {
        const response = await fetch('/api/admin/moderation/flagged-moments');
        const data = await response.json();
        setFlaggedMoments(data.moments);
    };
    
    const handleReview = async (momentId, action) => {
        await fetch(`/api/admin/moderation/moments/${momentId}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ action }),  // 'approve' or 'delete'
        });
        
        // Refresh list
        fetchFlaggedMoments();
    };
    
    return (
        <div>
            <Typography variant="h4">Moderation Dashboard</Typography>
            <List>
                {flaggedMoments.map(moment => (
                    <ListItem key={moment.id}>
                        <ListItemText
                            primary={moment.comment}
                            secondary={`Flagged by ${moment.flaggedByCount} users · Reason: ${moment.flagReason}`}
                        />
                        <ListItemSecondaryAction>
                            <Button onClick={() => handleReview(moment.id, 'approve')}>Approve</Button>
                            <Button onClick={() => handleReview(moment.id, 'delete')} color="secondary">Delete</Button>
                        </ListItemSecondaryAction>
                    </ListItem>
                ))}
            </List>
        </div>
    );
};
```

**API:** `server/nativeapi/moderation.go`

```go
func (api *Router) GetFlaggedMoments(w http.ResponseWriter, r *http.Request) {
    // Check if user is admin
    user, _ := request.UserFrom(r.Context())
    if !user.IsAdmin {
        http.Error(w, "Forbidden", http.StatusForbidden)
        return
    }
    
    // Fetch flagged moments from social service
    moments, err := api.socialClient.GetFlaggedMoments()
    if err != nil {
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "moments": moments,
    })
}

func (api *Router) ReviewMoment(w http.ResponseWriter, r *http.Request) {
    user, _ := request.UserFrom(r.Context())
    if !user.IsAdmin {
        http.Error(w, "Forbidden", http.StatusForbidden)
        return
    }
    
    momentID := chi.URLParam(r, "id")
    
    var req struct {
        Action string `json:"action"`  // 'approve' or 'delete'
    }
    json.NewDecoder(r.Body).Decode(&req)
    
    if req.Action == "delete" {
        api.socialClient.DeleteMoment(momentID)
    } else if req.Action == "approve" {
        api.socialClient.UnflagMoment(momentID)
    }
    
    w.WriteHeader(http.StatusOK)
}
```

**Tasks:**
- [ ] Create admin dashboard UI (moderation page)
- [ ] Add API endpoints (get flagged moments, review moments)
- [ ] Add role-based access control (admin-only)
- [ ] Add bulk actions (approve/delete multiple moments)

**Deliverable:** Moderation tools

**Acceptance Criteria:**
- Admins can view flagged moments
- Admins can approve/delete moments
- Non-admins cannot access moderation dashboard

---

### 6. Load Testing & Capacity Planning

**Goal:** Validate system can handle expected production load.

**Load Test Scenarios:**

1. **Normal Load:** 100 concurrent users, 1000 requests/minute
2. **Peak Load:** 500 concurrent users, 5000 requests/minute
3. **Spike Load:** 1000 concurrent users for 5 minutes
4. **Sustained Load:** 200 concurrent users for 1 hour

**Tools:** k6 (load testing)

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '2m', target: 100 },  // Ramp up to 100 users
        { duration: '5m', target: 100 },  // Stay at 100 users
        { duration: '2m', target: 500 },  // Ramp up to 500 users
        { duration: '5m', target: 500 },  // Stay at 500 users
        { duration: '2m', target: 0 },    // Ramp down to 0 users
    ],
};

export default function () {
    // Test telemetry ingestion
    const payload = JSON.stringify([
        {
            mediaFileId: 'track-123',
            eventType: 'play',
            timestamp: Date.now(),
            position: 0,
            metadata: { sessionId: 'session-abc' }
        }
    ]);
    
    const res = http.post('http://localhost:4533/api/events/listen', payload, {
        headers: { 
            'Content-Type': 'application/json',
            'Authorization': 'Bearer YOUR_JWT_TOKEN'
        },
    });
    
    check(res, {
        'status is 201': (r) => r.status === 201,
        'response time < 200ms': (r) => r.timings.duration < 200,
    });
    
    sleep(1);
}
```

**Run test:**

```bash
k6 run --vus 100 --duration 10m load-test.js
```

**Tasks:**
- [ ] Write load test scripts (k6)
- [ ] Run normal load test (100 users)
- [ ] Run peak load test (500 users)
- [ ] Run spike load test (1000 users)
- [ ] Analyze results (identify bottlenecks)
- [ ] Document capacity planning (max users per instance)

**Deliverable:** Load test results + capacity plan

**Acceptance Criteria:**
- System handles 100 concurrent users (P95 latency <500ms)
- System handles 500 concurrent users (P95 latency <1s)
- No errors during sustained load (1 hour)
- Capacity plan documented (max users per instance)

---

### 7. Documentation

**Goal:** Comprehensive documentation for deployment and operations.

**Documents to Create:**

1. **DEPLOYMENT.md:** Step-by-step deployment guide
2. **OPERATIONS.md:** Day-to-day operations (backups, monitoring, troubleshooting)
3. **API_REFERENCE.md:** Complete API documentation (endpoints, schemas, examples)
4. **ARCHITECTURE.md:** System architecture overview (components, data flow, technology stack)
5. **SECURITY.md:** Security best practices (authentication, authorization, encryption)
6. **PRIVACY.md:** Privacy policy and GDPR compliance details

**Tasks:**
- [ ] Write deployment guide (Docker, Kubernetes, bare metal)
- [ ] Write operations guide (backups, monitoring, troubleshooting)
- [ ] Generate API reference (Swagger/OpenAPI)
- [ ] Update architecture diagram
- [ ] Write security guide
- [ ] Write privacy policy

**Deliverable:** Complete documentation

**Acceptance Criteria:**
- New team member can deploy system using DEPLOYMENT.md
- Operator can troubleshoot issues using OPERATIONS.md
- Developer can integrate with API using API_REFERENCE.md

---

## Operational Needs

### Infrastructure
- Production environment (separate from staging)
- Backups (automated daily backups for all databases)
- Monitoring (Prometheus + Grafana + Alertmanager)
- Logging (Elasticsearch + Kibana or Loki + Grafana)

### Monitoring
- All metrics from previous phases
- Additional alerts for production issues

### Data Backup
- Daily backups (all databases: SQLite, PostgreSQL, TimescaleDB)
- Backup retention: 30 days
- Disaster recovery plan (RTO: 4 hours, RPO: 24 hours)

---

## Risks & Mitigations

### Risk: Production outage (service downtime)
**Likelihood:** Low  
**Impact:** Critical  
**Mitigation:**
- High availability (multiple instances, load balancer)
- Health checks (Kubernetes liveness/readiness probes)
- Alerting (PagerDuty, Opsgenie)
- Disaster recovery plan

### Risk: Data breach (unauthorized access)
**Likelihood:** Low  
**Impact:** Critical  
**Mitigation:**
- Security hardening (rate limiting, CSRF protection, input validation)
- Regular security audits
- Incident response plan

---

## Definition of Done

### Must Have
- ✅ Security hardening complete (rate limiting, CSRF, input validation)
- ✅ GDPR compliance (consent, export, deletion, retention)
- ✅ Performance optimization (caching, query tuning, horizontal scaling)
- ✅ Observability (structured logging, tracing, metrics)
- ✅ Moderation tools (admin dashboard)
- ✅ Load testing (validated for expected load)
- ✅ Documentation (deployment, operations, API)

### Should Have
- ✅ Automated backups
- ✅ Disaster recovery plan
- ✅ Incident response plan

### Could Have
- Multi-region deployment (defer to later phase)
- CDN integration (defer to later phase)

---

## Success Metrics

- **Uptime:** >99.5% (production)
- **Security:** No critical/high vulnerabilities
- **Performance:** P95 latency <500ms (under normal load)
- **Data Loss:** 0% (backups + disaster recovery)

---

## Next Steps (Post-Launch)

After Phase 7 is complete:
1. Deploy to production (rollout to small user group first)
2. Monitor for 2 weeks (ensure stability)
3. Rollout to all users
4. Collect feedback (surveys, user interviews)
5. Plan next features (Phase 8+):
   - Mobile app optimization
   - Real-time features (WebSocket, SSE)
   - Advanced ML (audio fingerprinting, deep learning models)
   - Multi-region deployment

---

**Estimated Effort:**
- Senior Backend Engineer: 80 hours (security, performance, observability)
- Mid-Level Backend Engineer: 60 hours (GDPR compliance, moderation tools)
- DevOps Engineer: 80 hours (deployment, backups, monitoring, load testing)
- Technical Writer: 40 hours (documentation)
- **Total: 260 hours (~6.5 person-weeks with 40-hour weeks)**

---

**End of Phase 7**

**🎉 System Ready for Production! 🎉**
