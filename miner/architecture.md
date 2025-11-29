# Certen Miner Architecture

## System Overview

The Certen Miner is a standalone P2P mining node that implements a validator audit system using a LXR proof-of-work algorithm. 

### Core Design Principles

1. **Decentralization**: No central authority required for operation
2. **Modularity**: Clean separation of concerns with interface-based design
3. **Security**: Cryptographic identity and transport encryption throughout
4. **Performance**: Memory-hard proof-of-work with configurable resource usage
5. **Reliability**: Graceful error handling and automatic recovery mechanisms

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Certen Miner Process                    │
├─────────────────────────────────────────────────────────────┤
│  CLI Layer (Cobra)                                         │
│  ├─ init     ├─ run      ├─ status    ├─ validate         │
├─────────────────────────────────────────────────────────────┤
│  Core Mining Node                                          │
│  ├─ P2P Host (LibP2P)   ├─ LXR Engine   ├─ Proof Verifier │
│  ├─ DHT/GossipSub       ├─ Config Mgmt  ├─ Identity Mgmt   │
├─────────────────────────────────────────────────────────────┤
│  Network Layer                                             │
│  ├─ QUIC Transport      ├─ TCP Transport ├─ mDNS Discovery │
│  ├─ Noise Encryption    ├─ Peer Discovery                  │
└─────────────────────────────────────────────────────────────┘
```

## Package Structure

### Command Layer (`cmd/`)

#### `cmd/miner/main.go`
**Purpose**: Primary CLI entry point using Cobra framework
**Dependencies**: `pkg/cli`
**Responsibilities**:
- Command-line argument parsing
- Global configuration initialization
- Error handling and process termination

#### `cmd/integrated-miner/main.go`
**Purpose**: Standalone binary with embedded configuration for containerized deployments
**Dependencies**: Direct imports of core packages
**Responsibilities**:
- Environment variable configuration
- Embedded miner node creation
- Signal handling for graceful shutdown

### Core Packages (`pkg/`)

#### Configuration Management (`pkg/config/`)

**File**: `config.go`
**Primary Type**: `Config`
**Dependencies**: Viper, UUID, YAML

```go
type Config struct {
    MinerID        string        `yaml:"miner_id"`
    ValidatorURL   string        `yaml:"validator_url"`
    ListenAddrs    []string      `yaml:"listen_addrs"`
    BootstrapPeers []string      `yaml:"bootstrap_peers"`
    AuditInterval  time.Duration `yaml:"audit_interval"`
    DataDir        string        `yaml:"data_dir"`
    IdentityKey    string        `yaml:"identity_key"`
    LogLevel       string        `yaml:"log_level"`
}
```

**Key Functions**:
- `LoadConfig(path string)`: YAML deserialization with validation
- `SaveConfig(cfg *Config, path string)`: Atomic configuration writing
- `DefaultConfig()`: Sensible defaults for new installations
- `ExpandPath(path string)`: Home directory resolution

#### Identity Management (`pkg/identity/`)

**File**: `identity.go`
**Primary Functions**: Ed25519 key management for LibP2P
**Dependencies**: LibP2P crypto, peer ID generation

**Key Functions**:
- `GeneratePrivateKey()`: Secure Ed25519 key generation using crypto/rand
- `LoadPrivateKey(path string)`: Binary key file loading with permission checks
- `SavePrivateKey(key crypto.PrivKey, path string)`: Atomic key writing with 0600 permissions
- `GetPeerID(keyPath string)`: Deterministic peer ID derivation from public key

**Security Considerations**:
- Private keys never appear in logs or network traffic
- File permissions strictly enforced (0600)
- Peer IDs serve as both identity and routing addresses
- Compatible with LibP2P Noise protocol encryption

#### CLI Interface (`pkg/cli/`)

**Files**: `root.go`, `init.go`, `run.go`, `status.go`, `validate.go`
**Framework**: Cobra with Viper integration
**Design Pattern**: Command pattern with shared configuration state

##### Command Implementations

**`init.go`** - Initialization Command:
```go
func runInit(cmd *cobra.Command, args []string) error {
    // 1. Create directory structure (~/.certen/miner)
    // 2. Generate Ed25519 identity key
    // 3. Create default YAML configuration
    // 4. Validate configuration completeness
}
```

**`run.go`** - Mining Operation Command:
```go
func runMiner(cmd *cobra.Command, args []string) error {
    // 1. Load and validate configuration
    // 2. Initialize LXR engine with memory parameters
    // 3. Create proof verifier with validator API settings
    // 4. Start P2P node with DHT and GossipSub
    // 5. Begin audit and heartbeat loops
    // 6. Handle graceful shutdown on signals
}
```

**`validate.go`** - Environment Validation:
- Configuration file parsing validation
- Identity key accessibility verification
- Validator API connectivity testing
- Network port binding validation
- Dependency availability checks

#### LXR Mining Engine (`pkg/lxr/`)

**Files**: `lxrhash.go` (Accumulate implementation), `lxr_runner.go` (Mining wrapper)

##### Core Algorithm (`lxrhash.go`)

**LxrPow Structure**:
```go
type LxrPow struct {
    Loops   uint64  // Number of translation passes
    ByteMap []byte  // Randomized translation table
    MapBits uint64  // Size parameter (2^MapBits = MapSize)
    MapSize uint64  // Actual table size in bytes
    Passes  uint64  // ByteMap randomization iterations
}
```

**Algorithm Phases**:
1. **Initialization**: `GenerateByteMap()` creates randomized lookup table
2. **Mixing**: `mix()` combines challenge hash with nonce using XorShift operations
3. **Translation**: Multiple loops through ByteMap with state evolution
4. **Extraction**: Final proof-of-work value with difficulty encoding

**Memory Usage**:
- TableBits 20: 1MB (development)
- TableBits 25: 32MB (production)
- TableBits 30: 1GB (maximum security)

##### Mining Wrapper (`lxr_runner.go`)

**LXRRunner Structure**:
```go
type LXRRunner struct {
    config        *Config
    logger        *log.Logger
    mu            sync.RWMutex
    hashCount     uint64
    lastHashCount uint64
    lastTime      time.Time
    lastRate      float64
    lxr           *LxrPow
}
```

**Core Mining Function**:
```go
func (r *LXRRunner) VerifyWithProof(bundle *ValidatorProofBundle) (*AuditResult, error) {
    // 1. Validate proof bundle structure
    // 2. Generate 32-byte challenge from bundle hash
    // 3. Calculate difficulty based on block height
    // 4. Perform LXR proof-of-work mining
    // 5. Validate proof bundle contents semantically
    // 6. Return audit result with LXR proof
}
```

**Difficulty Calculation**:
```go
func (r *LXRRunner) calculateDifficulty(blockHeight uint64) uint64 {
    baseDifficulty := uint64(1000)
    heightFactor := blockHeight / 1000
    return baseDifficulty + (heightFactor * 100)
}
```

#### Proof Verification (`pkg/verifier/`)

**File**: `proof_verifier.go`
**Primary Type**: `ProofVerifier`
**Dependencies**: HTTP client, JSON processing, caching

**ProofVerifier Structure**:
```go
type ProofVerifier struct {
    config             *Config
    logger             *log.Logger
    httpClient         *http.Client
    mu                 sync.RWMutex
    lastHeight         uint64
    lastVerifiedHeight uint64
    heightCache        map[uint64]*ValidatorProofBundle
}
```

**API Integration**:
- `GET /api/v1/validator/latest`: Current block height
- `GET /api/v1/validator/proof/{height}`: Canonical JSON proof bundle
- HTTP timeout configuration (30 seconds default)
- Automatic retry logic with exponential backoff

**Caching Strategy**:
- In-memory LRU cache (100 blocks maximum)
- Cache key: block height
- Cache eviction: automatic cleanup of old entries
- Cache hits logged for performance monitoring

#### P2P Networking (`pkg/node/`)

**Files**: `node.go` (implementation), `types.go` (interfaces and data structures)

##### Interface Definitions (`types.go`)

**LXRRunner Interface**:
```go
type LXRRunner interface {
    VerifyWithProof(bundle *ValidatorProofBundle) (*AuditResult, error)
    GetHashRate() float64
}
```

**ProofVerifier Interface**:
```go
type ProofVerifier interface {
    LatestHeight() uint64
    FetchProofBundle(height uint64) (*ValidatorProofBundle, error)
    LastVerifiedHeight() uint64
}
```

**Data Structures**:
- `ValidatorProofBundle`: Canonical proof data from validator
- `AuditResult`: Mining result with LXR proof and quality score
- `TxProof`: Individual transaction proof within bundle
- `Config`: Node configuration parameters

##### P2P Implementation (`node.go`)

**MinerNode Structure**:
```go
type MinerNode struct {
    ctx           context.Context
    config        Config
    host          host.Host
    dht           *dht.IpfsDHT
    pubsub        *pubsub.PubSub
    auditSub      *pubsub.Subscription
    heartbeatSub  *pubsub.Subscription
    lxr           LXRRunner
    verifier      ProofVerifier
    logger        *log.Logger
}
```

**LibP2P Host Configuration**:
```go
func createLibP2PHost(config Config, logger *log.Logger) (host.Host, error) {
    return libp2p.New(
        libp2p.ListenAddrStrings(config.ListenAddrs...),
        libp2p.Security(noise.ID, noise.New),
        libp2p.DefaultTransports, // QUIC + TCP
        libp2p.NATPortMap(),
        libp2p.EnableRelay(),
        libp2p.EnableAutoRelay(),
    )
}
```

**DHT Configuration**:
```go
func setupDHT(ctx context.Context, h host.Host, logger *log.Logger) (*dht.IpfsDHT, error) {
    kademliaDHT, err := dht.New(ctx, h)
    if err != nil {
        return nil, err
    }

    if err = kademliaDHT.Bootstrap(ctx); err != nil {
        return nil, err
    }

    return kademliaDHT, nil
}
```

#### Protocol Definitions (`pkg/mining/`)

**Files**: `mining.proto` (schema), `mining.pb.go` (generated code)

**Protocol Buffer Schema**:
```protobuf
syntax = "proto3";
package mining;
option go_package = "github.com/certenIO/miner/pkg/mining";

message MinerMessage {
  oneof msg {
    AuditBatch audit_batch = 1;
    MinerHeartbeat heartbeat = 2;
    MisbehaviorReport misbehavior = 3;
    ChallengeRequest challenge_req = 4;
    ChallengeResponse challenge_resp = 5;
  }

  string message_id = 10;
  string sender_id = 11;
  int64 timestamp = 12;
  bytes signature = 13;
}

message LXRProof {
  bytes challenge = 1;
  uint64 nonce = 2;
  bytes solution = 3;
  uint64 difficulty = 4;
  uint32 table_bits = 5;
  uint32 loops = 6;
  uint32 passes = 7;
}
```

## Data Flow Architecture

### Initialization Flow

1. **Key Generation**: Ed25519 private key created with crypto/rand
2. **Configuration**: Default YAML configuration written to ~/.certen/miner/
3. **Validation**: Environment checks for connectivity and dependencies
4. **Identity**: Peer ID derived from public key for LibP2P operations

### Mining Operation Flow

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Proof Fetcher  │───▶│   LXR Engine     │───▶│  P2P Publisher  │
│                 │    │                  │    │                 │
│ 1. Latest Height│    │ 1. Hash Bundle   │    │ 1. Attestation  │
│ 2. Fetch Bundle │    │ 2. Calculate     │    │ 2. Broadcast    │
│ 3. Validate     │    │    Difficulty    │    │ 3. Log Result   │
│ 4. Cache        │    │ 3. Mine LXR      │    │                 │
│                 │    │ 4. Verify Proof  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

### Network Communication Flow

1. **Bootstrap**: Connect to known peers via multiaddr
2. **Discovery**: DHT population and mDNS local discovery
3. **Subscription**: Join GossipSub topics for audit and heartbeat
4. **Broadcasting**: Publish attestations and receive peer messages
5. **Maintenance**: Periodic DHT refresh and peer health checks

### Error Handling Strategy

**Graceful Degradation**:
- Network failures: Retry with exponential backoff
- Validator API errors: Cache previous results and continue
- Mining failures: Log errors and attempt next block
- Peer disconnections: Automatic reconnection attempts

**Recovery Mechanisms**:
- DHT bootstrap every 15 minutes
- Peer reputation tracking with automatic removal
- Circuit breaker pattern for failing validators
- Graceful shutdown on system signals

## Interface Design Patterns

### Dependency Injection

The system uses interface-based dependency injection to enable testing and modularity:

```go
type MinerNode struct {
    lxr      LXRRunner      // Interface for mining engine
    verifier ProofVerifier  // Interface for proof fetching
}

func NewMinerNode(config Config, lxr LXRRunner, verifier ProofVerifier) *MinerNode {
    return &MinerNode{
        config:   config,
        lxr:      lxr,
        verifier: verifier,
    }
}
```

### Observer Pattern

The node implements observer pattern for P2P message handling:

```go
func (n *MinerNode) runAuditMessageLoop() {
    for {
        msg, err := n.auditSub.Next(n.ctx)
        if err != nil {
            continue
        }

        // Process audit attestation from peer
        go n.handleAuditMessage(msg)
    }
}
```

### Builder Pattern

Configuration follows builder pattern for complex object creation:

```go
config := config.DefaultConfig()
config.SetValidatorURL("https://validator.certen.io")
config.SetAuditInterval(5 * time.Second)
config.AddBootstrapPeer("/dns4/bootstrap.certen.io/tcp/4001/p2p/...")
```
