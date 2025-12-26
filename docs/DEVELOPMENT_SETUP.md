# Development Setup Guide

This guide provides comprehensive instructions for setting up the Navidrome development environment.

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Go | 1.25+ | Backend development |
| Node.js | 20+ | Frontend development |
| npm | 9+ | Package management |
| FFmpeg | 4.0+ | Audio transcoding |
| SQLite | 3.x | Database |
| Git | 2.x | Version control |
| TagLib | 2.x | Audio metadata extraction |

### Optional Software

| Software | Purpose |
|----------|---------|
| Docker | Containerized development |
| Make | Build automation |
| golangci-lint | Go linting |

## Quick Start

### 1. Clone the Repository

```bash
# Clone the fork
git clone https://github.com/<your-org>/navidrome.git
cd navidrome

# Add upstream remote for syncing
git remote add upstream https://github.com/navidrome/navidrome.git
```

### 2. Install Dependencies

```bash
# Install all dependencies and setup development environment
make setup
```

This command will:
- Download Go dependencies
- Install Node.js dependencies (in `ui/` directory)
- Install golangci-lint
- Setup Git hooks

### 3. Build the Project

```bash
# Build backend and frontend
make build
```

### 4. Run in Development Mode

```bash
# Start with hot-reload for both backend and frontend
make dev
```

Access the web UI at: http://localhost:4533

## Detailed Setup

### Go Environment

1. Install Go 1.25 or later from https://go.dev/dl/

2. Verify installation:
```bash
go version
# Should output: go version go1.25.x ...
```

3. Download Go dependencies:
```bash
go mod download
```

### Node.js Environment

1. Install Node.js 20+ (use nvm recommended):
```bash
nvm install 20
nvm use 20
```

2. Verify installation:
```bash
node --version
# Should output: v20.x.x
```

3. Install UI dependencies:
```bash
cd ui && npm ci
```

### TagLib (Required for Scanner)

TagLib is required for reading audio metadata. Installation varies by OS:

**Ubuntu/Debian:**
```bash
sudo apt-get install libtag1-dev pkg-config
```

**macOS:**
```bash
brew install taglib pkg-config
```

**Windows:**
Use MinGW or MSYS2 with taglib package.

### FFmpeg (Required for Transcoding)

**Ubuntu/Debian:**
```bash
sudo apt-get install ffmpeg
```

**macOS:**
```bash
brew install ffmpeg
```

**Windows:**
Download from https://ffmpeg.org/download.html and add to PATH.

## Development Commands

### Build Commands

```bash
# Build backend and frontend
make build

# Build only frontend
make buildjs

# Build with debug symbols (for debugging)
make debug-build

# Cross-compile for multiple platforms
make docker-build

# Build Docker image
make docker-image
```

### Test Commands

```bash
# Run Go tests
make test

# Run Go tests with race detector
make test-race

# Run JavaScript tests
make test-js

# Run all tests (Go + JS + i18n)
make testall

# Run Go tests in watch mode (re-run on changes)
make watch

# Run tests for specific package
make test PKG=./core/scrobbler
```

### Development Commands

```bash
# Start development servers (hot-reload)
make dev

# Start only backend in development mode
make server

# Stop development servers
make stop

# Lint Go code
make lint

# Lint Go and JS code
make lintall

# Format code
make format

# Update dependency injection (Wire)
make wire

# Update snapshot tests
make snapshots
```

### Database Commands

```bash
# Create new SQL migration
make migration-sql name=add_listen_event_table

# Create new Go migration
make migration-go name=add_listen_event_table
```

### Plugin Commands

```bash
# Generate Go code from protobuf
make plugin-gen

# Build example plugins
make plugin-examples

# Build test plugins
make plugin-tests
```

### Other Commands

```bash
# Download free test music
make get-music

# Clean build artifacts
make clean

# Show all available commands
make help
```

## IDE Setup

### VS Code (Recommended)

Install these extensions:
- **Go** (golang.go) - Go language support
- **ESLint** - JavaScript linting
- **Prettier** - Code formatting

Recommended settings (`.vscode/settings.json`):
```json
{
  "go.buildTags": "netgo",
  "go.lintTool": "golangci-lint",
  "go.lintFlags": ["--fast"],
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  }
}
```

### GoLand / IntelliJ IDEA

1. Open the project directory
2. Enable Go Modules integration
3. Set build tags: `netgo`
4. Configure golangci-lint as external tool

## Configuration

Navidrome uses a configuration file (`navidrome.toml`) or environment variables.

Create a development config file:
```bash
cat > navidrome.toml << 'EOF'
# Development configuration
MusicFolder = "./music"
DataFolder = "./data"
LogLevel = "debug"
ScanSchedule = "@every 1h"

# Development settings
DevActivityPanel = true
DevAutoCreateAdminPassword = "admin"
EOF
```

### Key Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `MusicFolder` | - | Path to music library |
| `DataFolder` | `.` | Path to data directory |
| `LogLevel` | `info` | Log level (debug, info, warn, error) |
| `Address` | `0.0.0.0` | Server bind address |
| `Port` | `4533` | Server port |
| `ScanSchedule` | `@every 1m` | Cron expression for scanning |

See `conf/configuration.go` for all options.

## Project Structure

```
navidrome/
├── adapters/          # External service adapters (TagLib)
├── cmd/               # CLI commands and application entry
├── conf/              # Configuration management
├── consts/            # Constants and version info
├── core/              # Business logic layer
│   ├── agents/        # External metadata providers
│   ├── artwork/       # Cover art processing
│   ├── ffmpeg/        # FFmpeg transcoding
│   ├── lyrics/        # Lyrics fetching
│   ├── playback/      # Jukebox playback
│   └── scrobbler/     # Play tracking (KEY for telemetry)
├── db/                # Database layer
│   └── migrations/    # SQL migrations
├── log/               # Logging utilities
├── model/             # Domain models and interfaces
├── persistence/       # Repository implementations (SQL)
├── plugins/           # WASM plugin system
├── scanner/           # Library scanner
├── scheduler/         # Cron job scheduler
├── server/            # HTTP server and API
│   ├── events/        # SSE (Server-Sent Events)
│   ├── nativeapi/     # Native REST API
│   ├── public/        # Public share endpoints
│   └── subsonic/      # Subsonic API implementation
├── tests/             # Integration tests
├── ui/                # React frontend
│   ├── src/           # React components
│   └── public/        # Static assets
└── utils/             # Utility functions
```

## Troubleshooting

### Build Errors

**Error: `Package taglib was not found`**
- Install TagLib development package (see TagLib section above)
- Ensure `pkg-config` is installed

**Error: `go.mod requires go >= 1.25`**
- Upgrade Go to version 1.25 or later

**Error: Node modules not found**
- Run `cd ui && npm ci`

### Runtime Errors

**Error: `no such table: media_file`**
- Delete the SQLite database and restart (migrations will run automatically)
- Database location: `<DataFolder>/navidrome.db`

**Error: `ffmpeg not found`**
- Install FFmpeg and ensure it's in PATH

### Test Failures

**Tests fail with `cgo disabled`**
- Set `CGO_ENABLED=1`
- Ensure C compiler is available (gcc/clang)

## Next Steps

After setting up your development environment:

1. Read `docs/architecture/ARCHITECTURE_MAP.md` to understand the codebase
2. Read `docs/TESTING_GUIDE.md` for testing conventions
3. Read `docs/CODE_STYLE_GUIDE.md` for coding standards
4. Check `docs/FORK_MANAGEMENT.md` for branch/merge workflow

## Getting Help

- [Navidrome Discord](https://discord.gg/xh7j7yF)
- [GitHub Issues](https://github.com/navidrome/navidrome/issues)
- [Navidrome Documentation](https://www.navidrome.org/docs/)
