# Phase 4: Moment Layer (Timestamped Comments/Reactions)
**Duration:** 3-4 weeks  
**Team Size:** 4-5 engineers  
**Dependencies:** Phase 2 complete (telemetry), Phase 3 complete (social graph)

---

## Objective

Build the **Moments layer** - timestamped comments and reactions on tracks:
1. Allow users to add comments at specific timestamps in tracks (e.g., "Great guitar solo!" at 2:35)
2. Implement reactions (emoji) on moments
3. Visualize moments as heatmap on track progress bar
4. Aggregate moments to find "hot spots" (most-reacted sections)
5. Show friends' moments in listening UI (social discovery)
6. Privacy controls (public/friends/private moments)

This creates a **SoundCloud-style commenting experience** with social discovery.

---

## Scope

### In Scope
- ✅ Moments table (comments with timestamps)
- ✅ Reactions table (emoji reactions on moments)
- ✅ Moment heatmap visualization (progress bar overlay)
- ✅ Moment creation UI (comment at current playback position)
- ✅ Moment browsing UI (scroll through moments on a track)
- ✅ Privacy controls (public/friends/private moments)
- ✅ Hot spot detection (aggregate reactions by time range)
- ✅ Friend moments feed (see what friends are commenting on)

### Out of Scope
- ❌ Threaded replies (defer to later phase)
- ❌ Moment editing (immutable for v1)
- ❌ Rich media in comments (text only)
- ❌ Moderation tools (defer to Phase 7)
- ❌ Notifications (defer to later phase)

---

## Architecture

### Data Model

**Moments stored in Social Service (PostgreSQL):**
- Moments are social features (tied to social graph)
- Need complex queries (find moments by friends, aggregate by timestamp)
- PostgreSQL better suited than SQLite for this

**Moment Lifecycle:**
1. User pauses at 2:35 in a track
2. Clicks "Add Moment" button
3. Types comment: "Great guitar solo!"
4. Moment saved to Social Service (with timestamp_ms: 155000)
5. Other users see moment as dot on progress bar
6. Hover over dot → see comment
7. Click dot → seek to 2:35 and play
8. React to moment with emoji (👏, 🔥, ❤️)

---

## Key Technical Work Items

### 1. Database Schema: Moments & Reactions

**Goal:** Add tables to Social Service PostgreSQL for moments and reactions.

**Schema Design:**

```sql
-- Moments (timestamped comments on tracks)
CREATE TABLE moments (
    id              SERIAL PRIMARY KEY,
    user_id         VARCHAR(255) NOT NULL,
    track_id        VARCHAR(255) NOT NULL,       -- Navidrome media_file.id
    timestamp_ms    INTEGER NOT NULL,            -- Position in track (milliseconds)
    comment         TEXT NOT NULL,               -- Comment text (max 500 chars)
    visibility      VARCHAR(50) DEFAULT 'friends', -- 'public', 'friends', 'private'
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CHECK (timestamp_ms >= 0),
    CHECK (char_length(comment) <= 500)
);

CREATE INDEX idx_moments_track_timestamp ON moments(track_id, timestamp_ms);
CREATE INDEX idx_moments_user ON moments(user_id, created_at DESC);
CREATE INDEX idx_moments_created_at ON moments(created_at DESC);

-- Reactions (emoji reactions on moments)
CREATE TABLE moment_reactions (
    id              SERIAL PRIMARY KEY,
    moment_id       INTEGER NOT NULL,
    user_id         VARCHAR(255) NOT NULL,
    emoji           VARCHAR(10) NOT NULL,        -- Unicode emoji (e.g., '👏', '🔥')
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (moment_id) REFERENCES moments(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE(moment_id, user_id)                  -- One reaction per user per moment
);

CREATE INDEX idx_moment_reactions_moment ON moment_reactions(moment_id, created_at DESC);
CREATE INDEX idx_moment_reactions_user ON moment_reactions(user_id, created_at DESC);

-- Moment stats (materialized view for performance)
CREATE MATERIALIZED VIEW moment_stats AS
SELECT 
    m.id AS moment_id,
    m.track_id,
    m.timestamp_ms,
    COUNT(mr.id) AS reaction_count,
    COUNT(DISTINCT mr.user_id) AS unique_reactors,
    STRING_AGG(DISTINCT mr.emoji, ',') AS emoji_list
FROM moments m
LEFT JOIN moment_reactions mr ON m.id = mr.moment_id
GROUP BY m.id, m.track_id, m.timestamp_ms;

CREATE INDEX idx_moment_stats_track ON moment_stats(track_id, reaction_count DESC);

-- Refresh stats (run every 5 minutes)
-- REFRESH MATERIALIZED VIEW CONCURRENTLY moment_stats;
```

**Design Decisions:**
- **timestamp_ms instead of timestamp_seconds:** More precise (matches HTML5 Audio API)
- **500 char limit on comments:** Keeps UI clean (SoundCloud uses ~300 chars)
- **visibility per moment:** Users can choose public vs. friends-only for each comment
- **UNIQUE(moment_id, user_id) for reactions:** Users can only react once per moment (can change emoji)
- **Materialized view for stats:** Pre-compute reaction counts (avoid N+1 queries)

**Migration File:** `social-service/migrations/004_add_moments.sql`

**Tasks:**
- [ ] Create migration file
- [ ] Test migrations (up/down)
- [ ] Add domain models (`domain/moment.go`, `domain/moment_reaction.go`)
- [ ] Add repositories (`repository/postgres/moment_repository.go`)
- [ ] Document schema

**Deliverable:** Moments schema + repositories

**Acceptance Criteria:**
- Migrations run successfully
- Can insert moments with timestamps
- Can add reactions (one per user per moment)
- Stats view reflects reaction counts

---

### 2. GraphQL Schema Extensions

**Goal:** Add moments queries and mutations to GraphQL API.

**Schema Extensions:** `social-service/internal/graphql/schema.graphql`

```graphql
# ===== Moment Types =====

type Moment {
  id: ID!
  user: User!
  trackId: ID!
  track: Track                     # Resolved from Navidrome (cached)
  timestampMs: Int!
  comment: String!
  visibility: Visibility!
  reactionCount: Int!
  topReactions: [ReactionSummary!]! # Top 3 emoji with counts
  myReaction: String               # Current user's reaction emoji (if any)
  createdAt: String!
}

type ReactionSummary {
  emoji: String!
  count: Int!
}

type MomentConnection {
  edges: [MomentEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type MomentEdge {
  node: Moment!
  cursor: String!
}

# Moment heatmap (aggregated by time buckets)
type MomentHeatmap {
  trackId: ID!
  buckets: [HeatmapBucket!]!
}

type HeatmapBucket {
  startMs: Int!                    # Bucket start (e.g., 0ms, 10000ms, 20000ms)
  endMs: Int!                      # Bucket end (e.g., 9999ms, 19999ms, 29999ms)
  momentCount: Int!                # Number of moments in this bucket
  reactionCount: Int!              # Total reactions in this bucket
  intensity: Float!                # Normalized 0.0-1.0 (for heatmap color)
}

# ===== Query Extensions =====

extend type Query {
  # Get moments for a track
  trackMoments(
    trackId: ID!
    visibility: Visibility          # Filter: public, friends, private (defaults to public+friends)
    after: String
    limit: Int = 50
  ): MomentConnection!
  
  # Get moment heatmap (for progress bar overlay)
  trackMomentHeatmap(
    trackId: ID!
    bucketSizeMs: Int = 10000      # 10-second buckets by default
  ): MomentHeatmap!
  
  # Get moments by user
  userMoments(
    userId: ID!
    after: String
    limit: Int = 50
  ): MomentConnection!
  
  # Get moments from friends (social discovery)
  friendMoments(
    after: String
    limit: Int = 50
  ): MomentConnection!
  
  # Get single moment
  moment(id: ID!): Moment
}

# ===== Mutation Extensions =====

extend type Mutation {
  # Create moment
  createMoment(input: CreateMomentInput!): Moment!
  
  # Delete moment (only by owner)
  deleteMoment(id: ID!): Boolean!
  
  # React to moment
  reactToMoment(momentId: ID!, emoji: String!): Boolean!
  
  # Remove reaction
  removeReaction(momentId: ID!): Boolean!
}

input CreateMomentInput {
  trackId: ID!
  timestampMs: Int!
  comment: String!
  visibility: Visibility = FRIENDS
}
```

**Resolver Implementation:**

**Create Moment:**

```go
func (r *Resolver) CreateMoment(ctx context.Context, args struct{ Input CreateMomentInput }) (*domain.Moment, error) {
    userID := getUserIDFromContext(ctx)
    
    // Validate comment length
    if len(args.Input.Comment) == 0 || len(args.Input.Comment) > 500 {
        return nil, fmt.Errorf("comment must be 1-500 characters")
    }
    
    // Validate timestamp (must be >= 0)
    if args.Input.TimestampMs < 0 {
        return nil, fmt.Errorf("timestamp must be >= 0")
    }
    
    // Validate track exists (query Navidrome API)
    track, err := r.navidromeClient.GetTrack(ctx, args.Input.TrackID)
    if err != nil {
        return nil, fmt.Errorf("track not found")
    }
    
    // Validate timestamp doesn't exceed track duration
    if args.Input.TimestampMs > track.Duration*1000 {
        return nil, fmt.Errorf("timestamp exceeds track duration")
    }
    
    // Create moment
    moment := &domain.Moment{
        UserID:      userID,
        TrackID:     args.Input.TrackID,
        TimestampMs: args.Input.TimestampMs,
        Comment:     args.Input.Comment,
        Visibility:  args.Input.Visibility,
        CreatedAt:   time.Now(),
    }
    
    if err := r.momentService.Create(ctx, moment); err != nil {
        return nil, err
    }
    
    // Create activity (for feed)
    activity := &domain.Activity{
        UserID:       userID,
        ActivityType: "moment_create",
        EntityType:   "moment",
        EntityID:     moment.ID,
        Metadata: map[string]interface{}{
            "trackId":     args.Input.TrackID,
            "trackTitle":  track.Title,
            "timestampMs": args.Input.TimestampMs,
        },
        Timestamp: moment.CreatedAt,
    }
    r.activityService.Create(ctx, activity)
    
    return moment, nil
}
```

**Track Moment Heatmap:**

```go
func (r *Resolver) TrackMomentHeatmap(ctx context.Context, args struct {
    TrackID      string
    BucketSizeMs int
}) (*domain.MomentHeatmap, error) {
    // Get track duration
    track, err := r.navidromeClient.GetTrack(ctx, args.TrackID)
    if err != nil {
        return nil, err
    }
    
    // Generate buckets (e.g., [0-10s, 10-20s, 20-30s, ...])
    bucketSize := args.BucketSizeMs
    trackDurationMs := track.Duration * 1000
    numBuckets := (trackDurationMs / bucketSize) + 1
    
    buckets := make([]domain.HeatmapBucket, numBuckets)
    for i := 0; i < numBuckets; i++ {
        buckets[i] = domain.HeatmapBucket{
            StartMs: i * bucketSize,
            EndMs:   (i+1)*bucketSize - 1,
        }
    }
    
    // Query moments for track (with reaction counts)
    moments, err := r.momentService.GetByTrack(ctx, args.TrackID)
    if err != nil {
        return nil, err
    }
    
    // Aggregate moments into buckets
    for _, moment := range moments {
        bucketIndex := moment.TimestampMs / bucketSize
        if bucketIndex < numBuckets {
            buckets[bucketIndex].MomentCount++
            buckets[bucketIndex].ReactionCount += moment.ReactionCount
        }
    }
    
    // Normalize intensity (0.0-1.0)
    maxReactions := 0
    for _, bucket := range buckets {
        if bucket.ReactionCount > maxReactions {
            maxReactions = bucket.ReactionCount
        }
    }
    for i := range buckets {
        if maxReactions > 0 {
            buckets[i].Intensity = float64(buckets[i].ReactionCount) / float64(maxReactions)
        }
    }
    
    return &domain.MomentHeatmap{
        TrackID: args.TrackID,
        Buckets: buckets,
    }, nil
}
```

**Tasks:**
- [ ] Extend GraphQL schema
- [ ] Implement resolvers (createMoment, deleteMoment, trackMoments, trackMomentHeatmap)
- [ ] Add validation (comment length, timestamp bounds)
- [ ] Test mutations (create moment, add reaction)
- [ ] Test queries (get moments, get heatmap)

**Deliverable:** Moments GraphQL API

**Acceptance Criteria:**
- Can create moments with valid timestamps
- Can query moments for a track (filtered by visibility)
- Can generate heatmap (aggregated by 10s buckets)
- Can react to moments (emoji reactions)

---

### 3. UI: Moment Heatmap Overlay

**Goal:** Visualize moments on track progress bar (SoundCloud-style).

**Component:** `ui/src/audioplayer/MomentHeatmap.jsx`

```tsx
import React, { useEffect, useState } from 'react';
import { useGraphQL } from '../hooks/useGraphQL';
import './MomentHeatmap.css';

export const MomentHeatmap = ({ trackId, duration, currentTime, onSeek }) => {
    const [heatmap, setHeatmap] = useState(null);
    
    const { data, loading } = useGraphQL(`
        query GetHeatmap($trackId: ID!) {
            trackMomentHeatmap(trackId: $trackId, bucketSizeMs: 10000) {
                buckets {
                    startMs
                    endMs
                    momentCount
                    reactionCount
                    intensity
                }
            }
        }
    `, { trackId });
    
    useEffect(() => {
        if (data) setHeatmap(data.trackMomentHeatmap);
    }, [data]);
    
    if (loading || !heatmap) return <div className="heatmap-loading" />;
    
    return (
        <div className="heatmap-container">
            {heatmap.buckets.map((bucket, index) => {
                const width = ((bucket.endMs - bucket.startMs) / (duration * 1000)) * 100;
                const left = (bucket.startMs / (duration * 1000)) * 100;
                const opacity = 0.2 + (bucket.intensity * 0.8); // 0.2-1.0 opacity
                
                return (
                    <div
                        key={index}
                        className="heatmap-bucket"
                        style={{
                            left: `${left}%`,
                            width: `${width}%`,
                            opacity: opacity,
                            backgroundColor: bucket.momentCount > 0 ? '#ff5500' : 'transparent',
                        }}
                        onClick={() => onSeek(bucket.startMs / 1000)}
                        title={`${bucket.momentCount} moments, ${bucket.reactionCount} reactions`}
                    />
                );
            })}
        </div>
    );
};
```

**CSS:** `ui/src/audioplayer/MomentHeatmap.css`

```css
.heatmap-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    pointer-events: none;
}

.heatmap-bucket {
    position: absolute;
    height: 100%;
    cursor: pointer;
    pointer-events: auto;
    transition: opacity 0.2s;
}

.heatmap-bucket:hover {
    opacity: 1 !important;
}
```

**Integration:** Update `ui/src/audioplayer/Player.jsx`

```tsx
export const Player = ({ track }) => {
    const [currentTime, setCurrentTime] = useState(0);
    
    const handleSeek = (time) => {
        audioRef.current.currentTime = time;
        setCurrentTime(time);
    };
    
    return (
        <div className="player">
            <div className="progress-bar-container">
                <ProgressBar currentTime={currentTime} duration={track.duration} onSeek={handleSeek} />
                <MomentHeatmap 
                    trackId={track.id} 
                    duration={track.duration} 
                    currentTime={currentTime}
                    onSeek={handleSeek}
                />
            </div>
            {/* ... rest of player UI */}
        </div>
    );
};
```

**Tasks:**
- [ ] Create MomentHeatmap component
- [ ] Integrate into player UI (overlay on progress bar)
- [ ] Add click handler (seek to bucket start)
- [ ] Add hover tooltip (show moment/reaction counts)
- [ ] Test with real data (verify heatmap aligns with moments)

**Deliverable:** Moment heatmap visualization

**Acceptance Criteria:**
- Heatmap shows colored regions for moment clusters
- Intensity matches reaction counts (darker = more reactions)
- Clicking heatmap seeks to that position
- Hovering shows moment/reaction counts

---

### 4. UI: Moment Creation

**Goal:** Allow users to add moments at current playback position.

**Component:** `ui/src/audioplayer/MomentCreator.jsx`

```tsx
import React, { useState } from 'react';
import { TextField, Button, Select, MenuItem } from '@material-ui/core';
import { useMutation } from '../hooks/useGraphQL';

export const MomentCreator = ({ trackId, currentTime, onCreated }) => {
    const [comment, setComment] = useState('');
    const [visibility, setVisibility] = useState('FRIENDS');
    const [showForm, setShowForm] = useState(false);
    
    const [createMoment, { loading }] = useMutation(`
        mutation CreateMoment($input: CreateMomentInput!) {
            createMoment(input: $input) {
                id
                timestampMs
                comment
            }
        }
    `);
    
    const handleSubmit = async (e) => {
        e.preventDefault();
        
        if (comment.length === 0 || comment.length > 500) {
            alert('Comment must be 1-500 characters');
            return;
        }
        
        const result = await createMoment({
            variables: {
                input: {
                    trackId,
                    timestampMs: Math.floor(currentTime * 1000),
                    comment,
                    visibility,
                }
            }
        });
        
        if (result.data) {
            setComment('');
            setShowForm(false);
            onCreated(result.data.createMoment);
        }
    };
    
    if (!showForm) {
        return (
            <Button 
                variant="outlined" 
                size="small" 
                onClick={() => setShowForm(true)}
                className="add-moment-button"
            >
                💬 Add Moment
            </Button>
        );
    }
    
    return (
        <form onSubmit={handleSubmit} className="moment-creator-form">
            <TextField
                label={`Moment at ${formatTime(currentTime)}`}
                placeholder="What's happening here?"
                value={comment}
                onChange={(e) => setComment(e.target.value)}
                multiline
                rows={2}
                fullWidth
                inputProps={{ maxLength: 500 }}
                helperText={`${comment.length}/500`}
            />
            <div style={{ display: 'flex', justifyContent: 'space-between', marginTop: 8 }}>
                <Select value={visibility} onChange={(e) => setVisibility(e.target.value)}>
                    <MenuItem value="PUBLIC">Public</MenuItem>
                    <MenuItem value="FRIENDS">Friends</MenuItem>
                    <MenuItem value="PRIVATE">Private</MenuItem>
                </Select>
                <div>
                    <Button onClick={() => setShowForm(false)}>Cancel</Button>
                    <Button type="submit" variant="contained" color="primary" disabled={loading}>
                        Post
                    </Button>
                </div>
            </div>
        </form>
    );
};

function formatTime(seconds) {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${secs.toString().padStart(2, '0')}`;
}
```

**Integration:** Add to player UI

```tsx
export const Player = ({ track }) => {
    const [moments, setMoments] = useState([]);
    
    const handleMomentCreated = (moment) => {
        setMoments([...moments, moment]);
        // Refresh heatmap
    };
    
    return (
        <div className="player">
            {/* ... progress bar, heatmap ... */}
            <MomentCreator 
                trackId={track.id} 
                currentTime={currentTime}
                onCreated={handleMomentCreated}
            />
        </div>
    );
};
```

**Tasks:**
- [ ] Create MomentCreator component
- [ ] Add "Add Moment" button to player UI
- [ ] Add form (comment text, visibility select)
- [ ] Submit mutation (create moment)
- [ ] Show success message
- [ ] Refresh heatmap after creation

**Deliverable:** Moment creation UI

**Acceptance Criteria:**
- Can add moment at current playback position
- Comment is limited to 500 characters
- Can choose visibility (public/friends/private)
- New moment appears in heatmap immediately

---

### 5. UI: Moment Browser

**Goal:** Show all moments for a track (browse and seek to moments).

**Component:** `ui/src/song/MomentBrowser.jsx`

```tsx
export const MomentBrowser = ({ trackId, onSeek }) => {
    const { data, loading, fetchMore } = useGraphQL(`
        query GetTrackMoments($trackId: ID!, $after: String) {
            trackMoments(trackId: $trackId, after: $after, limit: 20) {
                edges {
                    node {
                        id
                        user {
                            username
                            displayName
                            avatarUrl
                        }
                        timestampMs
                        comment
                        reactionCount
                        topReactions {
                            emoji
                            count
                        }
                        myReaction
                        createdAt
                    }
                    cursor
                }
                pageInfo {
                    hasNextPage
                    endCursor
                }
            }
        }
    `, { trackId });
    
    const moments = data?.trackMoments?.edges?.map(edge => edge.node) || [];
    
    return (
        <div className="moment-browser">
            <Typography variant="h6">Moments ({moments.length})</Typography>
            {moments.map(moment => (
                <MomentCard 
                    key={moment.id} 
                    moment={moment} 
                    onSeek={() => onSeek(moment.timestampMs / 1000)}
                />
            ))}
            {data?.trackMoments?.pageInfo?.hasNextPage && (
                <Button onClick={() => fetchMore({ after: data.trackMoments.pageInfo.endCursor })}>
                    Load More
                </Button>
            )}
        </div>
    );
};

const MomentCard = ({ moment, onSeek }) => {
    const [myReaction, setMyReaction] = useState(moment.myReaction);
    
    const handleReact = async (emoji) => {
        // Call reactToMoment mutation
        setMyReaction(emoji);
    };
    
    return (
        <Card className="moment-card">
            <CardContent>
                <div className="moment-header">
                    <Avatar src={moment.user.avatarUrl} />
                    <div>
                        <Typography variant="body2"><strong>{moment.user.displayName}</strong></Typography>
                        <Typography variant="caption" color="textSecondary">
                            <span onClick={onSeek} style={{ cursor: 'pointer', color: '#1976d2' }}>
                                {formatTime(moment.timestampMs / 1000)}
                            </span>
                            {' · '}
                            {formatTimestamp(moment.createdAt)}
                        </Typography>
                    </div>
                </div>
                <Typography variant="body1" style={{ marginTop: 8 }}>
                    {moment.comment}
                </Typography>
                <div className="moment-reactions" style={{ marginTop: 8 }}>
                    {moment.topReactions.map(reaction => (
                        <Chip
                            key={reaction.emoji}
                            label={`${reaction.emoji} ${reaction.count}`}
                            size="small"
                            onClick={() => handleReact(reaction.emoji)}
                            color={myReaction === reaction.emoji ? 'primary' : 'default'}
                        />
                    ))}
                    <IconButton size="small" onClick={() => handleReact('👏')}>
                        <AddReactionIcon />
                    </IconButton>
                </div>
            </CardContent>
        </Card>
    );
};
```

**Tasks:**
- [ ] Create MomentBrowser component
- [ ] Create MomentCard component
- [ ] Add reaction buttons (emoji chips)
- [ ] Add seek-on-click (click timestamp → seek to position)
- [ ] Add pagination (load more moments)

**Deliverable:** Moment browsing UI

**Acceptance Criteria:**
- Can view all moments for a track
- Can click timestamp to seek to moment position
- Can react to moments (emoji reactions)
- Moments are sorted by timestamp (oldest first)

---

### 6. Privacy & Moderation

**Goal:** Implement privacy controls and basic moderation.

**Privacy Rules:**

```go
// social-service/internal/service/privacy_service.go

func (s *PrivacyService) CanViewMoment(ctx context.Context, viewerID string, moment *domain.Moment) (bool, error) {
    // Public moments: anyone can view
    if moment.Visibility == "public" {
        return true, nil
    }
    
    // Private moments: only owner can view
    if moment.Visibility == "private" {
        return viewerID == moment.UserID, nil
    }
    
    // Friends moments: owner and friends can view
    if moment.Visibility == "friends" {
        if viewerID == moment.UserID {
            return true, nil
        }
        
        // Check if viewer is following owner (or mutual follow)
        isFollowing, err := s.followRepo.IsFollowing(ctx, viewerID, moment.UserID)
        return isFollowing, err
    }
    
    return false, nil
}
```

**Filter moments by privacy in resolvers:**

```go
func (r *Resolver) TrackMoments(ctx context.Context, args struct {
    TrackID    string
    Visibility *string
    After      *string
    Limit      *int
}) (*MomentConnection, error) {
    viewerID := getUserIDFromContext(ctx)
    
    // Get all moments for track
    allMoments, err := r.momentService.GetByTrack(ctx, args.TrackID)
    if err != nil {
        return nil, err
    }
    
    // Filter by privacy
    var visibleMoments []domain.Moment
    for _, moment := range allMoments {
        canView, err := r.privacyService.CanViewMoment(ctx, viewerID, &moment)
        if err == nil && canView {
            visibleMoments = append(visibleMoments, moment)
        }
    }
    
    // Paginate and return
    return paginateMoments(visibleMoments, args.After, args.Limit), nil
}
```

**Basic Moderation (flag moments):**

```sql
-- Add to moments table
ALTER TABLE moments ADD COLUMN flagged BOOLEAN DEFAULT FALSE;
ALTER TABLE moments ADD COLUMN flag_reason TEXT;
```

```graphql
extend type Mutation {
  flagMoment(momentId: ID!, reason: String!): Boolean!
}
```

**Tasks:**
- [ ] Implement privacy checks in resolvers
- [ ] Add flag mutation (users can flag inappropriate moments)
- [ ] Add admin dashboard for reviewing flagged moments (defer UI to Phase 7)
- [ ] Document privacy rules

**Deliverable:** Privacy controls + basic moderation

**Acceptance Criteria:**
- Private moments are only visible to owner
- Friends moments are only visible to owner and followers
- Public moments are visible to everyone
- Users can flag inappropriate moments

---

## Operational Needs

### Infrastructure
- No new infrastructure (uses existing Social Service + PostgreSQL)
- Consider caching heatmaps in Redis (expensive aggregation query)

### Monitoring
- Add metrics for moments:
  - `social_moments_created_total` (counter)
  - `social_moment_reactions_total` (counter)
  - `social_moment_heatmap_generation_seconds` (histogram)
- Alert if heatmap generation >500ms (optimization needed)

### Data Retention
- Moments never expire (persistent annotations)
- Flagged moments reviewed by moderators (defer deletion to Phase 7)

---

## Risks & Mitigations

### Risk: Heatmap generation is slow (N moments on popular tracks)
**Likelihood:** Medium  
**Impact:** High  
**Mitigation:**
- Use materialized view for pre-computed stats
- Cache heatmaps in Redis (10-minute TTL)
- Optimize query with covering index (track_id, timestamp_ms, reaction_count)

### Risk: Spam moments (users add many low-quality comments)
**Likelihood:** Medium  
**Impact:** Medium  
**Mitigation:**
- Rate limit moment creation (max 10 moments per user per hour)
- Add flag/report feature (community moderation)
- Admin dashboard for reviewing flagged moments (Phase 7)

### Risk: Privacy violations (see friends-only moments when not following)
**Likelihood:** Low  
**Impact:** High  
**Mitigation:**
- Privacy checks in every resolver
- Integration tests for privacy (can't see private moments, can't see friends moments unless following)
- Security audit

---

## Definition of Done

### Must Have
- ✅ Moments table created (with reactions)
- ✅ GraphQL API for moments (create, query, react)
- ✅ Moment heatmap visualization (progress bar overlay)
- ✅ Moment creation UI (add moment at current time)
- ✅ Moment browser UI (view all moments, seek to moments)
- ✅ Privacy controls (public/friends/private)
- ✅ Deployed and tested end-to-end

### Should Have
- ✅ Reaction aggregation (top 3 emoji)
- ✅ Heatmap caching (Redis)
- ✅ Flag/report feature (basic moderation)

### Could Have
- Threaded replies (defer to later phase)
- Rich media in comments (images, GIFs) - defer
- Notifications (new reaction on your moment) - defer

---

## Success Metrics

- **Moment Adoption:** >15% of users create at least 1 moment
- **Engagement:** >25% of users view moments on tracks
- **Heatmap Performance:** P95 <300ms for heatmap generation
- **Reaction Rate:** >30% of moments receive at least 1 reaction

---

## Next Steps (Transition to Phase 5)

After Phase 4 is complete:
1. Monitor moment engagement metrics (2 weeks)
2. Collect user feedback on moments UI
3. Create Phase 5 branch: `feature/analytics-expansion`
4. Begin work on derived metrics (completion %, skip patterns)

---

**Estimated Effort:**
- Senior Backend Engineer (Go): 60 hours (moments schema, GraphQL resolvers, privacy logic)
- Mid-Level Backend Engineer (Go): 40 hours (heatmap aggregation, caching)
- Senior Frontend Engineer (React): 80 hours (heatmap visualization, moment creator, moment browser)
- Mid-Level Frontend Engineer (React): 60 hours (UI polish, animations, responsiveness)
- **Total: 240 hours (~6 person-weeks with 40-hour weeks)**

---

**End of Phase 4**
