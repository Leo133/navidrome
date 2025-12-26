# Testing Guide

This guide documents the testing conventions, patterns, and best practices for the Navidrome codebase.

## Testing Framework

### Go Tests

Navidrome uses [Ginkgo](https://github.com/onsi/ginkgo) with [Gomega](https://github.com/onsi/gomega) for BDD-style testing.

**Key Features:**
- BDD syntax (`Describe`, `Context`, `It`, `BeforeEach`)
- Rich matchers (`Expect`, `Eventually`, `Consistently`)
- Parallel test execution
- Focused tests (`FIt`, `FDescribe`) for debugging
- Pending tests (`PIt`, `PDescribe`) for WIP

### JavaScript Tests

The React UI uses [Jest](https://jestjs.io/) with [React Testing Library](https://testing-library.com/).

## Running Tests

### Go Tests

```bash
# Run all tests
make test

# Run tests with race detector
make test-race

# Run tests for specific package
make test PKG=./core/scrobbler

# Run tests in watch mode (re-run on changes)
make watch

# Run specific test by name (regex)
go tool ginkgo -tags netgo -v --focus "should track play count" ./...

# Run tests with verbose output
go test -tags netgo -v ./core/scrobbler/...

# Run tests and generate coverage report
go test -tags netgo -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### JavaScript Tests

```bash
# Run all JS tests
make test-js

# Run in watch mode
cd ui && npm test -- --watch

# Run specific test file
cd ui && npm test -- --testPathPattern="Player.test"

# Generate coverage report
cd ui && npm test -- --coverage
```

## Test Structure

### Go Test File Organization

Tests are co-located with the code they test:
```
core/
├── scrobbler/
│   ├── play_tracker.go           # Implementation
│   ├── play_tracker_test.go      # Tests
│   ├── scrobbler_suite_test.go   # Test suite setup
│   └── interfaces.go             # Interfaces
```

### Ginkgo Test Suite Setup

Each package has a suite file (`*_suite_test.go`):

```go
package scrobbler_test

import (
    "testing"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
)

func TestScrobbler(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Scrobbler Suite")
}
```

### Writing Tests (Ginkgo Style)

```go
package scrobbler_test

import (
    "context"
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"

    "github.com/navidrome/navidrome/core/scrobbler"
    "github.com/navidrome/navidrome/model"
)

var _ = Describe("PlayTracker", func() {
    var (
        ctx        context.Context
        ds         *mockDataStore
        playTracker scrobbler.PlayTracker
    )

    BeforeEach(func() {
        ctx = context.Background()
        ds = newMockDataStore()
        playTracker = scrobbler.NewPlayTracker(ds)
    })

    Describe("NowPlaying", func() {
        Context("when a valid track is submitted", func() {
            It("should update the now playing state", func() {
                err := playTracker.NowPlaying(ctx, "player1", "Web UI", "track123")
                Expect(err).ToNot(HaveOccurred())

                nowPlaying := playTracker.GetNowPlaying(ctx)
                Expect(nowPlaying).To(HaveLen(1))
                Expect(nowPlaying[0].MediaFileID).To(Equal("track123"))
            })
        })

        Context("when the player has scrobbling disabled", func() {
            BeforeEach(func() {
                // Setup player with scrobbling disabled
            })

            It("should not record the play", func() {
                // Test assertion
            })
        })
    })

    Describe("Submit", func() {
        It("should increment play count", func() {
            submission := model.Scrobble{
                MediaFileID: "track123",
                PlayTime:    time.Now(),
            }

            err := playTracker.Submit(ctx, []model.Scrobble{submission})
            Expect(err).ToNot(HaveOccurred())

            // Verify play count was incremented
            Expect(ds.IncPlayCountCalled).To(BeTrue())
        })
    })
})
```

## Mocking Patterns

### Mock DataStore

Navidrome uses dependency injection (Wire). Tests mock the `DataStore` interface:

```go
// tests/mock_data_store.go
type MockDataStore struct {
    mock.Mock
    
    // Embedded repositories
    *MockMediaFileRepository
    *MockAnnotationRepository
    // ... other repositories
}

func (m *MockDataStore) MediaFile(ctx context.Context) model.MediaFileRepository {
    return m.MockMediaFileRepository
}

func (m *MockDataStore) Annotation(ctx context.Context) model.AnnotationRepository {
    return m.MockAnnotationRepository
}
```

### Using Mocks in Tests

```go
var _ = Describe("SomeService", func() {
    var (
        ds   *MockDataStore
        svc  SomeService
    )

    BeforeEach(func() {
        ds = &MockDataStore{}
        ds.MockMediaFileRepository = &MockMediaFileRepository{}
        svc = NewSomeService(ds)
    })

    It("should call the repository", func() {
        // Setup expectations
        ds.MockMediaFileRepository.On("Get", mock.Anything, "track123").
            Return(&model.MediaFile{ID: "track123", Title: "Test"}, nil)

        // Execute
        result, err := svc.DoSomething(ctx, "track123")

        // Assert
        Expect(err).ToNot(HaveOccurred())
        Expect(result.Title).To(Equal("Test"))
        ds.MockMediaFileRepository.AssertExpectations(GinkgoT())
    })
})
```

## Snapshot Testing

Navidrome uses [cupaloy](https://github.com/bradleyjkemp/cupaloy) for snapshot testing of API responses:

```go
import "github.com/bradleyjkemp/cupaloy/v2"

var _ = Describe("API Response", func() {
    It("should match snapshot", func() {
        response := generateResponse()
        cupaloy.SnapshotT(GinkgoT(), response)
    })
})
```

Update snapshots:
```bash
make snapshots
# or
UPDATE_SNAPSHOTS=true go tool ginkgo ./server/subsonic/responses/...
```

## Integration Tests

Integration tests use real SQLite databases:

```go
var _ = Describe("Integration", func() {
    var (
        db   *sql.DB
        ds   model.DataStore
    )

    BeforeEach(func() {
        // Create in-memory database
        var err error
        db, err = sql.Open("sqlite3", ":memory:")
        Expect(err).ToNot(HaveOccurred())

        // Run migrations
        goose.SetBaseFS(embed.FS{})
        err = goose.Up(db, "migrations")
        Expect(err).ToNot(HaveOccurred())

        ds = persistence.New(db)
    })

    AfterEach(func() {
        db.Close()
    })

    It("should persist data correctly", func() {
        // Test with real database
    })
})
```

## Test Data

### Using Test Fixtures

Test fixtures are stored in `testdata/` directories:

```
scanner/
├── testdata/
│   ├── audio/
│   │   ├── test.mp3
│   │   └── test.flac
│   └── playlists/
│       └── test.m3u
```

Access fixtures in tests:
```go
func TestScanner(t *testing.T) {
    testFile := filepath.Join("testdata", "audio", "test.mp3")
    // Use testFile...
}
```

### Creating Test Data

For database tests, use helper functions:

```go
func createTestMediaFile(ds model.DataStore, id, title string) *model.MediaFile {
    mf := &model.MediaFile{
        ID:       id,
        Title:    title,
        Duration: 180,
        // ... other fields
    }
    err := ds.MediaFile(context.Background()).Put(mf)
    Expect(err).ToNot(HaveOccurred())
    return mf
}
```

## Test Coverage

### Generating Coverage Reports

```bash
# Generate coverage for all packages
go test -tags netgo -coverprofile=coverage.out ./...

# View HTML report
go tool cover -html=coverage.out -o coverage.html

# View coverage by function
go tool cover -func=coverage.out
```

### Critical Paths (Target >70% Coverage)

1. **Authentication** (`server/auth/`)
   - JWT generation and validation
   - Login/logout flows

2. **Play Tracking** (`core/scrobbler/`)
   - NowPlaying updates
   - Scrobble submission
   - External scrobbler dispatch

3. **Persistence** (`persistence/`)
   - Annotation operations (play count, rating, stars)
   - CRUD operations for all entities

4. **API Handlers** (`server/subsonic/`, `server/nativeapi/`)
   - All public endpoints
   - Error handling

## Best Practices

### Do's

✅ **Use descriptive test names**
```go
It("should increment play count when scrobbling a valid track", func() {
```

✅ **Test edge cases**
```go
Context("when the track ID is empty", func() {
    It("should return an error", func() {
```

✅ **Use table-driven tests for many inputs**
```go
entries := []struct {
    input    string
    expected int
}{
    {"100", 100},
    {"", 0},
    {"invalid", 0},
}
for _, e := range entries {
    It(fmt.Sprintf("should parse %q as %d", e.input, e.expected), func() {
        Expect(Parse(e.input)).To(Equal(e.expected))
    })
}
```

✅ **Clean up after tests**
```go
AfterEach(func() {
    // Clean up resources
})
```

### Don'ts

❌ **Don't test implementation details**
- Test behavior, not internal state

❌ **Don't use `FIt` or `FDescribe` in commits**
- These focus only on specific tests

❌ **Don't mock what you don't own**
- Use integration tests for external libraries

❌ **Don't skip tests without reason**
```go
// Bad: unexplained skip
Skip("TODO")

// Better: explain why
Skip("Requires external service not available in CI")
```

## Continuous Integration

Tests run automatically on every push and PR via GitHub Actions.

### CI Configuration (`.github/workflows/ci.yml`)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.25'
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y libtag1-dev ffmpeg
      - name: Run tests
        run: make test
```

## Writing Tests for New Features

When adding new features, follow this checklist:

- [ ] Write unit tests for all public functions
- [ ] Test happy path and error cases
- [ ] Add integration tests for database operations
- [ ] Add snapshot tests for API responses (if applicable)
- [ ] Ensure tests pass locally: `make test`
- [ ] Ensure tests pass with race detector: `make test-race`
- [ ] Check coverage: `go test -coverprofile=coverage.out ./your/package`

## References

- [Ginkgo Documentation](https://onsi.github.io/ginkgo/)
- [Gomega Documentation](https://onsi.github.io/gomega/)
- [Testing in Go](https://golang.org/doc/tutorial/add-a-test)
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)
