# Code Style Guide

This document defines the coding standards and conventions for the Navidrome fork.

## Go Conventions

### General Principles

1. **Follow Go conventions** - Use standard Go idioms and patterns
2. **Keep it simple** - Prefer clarity over cleverness
3. **Be consistent** - Match existing code style in the file/package

### Naming Conventions

**Packages:**
```go
// Good: lowercase, short, single word
package scrobbler
package persistence

// Bad: underscores, mixed case
package play_tracker
package PlayTracker
```

**Exported vs. Unexported:**
```go
// Exported (public): PascalCase
func ProcessEvent(e Event) error
type PlayTracker struct {}

// Unexported (private): camelCase
func processEvent(e Event) error
type playTracker struct {}
```

**Interfaces:**
```go
// Single-method interfaces: verb + "er"
type Reader interface { Read(p []byte) (n int, err error) }
type Scrobbler interface { Scrobble(ctx context.Context, s Scrobble) error }

// Multi-method interfaces: descriptive noun
type DataStore interface {
    MediaFile(ctx context.Context) MediaFileRepository
    Annotation(ctx context.Context) AnnotationRepository
}
```

**Acronyms:**
```go
// All caps for acronyms
var userID string
var httpClient *http.Client
var xmlParser XMLParser

// Not: userId, httpClient, XmlParser
```

### Error Handling

**Always check errors:**
```go
// Good
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doing something: %w", err)
}

// Bad: ignoring errors
result, _ := doSomething()
```

**Wrap errors with context:**
```go
// Good
if err != nil {
    return fmt.Errorf("failed to save user %s: %w", userID, err)
}

// Bad: losing context
if err != nil {
    return err
}
```

**Sentinel errors:**
```go
// Define package-level errors
var ErrNotFound = errors.New("not found")
var ErrUnauthorized = errors.New("unauthorized")

// Check with errors.Is
if errors.Is(err, ErrNotFound) {
    // handle not found
}
```

### Context

**Always pass context as first parameter:**
```go
// Good
func GetUser(ctx context.Context, id string) (*User, error)

// Bad
func GetUser(id string, ctx context.Context) (*User, error)
func GetUser(id string) (*User, error) // missing context
```

**Extract user from context:**
```go
func (r *repository) Get(ctx context.Context, id string) (*Entity, error) {
    user := request.UserFrom(ctx)
    if user == nil {
        return nil, ErrUnauthorized
    }
    // ... use user.ID for filtering
}
```

### Logging

**Use structured logging:**
```go
import "github.com/navidrome/navidrome/log"

// Good: structured logging with context
log.Info(ctx, "Processing event", "eventType", event.Type, "userId", event.UserID)

// Bad: printf-style
log.Infof("Processing event %s for user %s", event.Type, event.UserID)
```

**Log levels:**
- `Debug`: Detailed debugging information
- `Info`: General operational information
- `Warn`: Warning conditions (recoverable errors)
- `Error`: Error conditions (may need attention)

**Include context:**
```go
// The context carries request ID, user ID, etc.
log.Debug(ctx, "Starting operation", "operation", "scan")
```

### File Organization

**Group imports:**
```go
import (
    // Standard library
    "context"
    "fmt"
    "time"

    // Third-party packages
    "github.com/go-chi/chi/v5"
    
    // Internal packages
    "github.com/navidrome/navidrome/model"
    "github.com/navidrome/navidrome/persistence"
)
```

**Order declarations:**
```go
// 1. Constants
const MaxRetries = 3

// 2. Variables
var defaultTimeout = 30 * time.Second

// 3. Types
type Service struct {
    ds model.DataStore
}

// 4. Constructor
func NewService(ds model.DataStore) *Service {
    return &Service{ds: ds}
}

// 5. Methods (group by functionality)
func (s *Service) Get(ctx context.Context, id string) (*Entity, error) { ... }
func (s *Service) Create(ctx context.Context, e *Entity) error { ... }
func (s *Service) Update(ctx context.Context, e *Entity) error { ... }
func (s *Service) Delete(ctx context.Context, id string) error { ... }

// 6. Private helper functions
func (s *Service) validate(e *Entity) error { ... }
```

### Testing

**Test file naming:**
```
file.go        -> file_test.go
service.go     -> service_test.go
```

**Use Ginkgo/Gomega:**
```go
var _ = Describe("Service", func() {
    Describe("Get", func() {
        Context("when entity exists", func() {
            It("should return the entity", func() {
                Expect(result).ToNot(BeNil())
            })
        })
        
        Context("when entity does not exist", func() {
            It("should return ErrNotFound", func() {
                Expect(err).To(MatchError(ErrNotFound))
            })
        })
    })
})
```

### Comments

**Package comments:**
```go
// Package scrobbler implements play tracking and external scrobbling.
// It handles Now Playing updates, scrobble submissions, and dispatches
// to external services like Last.fm and ListenBrainz.
package scrobbler
```

**Exported function comments:**
```go
// NowPlaying updates the now playing state for a player.
// It stores the current track in memory and dispatches to external scrobblers.
// The ctx must contain a valid user and player.
func (t *PlayTracker) NowPlaying(ctx context.Context, trackID string) error {
```

**Don't state the obvious:**
```go
// Bad: obvious comment
// GetUser gets a user
func GetUser(id string) *User

// Good: adds useful information
// GetUser retrieves a user by ID. Returns ErrNotFound if the user doesn't exist.
func GetUser(id string) (*User, error)
```

## React/JavaScript Conventions

### Component Structure

**Functional components with hooks:**
```jsx
import React, { useState, useEffect } from 'react';
import { makeStyles } from '@material-ui/core/styles';

const useStyles = makeStyles((theme) => ({
    root: {
        padding: theme.spacing(2),
    },
}));

const MyComponent = ({ prop1, prop2 }) => {
    const classes = useStyles();
    const [state, setState] = useState(initialValue);

    useEffect(() => {
        // effect logic
    }, [dependency]);

    const handleClick = () => {
        // handler logic
    };

    return (
        <div className={classes.root}>
            {/* JSX */}
        </div>
    );
};

export default MyComponent;
```

### State Management

**Local state:**
```jsx
const [value, setValue] = useState('');
```

**Global state (Redux):**
```jsx
import { useSelector, useDispatch } from 'react-redux';
import { updateSetting } from './actions';

const Component = () => {
    const setting = useSelector(state => state.settings.value);
    const dispatch = useDispatch();

    const handleChange = (newValue) => {
        dispatch(updateSetting(newValue));
    };
};
```

### Styling

**Use makeStyles:**
```jsx
const useStyles = makeStyles((theme) => ({
    container: {
        display: 'flex',
        flexDirection: 'column',
        gap: theme.spacing(2),
    },
    button: {
        marginTop: theme.spacing(1),
    },
}));
```

### File Organization

```
src/
├── components/        # Reusable UI components
│   ├── Button/
│   │   ├── Button.js
│   │   ├── Button.test.js
│   │   └── index.js
├── pages/            # Page components
├── hooks/            # Custom hooks
├── reducers/         # Redux reducers
├── actions/          # Redux actions
├── utils/            # Utility functions
└── config/           # Configuration
```

## Linting

### Go Linting

**golangci-lint configuration (`.golangci.yml`):**
```yaml
run:
  timeout: 5m
  build-tags:
    - netgo

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused

linters-settings:
  errcheck:
    check-blank: true
```

**Run linting:**
```bash
make lint
```

### JavaScript Linting

**ESLint configuration:**
```json
{
    "extends": ["react-app", "react-app/jest"],
    "rules": {
        "no-unused-vars": "warn",
        "react-hooks/rules-of-hooks": "error",
        "react-hooks/exhaustive-deps": "warn"
    }
}
```

**Run linting:**
```bash
cd ui && npm run lint
```

### Formatting

**Go formatting:**
```bash
# Auto-format Go code
make format
# or
goimports -w .
```

**JavaScript formatting (Prettier):**
```bash
cd ui && npm run prettier
```

## Git Conventions

### Commit Messages

**Format:**
```
<type>: <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Formatting (no code change)
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance

**Examples:**
```
feat: add listen event ingestion endpoint

Add POST /api/events/listen endpoint for bulk event ingestion.
Events are validated and stored in the listen_event table.

Closes #123
```

```
fix: prevent duplicate scrobbles within 30 seconds

Add deduplication check to prevent the same track from being
scrobbled multiple times within a 30-second window.
```

### Branch Names

```
feature/telemetry-v1
feature/social-graph
fix/duplicate-scrobbles
docs/update-setup-guide
```

## Code Review Checklist

- [ ] Code follows style guide
- [ ] Tests are included and pass
- [ ] Documentation is updated
- [ ] No linter errors
- [ ] Error handling is complete
- [ ] Context is properly propagated
- [ ] Logging is appropriate
- [ ] No security vulnerabilities

## References

- [Effective Go](https://golang.org/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [React Documentation](https://reactjs.org/docs/getting-started.html)
- [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)
