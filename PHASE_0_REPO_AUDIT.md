# Phase 0: Repository Audit & Upstream Alignment
**Duration:** 1-2 weeks  
**Team Size:** 2-3 engineers  
**Dependencies:** None (prerequisite for all other phases)

---

## Objective

Establish a **solid foundation** for the fork by:
1. Deeply understanding Navidrome's codebase, conventions, and extension points
2. Setting up development environment with proper tooling
3. Creating automated tests and CI pipeline for fork maintenance
4. Documenting architecture and creating contribution guidelines
5. Aligning with upstream (evaluate recent changes, plan rebase strategy)

---

## Scope

### In Scope
- ✅ Complete code audit (file-by-file walkthrough of core modules)
- ✅ Test coverage analysis (identify untested critical paths)
- ✅ Development environment setup (Docker, IDE, linters, formatters)
- ✅ Fork management strategy (branching, rebasing, upstream sync)
- ✅ CI/CD pipeline for fork (tests, builds, deployments)
- ✅ Architecture documentation (diagrams, data flow, API contracts)
- ✅ Code style guide (Go conventions, naming, error handling)
- ✅ Dependency analysis (identify potential conflicts with planned changes)

### Out of Scope
- ❌ Feature development (no new code yet)
- ❌ Performance profiling (covered in Phase 1)
- ❌ User research or product design
- ❌ Upstream contributions (may happen later, but not required now)

---

## Key Technical Work Items

### 1. Development Environment Setup

**Tasks:**
- [ ] Clone Navidrome repository and create fork
- [ ] Install dependencies: Go 1.25+, Node.js 20+, FFmpeg, SQLite
- [ ] Set up IDE (VSCode with Go extension, recommended settings)
- [ ] Configure linters: golangci-lint, eslint, prettier
- [ ] Run full test suite: `make test` (Go) and `npm test` (UI)
- [ ] Build binary: `make build`
- [ ] Run in development mode: `make run` (hot reload with reflex)
- [ ] Access web UI: http://localhost:4533
- [ ] Create test data: `make testdata` (populate DB with sample library)

**Deliverable:** `DEVELOPMENT_SETUP.md` guide

**Acceptance Criteria:**
- All team members can build and run Navidrome locally
- Test suite passes without errors
- Hot reload works for both backend and frontend

---

### 2. Code Audit & Architecture Mapping

**Tasks:**
- [ ] Read all files in `/model/` (domain model definitions)
- [ ] Read all files in `/persistence/` (SQL repository implementations)
- [ ] Read all files in `/core/` (business logic layer)
- [ ] Read all files in `/server/` (API layer)
- [ ] Read `/core/scrobbler/play_tracker.go` in detail (critical for telemetry)
- [ ] Read `/server/events/sse.go` (SSE broker for real-time updates)
- [ ] Read `/plugins/manager.go` (plugin system, may use for social features)
- [ ] Document data flow for key operations:
  - User login → JWT generation → authenticated request
  - Library scan → metadata extraction → DB insert → SSE event
  - Stream request → transcoding decision → cache lookup → FFmpeg → response
  - Scrobble submit → play count increment → annotation update → external scrobbler dispatch
- [ ] Identify potential conflicts with planned changes (e.g., tables we'll modify)
- [ ] Map all API endpoints (Subsonic + Native)

**Deliverable:** `ARCHITECTURE_MAP.md` (expands on snapshot, with detailed diagrams)

**Acceptance Criteria:**
- Team has shared understanding of codebase structure
- Key data flows are documented with sequence diagrams
- Potential conflict areas are identified (e.g., annotation table, scrobble flow)

---

### 3. Test Coverage Analysis

**Tasks:**
- [ ] Run coverage report: `make test-coverage`
- [ ] Identify modules with <50% coverage
- [ ] Focus on critical paths:
  - Authentication (server/auth.go)
  - Scrobbling (core/scrobbler/play_tracker.go)
  - Persistence (persistence/sql_annotations.go)
  - API handlers (server/subsonic/media_annotation.go)
- [ ] Document testing conventions:
  - Ginkgo/Gomega syntax (BDD-style)
  - Mock usage (MockDataStore, MockRepository)
  - Test fixtures (testdata/ directories)
- [ ] Create test plan for new features (template for Phase 1+)

**Deliverable:** `TEST_COVERAGE_REPORT.md` + `TESTING_GUIDE.md`

**Acceptance Criteria:**
- Coverage report generated (HTML + JSON)
- Critical paths have >70% coverage (if not, add tests)
- Testing guide covers unit tests, integration tests, mocking

---

### 4. Fork Management Strategy

**Tasks:**
- [ ] Create fork organization on GitHub (e.g., `yourorg/navidrome`)
- [ ] Set up branch structure:
  - `upstream-master` (tracks navidrome/navidrome master)
  - `develop` (main development branch)
  - `feature/*` (feature branches)
  - `release/*` (release branches)
- [ ] Configure GitHub Actions:
  - On push to `develop`: run tests, build binary, publish Docker image
  - On pull request: run tests, check formatting, run linters
  - Weekly: auto-sync with upstream (create PR if changes detected)
- [ ] Document rebase workflow:
  - Fetch upstream: `git fetch upstream`
  - Rebase onto upstream: `git rebase upstream/master`
  - Resolve conflicts (prioritize upstream changes unless breaking)
  - Test after rebase: `make test`
  - Push to fork: `git push origin develop --force-with-lease`
- [ ] Set up semantic versioning:
  - Fork version: `0.52.0+social.1` (Navidrome 0.52.0 + social fork patch 1)
  - Update version in `consts/version.go`

**Deliverable:** `FORK_MANAGEMENT.md` + GitHub Actions workflows

**Acceptance Criteria:**
- Fork is connected to upstream as a remote
- CI pipeline runs on every push
- Weekly sync job creates PR with upstream changes
- Version scheme is documented

---

### 5. Dependency Analysis

**Tasks:**
- [ ] Review `go.mod` (113 dependencies as of Dec 2025)
- [ ] Identify dependencies we'll add:
  - Message queue client: `github.com/nats-io/nats.go` (if using NATS)
  - GraphQL server: `github.com/99designs/gqlgen` (for Analytics/Social APIs)
  - Time-series DB driver: `github.com/jackc/pgx/v5` (for TimescaleDB)
- [ ] Check for version conflicts
- [ ] Document dependency update policy:
  - Follow upstream for shared deps (Go version, chi, sqlite3, etc.)
  - Pin fork-specific deps to stable versions
  - Use Dependabot for automated updates
- [ ] Create `go.work` file for multi-module setup (if needed for Analytics/Social services)

**Deliverable:** `DEPENDENCIES.md` + updated `go.mod` (no changes yet, just documentation)

**Acceptance Criteria:**
- All dependencies are documented with rationale
- No known version conflicts with upstream
- Dependency update policy is clear

---

### 6. CI/CD Pipeline Setup

**Tasks:**
- [ ] Create GitHub Actions workflow: `.github/workflows/ci.yml`
  - Jobs: test, lint, build, docker-build
  - Trigger: on push to `develop`, on PR to `develop`
- [ ] Create GitHub Actions workflow: `.github/workflows/sync-upstream.yml`
  - Job: fetch upstream, create PR if changes exist
  - Trigger: weekly cron job
- [ ] Set up Docker Hub repository for fork images
- [ ] Configure image tagging:
  - `latest` → latest commit on `develop`
  - `v0.52.0-social.1` → tagged releases
  - `pr-123` → pull request builds (for testing)
- [ ] Set up staging environment:
  - Deploy `develop` branch to staging on every push
  - Accessible at `https://staging.yourapp.com`
  - Reset DB daily (automated job)

**Deliverable:** `.github/workflows/` directory + `DEPLOYMENT.md`

**Acceptance Criteria:**
- CI pipeline passes on `develop` branch
- Docker images are built and published automatically
- Staging environment is accessible and up-to-date

---

### 7. Architecture Documentation

**Tasks:**
- [ ] Create high-level system diagram (current state)
- [ ] Create data model diagram (ERD for existing tables)
- [ ] Create sequence diagrams for key flows:
  - User authentication
  - Track streaming
  - Scrobble submission
  - Library scan
- [ ] Create component interaction diagram (layers, dependencies)
- [ ] Document extension points (where we'll add new code):
  - New repository interfaces (ListenEventRepository)
  - New API endpoints (/api/events/listen)
  - New database tables (listen_event)
  - New services (Analytics, Social)
- [ ] Create target state diagram (with new components in gray)

**Deliverable:** `docs/architecture/` directory with diagrams (Mermaid or PlantUML)

**Acceptance Criteria:**
- Diagrams are version-controlled and easily editable (text-based)
- Diagrams cover current state and planned changes
- Team reviews and approves diagrams

---

### 8. Code Style Guide

**Tasks:**
- [ ] Document Go conventions:
  - Error handling: always check errors, wrap with context
  - Logging: use structured logging (log.Info(ctx, "message", "key", value))
  - Context: always pass context as first parameter
  - Naming: follow Go naming conventions (CamelCase for exported, camelCase for private)
  - File organization: group by feature (not by layer)
- [ ] Document React conventions:
  - Component structure: functional components with hooks
  - State management: Redux for global state, useState for local
  - Styling: Material-UI with makeStyles
  - Testing: Jest + React Testing Library
- [ ] Configure linters:
  - Go: `.golangci.yml` (already exists, review settings)
  - TypeScript: `.eslintrc` (already exists, review settings)
  - Prettier: `prettier.config.js` (already exists)
- [ ] Run linters on codebase: `make lint` (Go) and `npm run lint` (UI)

**Deliverable:** `CODE_STYLE_GUIDE.md` + updated linter configs

**Acceptance Criteria:**
- No linter errors on `develop` branch (warnings are acceptable)
- Team agrees on style conventions
- Pre-commit hook runs linters (optional but recommended)

---

### 9. Upstream Alignment Assessment

**Tasks:**
- [ ] Review Navidrome's recent commits (last 3 months)
- [ ] Identify breaking changes that might affect our fork:
  - Database schema changes (new migrations)
  - API endpoint changes (new parameters, deprecated endpoints)
  - Configuration changes (new settings, removed settings)
- [ ] Check Navidrome's roadmap (GitHub issues, discussions)
- [ ] Identify potential collaboration opportunities:
  - Generic features we could upstream (e.g., ListenEventRepository without social logic)
  - Bug fixes we discover during audit
- [ ] Create "fork divergence tracker":
  - Document every change we make to Navidrome code
  - Note whether it's upstreamable or fork-specific
  - Track rebase conflicts (file, line, resolution)

**Deliverable:** `UPSTREAM_ALIGNMENT.md` + `FORK_DIVERGENCE_TRACKER.md`

**Acceptance Criteria:**
- Team understands recent upstream changes
- Potential conflicts are identified
- Upstreaming strategy is defined

---

## Interfaces Introduced/Modified

**None** (Phase 0 is non-invasive, read-only analysis)

---

## Data Models Introduced/Modified

**None** (Phase 0 doesn't change schema)

---

## Operational Needs

### Infrastructure
- GitHub organization for fork
- Docker Hub account for images
- Cloud VM for staging environment (1 vCPU, 2GB RAM sufficient)

### Access/Permissions
- GitHub: team members need write access to fork
- Docker Hub: push access to image repository
- Staging server: SSH access for deployments

### Monitoring
- GitHub Actions logs (built-in)
- Docker build logs (built-in)
- Staging environment logs: `docker logs navidrome-staging`

### Documentation
- All docs in `/docs/` directory of fork
- Use Markdown for text docs
- Use Mermaid for diagrams (renders in GitHub)

---

## Risks & Mitigations

### Risk: Upstream changes break fork during audit period
**Likelihood:** Low  
**Impact:** Medium  
**Mitigation:**
- Pin to specific Navidrome commit during audit (don't track latest)
- Document baseline version: `v0.52.5` (as of Dec 25, 2025)
- After audit, create rebase plan before tracking upstream again

### Risk: Team lacks Go or React expertise
**Likelihood:** Medium  
**Impact:** High  
**Mitigation:**
- Allocate 1 week for language/framework training
- Pair programming during audit (senior + junior)
- External consultant for knowledge transfer (if budget allows)

### Risk: Test suite fails in local environment
**Likelihood:** Medium  
**Impact:** Medium  
**Mitigation:**
- Use Docker for reproducible environment (`make testenv`)
- Document common issues in `TROUBLESHOOTING.md`
- Set up Slack/Discord for team support

### Risk: Fork becomes out of sync with upstream quickly
**Likelihood:** High (Navidrome is actively developed)  
**Impact:** High  
**Mitigation:**
- Automated weekly sync job (GitHub Actions)
- Allocate 4 hours/week for rebase/conflict resolution
- Minimize changes to upstream code (prefer new files over edits)

---

## Definition of Done

### Must Have
- ✅ All team members can build and run Navidrome locally
- ✅ Full test suite passes on fork's `develop` branch
- ✅ CI/CD pipeline is operational (tests, builds, Docker images)
- ✅ Architecture documentation is complete and reviewed
- ✅ Code audit findings are documented (with notes for each critical file)
- ✅ Fork management strategy is documented and implemented
- ✅ Staging environment is deployed and accessible

### Should Have
- ✅ Test coverage report generated (>70% for critical paths)
- ✅ Code style guide documented and linters configured
- ✅ Dependency analysis complete
- ✅ Upstream alignment assessment done (recent changes reviewed)
- ✅ Weekly sync job configured (creates PR automatically)

### Could Have
- Pre-commit hooks for linting (nice to have, not required)
- Performance baseline (useful for future comparison, but not critical now)
- Security audit (important, but can be deferred to Phase 7)

---

## Success Metrics

- **Team Readiness:** All engineers can modify, test, and deploy code independently
- **Build Health:** CI pipeline has 0 failing jobs on `develop` branch
- **Documentation Completeness:** All deliverables from this phase exist and are reviewed
- **Upstream Tracking:** Fork is no more than 1 week behind upstream (sync job catches up weekly)

---

## Next Steps (Transition to Phase 1)

After Phase 0 is complete:
1. Create Phase 1 branch: `feature/stable-backend-baseline`
2. Set up performance monitoring (baseline metrics)
3. Begin work on Navidrome modifications (listen_event table, etc.)

---

**Estimated Effort:**
- Senior Engineer: 60 hours (code audit, architecture docs, CI setup)
- Mid-Level Engineer: 40 hours (testing, documentation, dependency analysis)
- Junior Engineer: 20 hours (development environment setup, style guide)
- **Total: 120 hours (~3 person-weeks with 40-hour weeks)**

---

**End of Phase 0**
