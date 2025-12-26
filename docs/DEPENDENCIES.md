# Dependency Analysis

This document analyzes the current dependencies of Navidrome and documents planned additions for the fork's social and analytics features.

## Current Dependencies

### Go Dependencies (go.mod)

**Core Dependencies:**

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/Masterminds/squirrel` | 1.5.4 | SQL query builder |
| `github.com/go-chi/chi/v5` | 5.2.3 | HTTP router |
| `github.com/go-chi/jwtauth/v5` | 5.3.3 | JWT authentication |
| `github.com/mattn/go-sqlite3` | 1.14.32 | SQLite driver (CGO) |
| `github.com/pocketbase/dbx` | 1.11.0 | Database abstraction |
| `github.com/pressly/goose/v3` | 3.26.0 | Database migrations |

**External Services:**

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/prometheus/client_golang` | 1.23.2 | Prometheus metrics |
| `github.com/gorilla/websocket` | 1.5.3 | WebSocket support |
| `github.com/robfig/cron/v3` | 3.0.1 | Cron scheduling |

**Utilities:**

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/google/uuid` | 1.6.0 | UUID generation |
| `github.com/google/wire` | 0.7.0 | Dependency injection |
| `github.com/sirupsen/logrus` | 1.9.3 | Structured logging |
| `github.com/spf13/viper` | 1.21.0 | Configuration |
| `github.com/spf13/cobra` | 1.10.2 | CLI framework |

**Media Processing:**

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/dhowden/tag` | Custom fork | Audio metadata (ID3, etc.) |
| `github.com/disintegration/imaging` | 1.6.2 | Image processing |

**Testing:**

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/onsi/ginkgo/v2` | 2.27.3 | BDD testing framework |
| `github.com/onsi/gomega` | 1.38.3 | Test matchers |
| `github.com/stretchr/testify` | 1.11.1 | Additional test utilities |
| `github.com/bradleyjkemp/cupaloy/v2` | 2.8.0 | Snapshot testing |

**Plugins:**

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/tetratelabs/wazero` | 1.11.0 | WASM runtime |
| `github.com/knqyf263/go-plugin` | 0.9.0 | Plugin framework |
| `google.golang.org/protobuf` | 1.36.11 | Protocol buffers |

### Node.js Dependencies (ui/package.json)

**Core:**

| Package | Version | Purpose |
|---------|---------|---------|
| `react` | 17.x | UI framework |
| `react-admin` | 3.19.x | Admin UI framework |
| `@material-ui/core` | 4.x | UI components |
| `redux` | 4.x | State management |

**Build:**

| Package | Version | Purpose |
|---------|---------|---------|
| `vite` | 4.x | Build tool |
| `typescript` | 4.x | Type checking |

## Planned Additions (Fork-Specific)

### For Social Service (External)

| Package | Version | Purpose | Risk |
|---------|---------|---------|------|
| `github.com/99designs/gqlgen` | 0.17.x | GraphQL server | Low |
| `github.com/jackc/pgx/v5` | 5.x | PostgreSQL driver | Low |
| `github.com/go-redis/redis/v9` | 9.x | Redis client | Low |

### For Analytics Service (External)

| Package | Version | Purpose | Risk |
|---------|---------|---------|------|
| `github.com/jackc/pgx/v5` | 5.x | TimescaleDB driver | Low |
| `github.com/99designs/gqlgen` | 0.17.x | GraphQL server | Low |

### For Navidrome Core (Minimal Additions)

| Package | Version | Purpose | Risk |
|---------|---------|---------|------|
| `github.com/nats-io/nats.go` | 1.x | Message queue | Low - optional |

**Note:** Fork-specific dependencies are optional and configured via feature flags.

## Dependency Update Policy

### Upstream Dependencies

- **Follow upstream** for all shared dependencies
- When upstream updates a dependency, accept the update
- Don't update shared dependencies independently (causes merge conflicts)

### Fork-Specific Dependencies

- **Pin to stable versions** (not `latest`)
- Update quarterly or for security fixes
- Test thoroughly before updating

### Security Updates

- Apply security patches immediately
- Use Dependabot for automated vulnerability alerts
- Review security advisories weekly

## Compatibility Matrix

| Component | Go Version | Node Version | SQLite | PostgreSQL |
|-----------|------------|--------------|--------|------------|
| Navidrome Core | 1.25+ | 20+ | 3.x | N/A |
| Social Service | 1.25+ | N/A | N/A | 15+ |
| Analytics Service | 1.25+ | N/A | N/A | 15+ (TimescaleDB) |
| Recommendation Service | N/A | N/A | N/A | 15+ |

## Potential Conflicts

### High Risk

None identified - fork adds new dependencies rather than changing existing ones.

### Medium Risk

1. **gqlgen version** - If upstream adds GraphQL, ensure compatible versions
2. **PostgreSQL driver** - Multiple drivers exist; standardize on `pgx`

### Low Risk

1. **NATS** - Optional dependency, can be disabled
2. **Redis** - Standard client, widely compatible

## Dependency Auditing

### Automated Checks

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "npm"
    directory: "/ui"
    schedule:
      interval: "weekly"
```

### Manual Audits

Run quarterly:
```bash
# Check for Go vulnerabilities
go list -json -m all | nancy sleuth

# Check for npm vulnerabilities
cd ui && npm audit
```

## License Compliance

All dependencies are compatible with Navidrome's GPL-3.0 license.

| License | Compatible | Packages |
|---------|------------|----------|
| MIT | ✅ Yes | Most dependencies |
| Apache 2.0 | ✅ Yes | Google packages |
| BSD | ✅ Yes | Various utilities |
| GPL-3.0 | ✅ Yes | Same license |
| LGPL | ✅ Yes | TagLib bindings |

## Build Requirements

### CGO Dependencies

Navidrome requires CGO for:
- SQLite (`github.com/mattn/go-sqlite3`)
- TagLib (audio metadata)

**Build requirements:**
- C compiler (gcc/clang)
- TagLib development headers
- SQLite development headers

### Cross-Compilation

Cross-compilation is handled via Docker:
```bash
make docker-build PLATFORMS=linux/amd64,linux/arm64
```

## References

- [Go Modules Reference](https://go.dev/ref/mod)
- [npm Documentation](https://docs.npmjs.com/)
- [Navidrome Dependencies](https://github.com/navidrome/navidrome/blob/master/go.mod)
