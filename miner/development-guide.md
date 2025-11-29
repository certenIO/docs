# Certen Miner Development Guide

## Project Structure and Code Organization

### Repository Layout

```
certen-miner/
├── cmd/                    # Command-line applications
│   ├── miner/             # Main CLI application
│   │   └── main.go        # Entry point
│   └── integrated-miner/  # Standalone binary for containers
│       ├── main.go        # Embedded configuration runner
│       └── Dockerfile     # Container build configuration
│
├── pkg/                   # Library packages (importable)
│   ├── cli/              # Cobra CLI command implementations
│   │   ├── root.go       # Base command and global flags
│   │   ├── init.go       # Initialize command
│   │   ├── run.go        # Mining operation command
│   │   ├── status.go     # Status reporting command
│   │   └── validate.go   # Environment validation command
│   │
│   ├── config/           # Configuration management
│   │   └── config.go     # YAML configuration and validation
│   │
│   ├── identity/         # Cryptographic identity management
│   │   └── identity.go   # Ed25519 key operations
│   │
│   ├── lxr/              # LXR proof-of-work implementation
│   │   ├── lxrhash.go    # Core algorithm (from Accumulate)
│   │   └── lxr_runner.go # Mining wrapper and difficulty
│   │
│   ├── mining/           # Protocol buffer definitions
│   │   ├── mining.proto  # Message schemas
│   │   └── mining.pb.go  # Generated Go code
│   │
│   ├── node/             # Core P2P mining node
│   │   ├── node.go       # LibP2P integration and main loop
│   │   └── types.go      # Interfaces and data structures
│   │
│   └── verifier/         # Validator proof verification
│       └── proof_verifier.go # HTTP API client and caching
│
├── build/                # Build artifacts (gitignored)
├── docs/                 # Development documentation
├── scripts/              # Build and deployment scripts
├── .github/              # GitHub Actions and templates
├── Makefile              # Cross-platform build system
├── go.mod                # Go module definition
├── go.sum                # Dependency checksums
└── README.md             # Project overview and quick start
```

### Package Dependencies

**Dependency Graph**:
```
cmd/miner
  └── pkg/cli
      ├── pkg/config
      │   └── (external: viper, yaml.v3)
      ├── pkg/identity
      │   └── (external: libp2p/crypto)
      ├── pkg/lxr
      │   └── pkg/mining (protobuf)
      ├── pkg/node
      │   ├── pkg/lxr
      │   ├── pkg/verifier
      │   └── (external: libp2p/*)
      └── pkg/verifier
          └── pkg/node (types only)
```

**Design Rules**:
- **No Circular Dependencies**: Enforced through interface design
- **Minimal External Dependencies**: Conservative dependency management
- **Interface Segregation**: Small, focused interfaces for testability
- **Dependency Injection**: Interfaces enable mocking and testing

## Development Workflow

### Git Workflow

**Branch Strategy**:
- **main**: Production-ready code, protected branch
- **develop**: Integration branch for features
- **feature/*****: Feature development branches
- **hotfix/*****: Critical bug fixes
- **release/*****: Release preparation branches

**Commit Convention**:
```
type(scope): description

[optional body]

[optional footer]
```

**Types**:
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code formatting (no logic changes)
- **refactor**: Code restructuring (no behavior changes)
- **test**: Adding or updating tests
- **chore**: Build system, CI, or tooling changes

**Examples**:
```bash
git commit -m "feat(lxr): implement parallel mining support"
git commit -m "fix(node): resolve peer discovery timeout issues"
git commit -m "docs(api): add comprehensive networking documentation"
```

### Development Environment

**Required Tools**:
```bash
# Go toolchain
go version  # 1.23+

# Code formatting and linting
go install golang.org/x/tools/cmd/gofumpt@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Protocol buffer compiler (optional)
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# Testing tools
go install github.com/rakyll/gotest@latest
go install github.com/axw/gocov/gocov@latest
```

**Pre-commit Hooks** (`.git/hooks/pre-commit`):
```bash
#!/bin/sh

set -e

echo "Running pre-commit checks..."

# Format code
echo "Formatting code..."
gofumpt -l -w .

# Run linter
echo "Running linter..."
golangci-lint run

# Run tests
echo "Running tests..."
go test -race -short ./...

# Check for direct dependency on internal packages
echo "Checking package dependencies..."
go list -deps ./cmd/miner | grep -E "pkg/(internal|private)" && {
    echo "ERROR: cmd packages should not import internal packages"
    exit 1
}

echo "All checks passed!"
```

### Code Style Guidelines

**Go Style Conventions**:
```go
// Package naming: lowercase, no underscores
package config

// Interface naming: noun or adjective ending in -er
type ProofVerifier interface {
    FetchProofBundle(height uint64) (*ValidatorProofBundle, error)
}

// Struct naming: CamelCase
type MinerNode struct {
    config   Config
    logger   *log.Logger
}

// Function naming: MixedCaps
func NewMinerNode(config Config) *MinerNode {
    return &MinerNode{
        config: config,
        logger: log.New(os.Stdout, "[MinerNode] ", log.LstdFlags),
    }
}

// Constants: MixedCaps or ALL_CAPS for exported constants
const DefaultTimeout = 30 * time.Second
const MAX_CONNECTIONS = 1000

// Error handling: explicit error checking
result, err := doSomething()
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}
```

**Documentation Standards**:
```go
// Package documentation: above package declaration
// Package config provides configuration management for the Certen miner,
// including YAML serialization, validation, and default value handling.
package config

// Function documentation: describes what, not how
// LoadConfig reads and validates a configuration file from the specified path.
// Returns an error if the file cannot be read or contains invalid settings.
func LoadConfig(path string) (*Config, error) {
    // Implementation...
}

// Complex struct documentation
// Config contains all configurable parameters for a Certen miner instance.
// All fields support environment variable override using the CERTEN_ prefix.
type Config struct {
    // MinerID uniquely identifies this miner instance in the network.
    // Generated automatically if not specified.
    MinerID string `yaml:"miner_id"`

    // ValidatorURL specifies the API endpoint for fetching proof bundles.
    // Must be a valid HTTP or HTTPS URL.
    ValidatorURL string `yaml:"validator_url"`
}
```

**Error Handling Patterns**:
```go
// Wrap errors with context
if err != nil {
    return fmt.Errorf("failed to connect to peer %s: %w", peerID, err)
}

// Custom error types for specific handling
type ValidationError struct {
    Field string
    Value interface{}
    Reason string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s=%v: %s", e.Field, e.Value, e.Reason)
}

// Sentinel errors for control flow
var ErrPeerNotFound = errors.New("peer not found")

if errors.Is(err, ErrPeerNotFound) {
    // Handle specific error condition
}
```

## Testing Strategy

### Test Organization

**Test File Structure**:
```
pkg/lxr/
├── lxr_runner.go
├── lxr_runner_test.go      # Unit tests
├── lxrhash.go
├── lxrhash_test.go         # Algorithm verification tests
├── lxrhash_bench_test.go   # Performance benchmarks
└── integration_test.go     # Integration tests
```

**Test Categories**:

1. **Unit Tests**: Test individual functions and methods
2. **Integration Tests**: Test component interactions
3. **Benchmark Tests**: Performance and resource usage tests
4. **Fuzz Tests**: Input validation and edge case discovery

### Unit Testing

**Test Structure**:
```go
func TestLXRRunner_VerifyWithProof(t *testing.T) {
    tests := []struct {
        name           string
        bundle         *node.ValidatorProofBundle
        expectedValid  bool
        expectedScore  float64
        expectedError  string
    }{
        {
            name: "valid proof bundle",
            bundle: &node.ValidatorProofBundle{
                ValidatorID: "test-validator",
                BlockHeight: 1000,
                AnchorRoot:  "abc123",
                AnchorProof: []byte("proof"),
                TxProofs:    []node.TxProof{{TxID: "tx1", Proof: []byte("p1")}},
                Timestamp:   time.Now().Unix(),
            },
            expectedValid: true,
            expectedScore: 1.0,
        },
        {
            name:          "nil bundle",
            bundle:        nil,
            expectedValid: false,
            expectedScore: 0.0,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            config := &lxr.Config{
                TableBits: 20, // Small table for tests
                Loops:     1,
                Passes:    1,
            }

            runner := lxr.NewLXRRunner(config, log.New(io.Discard, "", 0))

            result, err := runner.VerifyWithProof(tt.bundle)

            if tt.expectedError != "" {
                require.Error(t, err)
                assert.Contains(t, err.Error(), tt.expectedError)
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.expectedValid, result.Valid)
            assert.InDelta(t, tt.expectedScore, result.Score, 0.01)
        })
    }
}
```

**Mock Generation**:
```go
//go:generate mockgen -source=types.go -destination=mocks/mock_types.go

// Use generated mocks in tests
func TestMinerNode_PerformAudit(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockLXR := mocks.NewMockLXRRunner(ctrl)
    mockVerifier := mocks.NewMockProofVerifier(ctrl)

    // Set up expectations
    mockVerifier.EXPECT().
        LatestHeight().
        Return(uint64(1000)).
        Times(1)

    mockVerifier.EXPECT().
        FetchProofBundle(uint64(1000)).
        Return(testBundle, nil).
        Times(1)

    mockLXR.EXPECT().
        VerifyWithProof(testBundle).
        Return(expectedResult, nil).
        Times(1)

    // Test the actual functionality
    node := &MinerNode{
        lxr:      mockLXR,
        verifier: mockVerifier,
    }

    err := node.performAuditOnce()
    require.NoError(t, err)
}
```

### Integration Testing

**Network Integration Tests**:
```go
//go:build integration

func TestMinerNode_P2PNetworking(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test in short mode")
    }

    // Create test network with multiple nodes
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    nodes := make([]*node.MinerNode, 3)
    for i := range nodes {
        config := generateTestConfig(t, i)
        nodes[i] = createTestNode(t, config)

        defer nodes[i].Close()
    }

    // Wait for peer discovery
    time.Sleep(5 * time.Second)

    // Verify nodes are connected
    for i, n := range nodes {
        peers := n.GetConnectedPeers()
        assert.GreaterOrEqual(t, len(peers), 1,
            "Node %d should have at least 1 peer", i)
    }

    // Test message propagation
    testMessage := generateTestAuditAttestation(t)
    err := nodes[0].PublishAuditAttestation(testMessage)
    require.NoError(t, err)

    // Verify message received by other nodes
    time.Sleep(2 * time.Second)

    for i := 1; i < len(nodes); i++ {
        received := nodes[i].GetReceivedMessages()
        assert.Greater(t, len(received), 0,
            "Node %d should have received messages", i)
    }
}
```

**Validator API Integration**:
```go
func TestProofVerifier_Integration(t *testing.T) {
    if os.Getenv("VALIDATOR_URL") == "" {
        t.Skip("VALIDATOR_URL not set, skipping integration test")
    }

    config := &verifier.Config{
        ValidatorURL: os.Getenv("VALIDATOR_URL"),
        Timeout:      10 * time.Second,
    }

    verifier := verifier.NewProofVerifier(config,
        log.New(os.Stdout, "[Test] ", log.LstdFlags))

    // Test latest height fetch
    height := verifier.LatestHeight()
    assert.Greater(t, height, uint64(0), "Should fetch valid height")

    // Test proof bundle fetch
    bundle, err := verifier.FetchProofBundle(height)
    require.NoError(t, err)
    assert.Equal(t, height, bundle.BlockHeight)
    assert.NotEmpty(t, bundle.ValidatorID)
}
```

### Performance Testing

**Benchmark Tests**:
```go
func BenchmarkLXRHash(b *testing.B) {
    lxr := lxr.NewLxrPow(5, 20, 6) // 1MB table
    challenge := make([]byte, 32)
    rand.Read(challenge)

    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        lxr.LxrPoWHash(challenge, uint64(i))
    }
}

func BenchmarkLXRHashParallel(b *testing.B) {
    lxr := lxr.NewLxrPow(5, 20, 6)
    challenge := make([]byte, 32)
    rand.Read(challenge)

    b.ResetTimer()
    b.ReportAllocs()

    b.RunParallel(func(pb *testing.PB) {
        nonce := uint64(0)
        for pb.Next() {
            lxr.LxrPoWHash(challenge, atomic.AddUint64(&nonce, 1))
        }
    })
}

// Memory usage benchmarks
func BenchmarkLXRMemory(b *testing.B) {
    tableSizes := []uint64{20, 22, 24, 26} // 1MB, 4MB, 16MB, 64MB

    for _, size := range tableSizes {
        b.Run(fmt.Sprintf("TableSize%dMB", 1<<(size-20)), func(b *testing.B) {
            lxr := lxr.NewLxrPow(5, size, 6)
            challenge := make([]byte, 32)
            rand.Read(challenge)

            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                lxr.LxrPoWHash(challenge, uint64(i))
            }

            // Report memory statistics
            var m runtime.MemStats
            runtime.GC()
            runtime.ReadMemStats(&m)
            b.ReportMetric(float64(m.Alloc), "bytes/alloc")
            b.ReportMetric(float64(m.Sys), "bytes/sys")
        })
    }
}
```

**Load Testing**:
```go
func TestMinerNode_LoadTest(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping load test in short mode")
    }

    node := createTestNode(t, generateTestConfig(t, 0))
    defer node.Close()

    // Simulate high load conditions
    numGoroutines := 100
    numOperations := 1000

    var wg sync.WaitGroup
    errors := make(chan error, numGoroutines)

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            for j := 0; j < numOperations; j++ {
                if err := node.PerformAuditOnce(); err != nil {
                    errors <- err
                    return
                }
            }
        }()
    }

    wg.Wait()
    close(errors)

    // Check for errors
    errorCount := 0
    for err := range errors {
        t.Logf("Load test error: %v", err)
        errorCount++
    }

    // Allow some errors under high load, but not too many
    maxAllowedErrors := numGoroutines / 10 // 10%
    assert.LessOrEqual(t, errorCount, maxAllowedErrors,
        "Too many errors under load: %d/%d", errorCount, numGoroutines)
}
```

### Fuzz Testing

**Input Validation Fuzzing**:
```go
func FuzzValidateProofBundle(f *testing.F) {
    // Seed with valid inputs
    f.Add("validator1", uint64(1000), "anchor123", []byte("proof"))

    f.Fuzz(func(t *testing.T, validatorID string, height uint64,
        anchorRoot string, anchorProof []byte) {

        bundle := &node.ValidatorProofBundle{
            ValidatorID: validatorID,
            BlockHeight: height,
            AnchorRoot:  anchorRoot,
            AnchorProof: anchorProof,
            Timestamp:   time.Now().Unix(),
        }

        // Validation should not panic
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("Validation panicked: %v", r)
            }
        }()

        runner := lxr.NewLXRRunner(&lxr.Config{
            TableBits: 20,
            Loops:     1,
            Passes:    1,
        }, log.New(io.Discard, "", 0))

        _, err := runner.VerifyWithProof(bundle)
        // Error is acceptable, panic is not
        _ = err
    })
}
```

## Build System and CI/CD

### Makefile Structure

**Build Targets**:
```makefile
# Variables
BINARY_NAME := certen-miner
VERSION := $(shell git describe --tags --always --dirty)
BUILD_DIR := build
LDFLAGS := -ldflags "-X main.version=$(VERSION) -X main.buildTime=$(shell date -u +%Y%m%dT%H%M%S)"
GOFLAGS := -trimpath

# Default target
.PHONY: all
all: test build

# Development targets
.PHONY: build
build:
	@echo "Building $(BINARY_NAME)..."
	@mkdir -p $(BUILD_DIR)
	go build $(GOFLAGS) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME) ./cmd/miner

.PHONY: test
test:
	@echo "Running tests..."
	go test -race -v ./...

.PHONY: test-integration
test-integration:
	@echo "Running integration tests..."
	go test -race -v -tags=integration ./...

.PHONY: bench
bench:
	@echo "Running benchmarks..."
	go test -bench=. -benchmem -run=^$ ./...

.PHONY: coverage
coverage:
	@echo "Generating coverage report..."
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

# Code quality
.PHONY: lint
lint:
	@echo "Running linter..."
	golangci-lint run

.PHONY: fmt
fmt:
	@echo "Formatting code..."
	gofumpt -l -w .

# Cross-compilation
.PHONY: cross-compile
cross-compile:
	@echo "Cross-compiling for all platforms..."
	@for platform in $(PLATFORMS); do \
		os=$$(echo $$platform | cut -d'/' -f1); \
		arch=$$(echo $$platform | cut -d'/' -f2); \
		output_name=$(BINARY_NAME)-$$os-$$arch; \
		if [ "$$os" = "windows" ]; then output_name=$$output_name.exe; fi; \
		echo "Building for $$os/$$arch..."; \
		GOOS=$$os GOARCH=$$arch go build $(GOFLAGS) $(LDFLAGS) \
			-o $(BUILD_DIR)/$$output_name ./cmd/miner; \
	done

# Protocol buffers
.PHONY: protoc
protoc:
	@echo "Generating protocol buffers..."
	protoc --go_out=. --go_opt=paths=source_relative pkg/mining/mining.proto

# Dependencies
.PHONY: deps
deps:
	@echo "Downloading dependencies..."
	go mod download
	go mod tidy

.PHONY: deps-update
deps-update:
	@echo "Updating dependencies..."
	go get -u ./...
	go mod tidy

# Clean
.PHONY: clean
clean:
	@echo "Cleaning build artifacts..."
	rm -rf $(BUILD_DIR)
	go clean
```

### GitHub Actions

**CI Pipeline** (`.github/workflows/ci.yml`):
```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go-version: [1.23.x]

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: ${{ matrix.go-version }}

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cache/go-build
          ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        restore-keys: |
          ${{ runner.os }}-go-

    - name: Install dependencies
      run: make deps

    - name: Run linter
      uses: golangci/golangci-lint-action@v3
      with:
        version: latest

    - name: Run tests
      run: make test

    - name: Run benchmarks
      run: make bench

    - name: Generate coverage
      run: make coverage

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out

  build:
    runs-on: ubuntu-latest
    needs: test

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: 1.23.x

    - name: Build for current platform
      run: make build

    - name: Cross-compile
      run: make cross-compile

    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: binaries
        path: build/

  integration:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push'

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: 1.23.x

    - name: Install dependencies
      run: make deps

    - name: Run integration tests
      run: make test-integration
      env:
        VALIDATOR_URL: ${{ secrets.TEST_VALIDATOR_URL }}
```

**Release Pipeline** (`.github/workflows/release.yml`):
```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: 1.23.x

    - name: Cross-compile
      run: make cross-compile

    - name: Create checksums
      run: |
        cd build
        sha256sum * > checksums.txt

    - name: Create Release
      uses: goreleaser/goreleaser-action@v4
      with:
        distribution: goreleaser
        version: latest
        args: release --clean
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Code Quality Tools

**golangci-lint Configuration** (`.golangci.yml`):
```yaml
run:
  timeout: 5m
  modules-download-mode: readonly

linters:
  enable:
    - gofmt
    - goimports
    - golint
    - govet
    - ineffassign
    - misspell
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - structcheck
    - varcheck
    - deadcode
    - typecheck
    - gosec
    - unconvert
    - dupl
    - goconst
    - gocyclo

linters-settings:
  gocyclo:
    min-complexity: 15
  dupl:
    threshold: 100
  goconst:
    min-len: 3
    min-occurrences: 3

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gosec
        - dupl
```
