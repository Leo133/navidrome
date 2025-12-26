# Fork Management Strategy

This document describes the strategy for maintaining the Navidrome fork, including branching, merging, and upstream synchronization.

## Overview

This fork extends Navidrome with social features and deep listening analytics while maintaining compatibility with upstream Navidrome releases.

### Goals

1. **Minimize divergence** - Keep changes as small and isolated as possible
2. **Stay current** - Regularly sync with upstream releases
3. **Maintain quality** - All changes must pass tests and code review
4. **Enable upstreaming** - Generic improvements should be contributed back

## Branch Structure

```
upstream/master    ─────────────────────────────────────────────────────►
                          │                    │
                          │ (weekly sync)      │ (weekly sync)
                          ▼                    ▼
develop            ───────●────────────────────●─────────────────────────►
                          │                    │
                          │ feature branches   │
                          ▼                    ▼
feature/telemetry  ───────●────────────────────●
                                               │
                                               │ (merge to develop)
                                               ▼
```

### Branch Types

| Branch | Purpose | Lifetime |
|--------|---------|----------|
| `develop` | Main development branch | Permanent |
| `upstream-master` | Tracks navidrome/navidrome master | Permanent |
| `feature/*` | Feature development | Until merged |
| `release/*` | Release preparation | Until released |
| `hotfix/*` | Critical bug fixes | Until merged |

### Branch Naming

- `feature/telemetry-v1` - New feature development
- `feature/social-graph` - New feature development
- `release/0.52.0-social.1` - Release preparation
- `hotfix/fix-critical-bug` - Emergency fixes

## Workflow

### Daily Development

1. **Create feature branch from `develop`**
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b feature/my-feature
   ```

2. **Make changes and commit**
   ```bash
   git add .
   git commit -m "feat: add listen event table"
   ```

3. **Push and create PR**
   ```bash
   git push origin feature/my-feature
   # Create PR on GitHub targeting `develop`
   ```

4. **Merge after review**
   - Squash merge preferred for clean history
   - Delete feature branch after merge

### Weekly Upstream Sync

An automated GitHub Action runs weekly to sync with upstream:

1. **Fetch upstream changes**
   ```bash
   git fetch upstream master
   ```

2. **Create sync branch**
   ```bash
   git checkout -b sync/upstream-$(date +%Y%m%d) upstream/master
   ```

3. **Rebase develop onto upstream**
   ```bash
   git checkout develop
   git rebase sync/upstream-$(date +%Y%m%d)
   ```

4. **Resolve conflicts** (if any)
   - Prioritize upstream changes unless they break fork features
   - Document conflict resolution in PR

5. **Test**
   ```bash
   make test
   make lint
   ```

6. **Push**
   ```bash
   git push origin develop --force-with-lease
   ```

### Manual Upstream Sync

If automated sync fails or you need to sync manually:

```bash
# Add upstream remote (one-time)
git remote add upstream https://github.com/navidrome/navidrome.git

# Fetch latest upstream
git fetch upstream master

# Switch to develop
git checkout develop

# Rebase onto upstream
git rebase upstream/master

# If conflicts occur, resolve them:
# 1. Edit conflicting files
# 2. git add <resolved-files>
# 3. git rebase --continue

# Run tests
make test

# Push (force required after rebase)
git push origin develop --force-with-lease
```

### Release Process

1. **Create release branch**
   ```bash
   git checkout develop
   git checkout -b release/0.52.0-social.1
   ```

2. **Update version**
   - Edit `consts/version.go`
   - Update CHANGELOG.md

3. **Final testing**
   ```bash
   make testall
   make lintall
   ```

4. **Create PR and merge to develop**

5. **Tag release**
   ```bash
   git checkout develop
   git pull origin develop
   git tag v0.52.0-social.1
   git push origin v0.52.0-social.1
   ```

6. **CI builds and publishes Docker image**

## Versioning

### Version Scheme

Fork versions follow this pattern:

```
<navidrome-version>-social.<fork-patch>
```

Examples:
- `0.52.0-social.1` - First fork release based on Navidrome 0.52.0
- `0.52.0-social.2` - Second fork release based on Navidrome 0.52.0
- `0.52.5-social.1` - First fork release after syncing to Navidrome 0.52.5

### Version in Code

Update `consts/version.go`:
```go
const Version = "0.52.0-social.1"
```

## Conflict Resolution

### Common Conflict Areas

1. **Database migrations** - Fork adds new tables, upstream adds migrations
2. **PlayTracker** - Fork extends functionality, upstream may refactor
3. **API endpoints** - Fork adds new endpoints, upstream may change routing
4. **UI components** - Fork adds social features, upstream may update UI

### Resolution Guidelines

| Conflict Type | Resolution Strategy |
|---------------|---------------------|
| New upstream feature | Accept upstream, integrate with fork features |
| Upstream refactor | Accept upstream, adapt fork code to new patterns |
| Database schema | Ensure migration order is correct, test carefully |
| Configuration | Merge config options, avoid namespace collisions |
| Dependencies | Follow upstream for shared deps, pin fork-specific deps |

### Conflict Documentation

Document all conflict resolutions in the PR:

```markdown
## Conflict Resolution

- **file.go**: Upstream refactored function signature, updated fork code to match
- **migrations/**: Reordered fork migrations to run after upstream migrations
```

## Fork Divergence Tracker

Track all changes to Navidrome code in `docs/FORK_DIVERGENCE_TRACKER.md`.

For each change:
- Note the file and type of change
- Mark if it's upstreamable
- Track conflict history

## CI/CD Integration

### PR Checks

All PRs must pass:
- `make test` - Unit tests
- `make lint` - Code linting
- `make build` - Build verification

### Sync Workflow

Automated weekly sync:
1. Fetches upstream
2. Creates sync PR
3. Runs tests
4. Notifies maintainers if conflicts exist

## Best Practices

### Do's

✅ **Keep changes minimal** - Prefer adding new files over modifying existing ones
✅ **Document changes** - Update FORK_DIVERGENCE_TRACKER.md for every modification
✅ **Test after sync** - Always run full test suite after upstream sync
✅ **Use feature flags** - Enable/disable fork features via configuration
✅ **Communicate with upstream** - Report bugs found during fork development

### Don'ts

❌ **Don't rewrite history** - Avoid force-pushing to shared branches
❌ **Don't ignore conflicts** - Resolve carefully, test thoroughly
❌ **Don't accumulate debt** - Sync regularly, don't let fork drift too far
❌ **Don't break compatibility** - Maintain Subsonic API compatibility

## Troubleshooting

### Sync Fails Due to Conflicts

1. Create a manual sync branch
2. Carefully resolve conflicts
3. Test thoroughly
4. Get review before merging

### Tests Fail After Sync

1. Check if upstream changed test data or fixtures
2. Check if upstream changed API contracts
3. Update fork tests to match new behavior

### Build Fails After Sync

1. Check for dependency changes in `go.mod`
2. Check for removed/renamed packages
3. Run `go mod tidy` to update dependencies

## References

- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Semantic Versioning](https://semver.org/)
