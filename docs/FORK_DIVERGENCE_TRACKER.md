# Fork Divergence Tracker

This document tracks all modifications made to the upstream Navidrome codebase. It helps with conflict resolution during upstream syncs and identifies changes that could be contributed back to upstream.

## Overview

**Fork Base:** Navidrome v0.52.x (as of December 2025)  
**Last Upstream Sync:** Initial fork creation

## Change Categories

- 🆕 **New File** - File added by fork (no upstream equivalent)
- ✏️ **Modified** - Existing file modified
- 🔧 **Configuration** - Configuration or build changes
- 📚 **Documentation** - Documentation additions/changes

## Upstreamable Status

- ✅ **Upstreamable** - Generic improvement, could be contributed back
- ❌ **Fork-Specific** - Specific to social/analytics features
- ⚠️ **Partial** - Parts could be upstreamed

---

## Phase 0: Repository Audit (Current)

### Documentation

| File | Type | Upstreamable | Description |
|------|------|--------------|-------------|
| `docs/DEVELOPMENT_SETUP.md` | 🆕 | ⚠️ Partial | Development setup guide (generic parts could be upstream) |
| `docs/TESTING_GUIDE.md` | 🆕 | ⚠️ Partial | Testing conventions documentation |
| `docs/TEST_COVERAGE_REPORT.md` | 🆕 | ❌ | Fork-specific coverage analysis |
| `docs/FORK_MANAGEMENT.md` | 🆕 | ❌ | Fork management strategy |
| `docs/FORK_DIVERGENCE_TRACKER.md` | 🆕 | ❌ | This file |
| `docs/DEPENDENCIES.md` | 🆕 | ⚠️ Partial | Dependency analysis |
| `docs/CODE_STYLE_GUIDE.md` | 🆕 | ✅ | Could benefit upstream |
| `docs/DEPLOYMENT.md` | 🆕 | ⚠️ Partial | Deployment documentation |
| `docs/UPSTREAM_ALIGNMENT.md` | 🆕 | ❌ | Fork-specific upstream analysis |
| `docs/architecture/` | 🆕 | ⚠️ Partial | Architecture documentation |

### CI/CD

| File | Type | Upstreamable | Description |
|------|------|--------------|-------------|
| `.github/workflows/ci.yml` | 🆕 | ❌ | Fork CI pipeline |
| `.github/workflows/sync-upstream.yml` | 🆕 | ❌ | Upstream sync automation |

### Planning Documents

| File | Type | Upstreamable | Description |
|------|------|--------------|-------------|
| `MASTER_PLANNING_DOCUMENT.md` | 🆕 | ❌ | Fork roadmap |
| `ARCHITECTURE_SNAPSHOT.md` | 🆕 | ⚠️ Partial | Codebase analysis |
| `INTEGRATION_PATTERN_DECISION.md` | 🆕 | ❌ | Architecture decision |
| `PHASE_*.md` | 🆕 | ❌ | Phase documentation |

---

## Phase 2: Telemetry Features (Planned)

### Database

| File | Type | Upstreamable | Description | Conflict Risk |
|------|------|--------------|-------------|---------------|
| `db/migrations/XXXXXX_add_listen_event.sql` | 🆕 | ⚠️ Partial | Listen event table | Low - new file |
| `model/listen_event.go` | 🆕 | ⚠️ Partial | Listen event model | Low - new file |
| `model/datastore.go` | ✏️ | ⚠️ Partial | Add ListenEventRepository | Medium - shared file |

### Core

| File | Type | Upstreamable | Description | Conflict Risk |
|------|------|--------------|-------------|---------------|
| `persistence/listen_event_repository.go` | 🆕 | ⚠️ Partial | Repository implementation | Low - new file |
| `core/scrobbler/play_tracker.go` | ✏️ | ❌ | Event emission hooks | High - frequently modified |

### API

| File | Type | Upstreamable | Description | Conflict Risk |
|------|------|--------------|-------------|---------------|
| `server/nativeapi/events_api.go` | 🆕 | ❌ | Event ingestion endpoint | Low - new file |
| `server/nativeapi/native_api.go` | ✏️ | ❌ | Route registration | Medium - shared file |

### UI

| File | Type | Upstreamable | Description | Conflict Risk |
|------|------|--------------|-------------|---------------|
| `ui/src/player/TelemetryEmitter.js` | 🆕 | ❌ | Telemetry event emission | Low - new file |
| `ui/src/player/Player.js` | ✏️ | ❌ | Integrate telemetry hooks | Medium - shared file |

---

## Conflict History

### Template

```markdown
### Sync Date: YYYY-MM-DD

**Upstream Commit:** <hash>

| File | Conflict Type | Resolution | Notes |
|------|---------------|------------|-------|
| `path/to/file.go` | Code conflict | Accepted upstream, adapted fork | Description |
```

### Sync History

_No syncs yet - initial fork creation_

---

## Maintenance Notes

### High-Risk Files

These files are likely to conflict during upstream syncs:

1. **`model/datastore.go`** - Central interface, frequently updated
2. **`core/scrobbler/play_tracker.go`** - Core telemetry point, may be refactored
3. **`server/nativeapi/native_api.go`** - API routing, new endpoints added
4. **`go.mod`** - Dependencies updated regularly

### Mitigation Strategies

1. **Prefer new files** - Add features in new files rather than modifying existing ones
2. **Use interfaces** - Extend functionality via new interfaces
3. **Feature flags** - Enable fork features conditionally
4. **Regular syncs** - Sync weekly to catch conflicts early

---

## Statistics

| Category | Count |
|----------|-------|
| New Files | 15+ (documentation) |
| Modified Files | 0 (Phase 0 only adds docs) |
| Upstream Syncs | 0 |
| Conflict Resolutions | 0 |

---

**Last Updated:** December 2025
