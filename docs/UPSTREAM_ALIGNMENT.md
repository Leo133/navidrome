# Upstream Alignment Assessment

**Assessment Date:** December 2025  
**Navidrome Version:** v0.52.x  
**Fork Base Commit:** Initial fork creation

## Overview

This document analyzes recent Navidrome upstream changes and assesses potential conflicts with the fork's planned features.

## Recent Upstream Changes (Last 3 Months)

### Major Changes

#### 1. Scrobbles Table Addition (December 2024)

**Migration:** `20251206013022_create_scrobbles_table.sql`

**Schema:**
```sql
CREATE TABLE scrobbles (
    media_file_id VARCHAR(255) NOT NULL,
    user_id VARCHAR(255) NOT NULL,
    submission_time INTEGER NOT NULL
);
CREATE INDEX scrobbles_date ON scrobbles(submission_time);
```

**Impact on Fork:**
- ✅ Positive: Foundation for listen history
- ⚠️ Consideration: Our `listen_event` table is more detailed
- 📋 Action: Build on this, don't replace it

#### 2. Multi-Library Support (Recent)

**Changes:**
- Added `library` table
- Added `library_id` foreign key to `media_file`, `album`, `artist`, `folder`
- User-library access control via `user_library` join table

**Impact on Fork:**
- ✅ No conflict: Social features work at user level, not library level
- 📋 Action: Ensure telemetry events include `library_id` context

#### 3. Plugin System Enhancements

**Changes:**
- Improved WASM runtime (wazero update)
- Added WebSocket host service
- Added SubsonicAPI host service

**Impact on Fork:**
- ✅ Positive: Better plugin capabilities
- ⚠️ Consideration: Could potentially implement social as plugin (not recommended for performance)

### Minor Changes

1. **UI Updates:** React version updates, component improvements
2. **Transcoding:** Performance optimizations
3. **Scanner:** Bug fixes for edge cases
4. **API:** Minor OpenSubsonic compatibility improvements

## Conflict Risk Assessment

### High-Risk Files

These files are modified by both upstream and our planned changes:

| File | Upstream Activity | Fork Changes | Risk | Mitigation |
|------|-------------------|--------------|------|------------|
| `model/datastore.go` | Medium (new repos) | Add ListenEventRepository | Medium | Use interface extension |
| `core/scrobbler/play_tracker.go` | Medium | Add event emission | High | Minimize changes, use hooks |
| `server/nativeapi/native_api.go` | Low | Add routes | Low | Register new routes only |

### Low-Risk Areas

| Area | Reason |
|------|--------|
| `docs/` | New directory, no conflicts |
| `db/migrations/` | New files only, ordered by timestamp |
| External services | Completely separate codebases |

## Roadmap Alignment

### Navidrome Roadmap (from GitHub Issues)

| Feature | Status | Impact on Fork |
|---------|--------|----------------|
| Smart Playlists | In Progress | No conflict |
| Improved Lyrics | Planned | No conflict |
| Better Search | Planned | Potential synergy |
| Mobile PWA | Discussed | UI changes may require adaptation |

### Collaboration Opportunities

Features we could contribute back to upstream:

1. **ListenEventRepository (partial)**
   - Generic event storage (without social-specific logic)
   - Could benefit upstream analytics

2. **Improved Play Tracking**
   - More detailed scrobble data
   - Interval-based tracking

3. **Privacy Controls**
   - Incognito mode
   - Per-device opt-out

## Rebase Strategy

### Weekly Sync Process

1. **Fetch upstream**
   ```bash
   git fetch upstream master
   ```

2. **Review changes**
   ```bash
   git log develop..upstream/master --oneline
   ```

3. **Assess conflicts**
   - Check files modified in both branches
   - Review migration order

4. **Rebase**
   ```bash
   git checkout develop
   git rebase upstream/master
   ```

5. **Test**
   ```bash
   make test
   make lint
   ```

6. **Push**
   ```bash
   git push origin develop --force-with-lease
   ```

### Conflict Resolution Priority

1. **Accept upstream changes** unless they break fork features
2. **Adapt fork code** to match new patterns
3. **Document** all conflict resolutions
4. **Test thoroughly** after merge

## Monitoring Upstream

### GitHub Watch List

Watch these upstream files for changes:

- `model/datastore.go`
- `core/scrobbler/*`
- `server/nativeapi/*`
- `db/migrations/*`
- `go.mod`

### Changelog Review

Review each upstream release:
1. Read release notes
2. Check breaking changes
3. Update fork documentation
4. Plan rebase timeline

## Version Compatibility

### Navidrome API Versions

| API | Version | Status |
|-----|---------|--------|
| Subsonic | 1.16.1 | Stable |
| OpenSubsonic | 1.0 | Stable |
| Native REST | - | Evolving |

### Database Schema Compatibility

Our migrations run after upstream migrations:
- Upstream: `20251206013022_create_scrobbles_table.sql`
- Fork: `20251220000001_create_listen_event_table.sql` (future)

## Action Items

### Before Phase 2 (Telemetry)

- [ ] Review upstream PlayTracker changes
- [ ] Check for new scrobble-related features
- [ ] Assess migration compatibility

### Before Each Phase

- [ ] Sync with upstream
- [ ] Review conflict risk
- [ ] Update this document

## References

- [Navidrome GitHub Repository](https://github.com/navidrome/navidrome)
- [Navidrome Releases](https://github.com/navidrome/navidrome/releases)
- [Navidrome Discussions](https://github.com/navidrome/navidrome/discussions)
- [Navidrome Discord](https://discord.gg/xh7j7yF)
