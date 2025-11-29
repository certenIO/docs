# Certen Miner Algorithms

## LXR Proof-of-Work Algorithm

### Overview

The LXR (Lookup XOR Randomization) algorithm is a memory-hard proof-of-work function designed to resist ASIC optimization through random memory access patterns

### Algorithm Components

#### 1. Initialization Phase

**ByteMap Generation**:
```go
func (lx *LxrPow) GenerateByteMap() {
    // Initialize sequential values
    for i := range lx.ByteMap {
        lx.ByteMap[i] = byte(i)
    }

    // Shuffle with cryptographic randomization
    for i := uint64(0); i < lx.Passes; i++ {
        seed := uint64(0xFAB3C4E9D2A18567) + i
        for j := uint64(0); j < lx.MapSize; j++ {
            // XorShift random number generation
            seed = seed<<19 ^ seed>>7 ^ j
            k := seed & (lx.MapSize - 1)
            // Swap bytes for randomization
            lx.ByteMap[j], lx.ByteMap[k] = lx.ByteMap[k], lx.ByteMap[j]
        }
    }
}
```

**Parameters**:
- **MapSize**: 2^MapBits bytes (1MB to 1GB typical range)
- **Passes**: Number of shuffling iterations (6 recommended)
- **Seed**: Fixed constant 0xFAB3C4E9D2A18567 for deterministic generation

**Security Properties**:
- Uniform distribution of byte values after shuffling
- Deterministic but pseudorandom memory layout
- Resistance to precomputation attacks due to large state space

#### 2. Challenge Preparation (Mix Phase)

**Function Signature**:
```go
func (lx *LxrPow) mix(hash []byte, nonce uint64) ([32]byte, uint64)
```

**Input Requirements**:
- `hash`: Exactly 32 bytes (SHA256 output)
- `nonce`: 64-bit iteration counter

**Algorithm Steps**:

1. **State Initialization**:
```go
state := uint64(0xb32c25d16e362f32)  // Fixed initialization constant
nonce ^= nonce<<19 ^ state           // XorShift randomization
nonce ^= nonce >> 1
nonce ^= nonce << 3
```

2. **Sponge Construction**:
```go
var array [5]uint64
array[0] = nonce
for i := 1; i < 5; i++ {
    array[i] = binary.BigEndian.Uint64(hash[(i-1)<<3:])  // 8 bytes each
    state ^= state<<7 ^ array[i]
    state ^= state >> 3
    state ^= state << 5
}
```

3. **State Mixing**:
```go
for i, a := range array {
    left1 := (a & 0x1F) | 1                // Random shift 1-32
    right := ((a & 0xF0) >> 5) | 1         // Random shift 1-16
    left2 := ((a&0x1F0000)>>17)<<1 ^ left1 // Different from left1

    state ^= state<<left1 ^ a<<5
    state ^= state >> right
    state ^= state << left2
    array[i] ^= state
}
```

4. **Hash Extraction**:
```go
var newHash [32]byte
for i, a := range array[:4] {
    a ^= state
    binary.BigEndian.PutUint64(newHash[i*8:], a)
    state ^= state<<27 ^ uint64(i)<<19
    state ^= state >> 11
    state ^= state << 13
}
```

**Security Analysis**:
- Avalanche effect: Single bit change affects entire output
- Nonlinearity: XOR and bit shift operations prevent linear attacks
- State space: 2^320 possible internal states (5×64-bit array + state)
- Pseudorandomness: Passes statistical tests for randomness

#### 3. Translation Phase

**Core Algorithm**:
```go
func (lx *LxrPow) LxrPoWHash(hash []byte, nonce uint64) ([]byte, uint64) {
    mask := lx.MapSize - 1
    LHash, state := lx.mix(hash, nonce)

    // Translation loops
    for i := uint64(0); i < lx.Loops; i++ {
        for j := range LHash {
            b := uint64(lx.ByteMap[state&mask])  // Memory lookup
            v := uint64(LHash[j])                // Current hash byte

            // XorShift evolution with memory input
            state ^= state<<17 ^ b<<31
            state ^= state>>7 ^ v<<23
            state ^= state << 5

            LHash[j] = byte(state)              // Update hash
        }
        time.Sleep(0)  // Cooperative scheduling
    }

    _, state = lx.mix(LHash[:], state)
    return LHash[:], state
}
```

**Memory Access Pattern**:
- **Address Calculation**: `state & mask` creates uniform distribution
- **Cache Behavior**: Random access pattern defeats CPU cache optimization
- **Memory Bandwidth**: Full utilization of available memory bandwidth
- **ASIC Resistance**: Memory access cost dominates computation cost

**Performance Characteristics**:
- **Memory**: 2^MapBits bytes (configurable)
- **Compute**: O(Loops × 32) memory lookups per hash
- **Latency**: Memory access latency becomes bottleneck
- **Parallelization**: Limited by memory bandwidth, not compute units

### Difficulty Encoding and Validation

#### Difficulty Representation

The LXR algorithm encodes difficulty as the number of leading 0xFF bytes in the proof-of-work value:

```
Difficulty 1:      Any value (always passes)
Difficulty 256:    0xFF________ (1/256 probability)
Difficulty 65536:  0xFFFF______ (1/65536 probability)
Difficulty 16M:    0xFFFFFF____ (1/16777216 probability)
```

#### Validation Algorithm

```go
func checkLXRDifficulty(pow uint64, targetDifficulty uint64) bool {
    if targetDifficulty <= 1 {
        return true  // Minimal difficulty for testing
    }

    // Count leading 0xFF bytes
    leadingFFBytes := 0
    for shift := uint(56); shift <= 56; shift -= 8 {
        if byte(pow>>shift) == 0xFF {
            leadingFFBytes++
        } else {
            break
        }
    }

    // Calculate required leading bytes
    requiredLeadingBytes := 0
    remainingDifficulty := targetDifficulty
    for remainingDifficulty >= 256 && requiredLeadingBytes < 8 {
        remainingDifficulty /= 256
        requiredLeadingBytes++
    }

    // Exact match: check remaining portion
    if leadingFFBytes == requiredLeadingBytes {
        if requiredLeadingBytes == 0 {
            threshold := ^uint64(0) / targetDifficulty
            return pow >= threshold
        }

        mask := ^uint64(0) >> (uint(requiredLeadingBytes) * 8)
        nonFFPortion := pow & mask
        threshold := mask / remainingDifficulty
        return nonFFPortion >= threshold
    }

    return leadingFFBytes > requiredLeadingBytes
}
```

**Mathematical Foundation**:
- **Probability**: P(success) = 1/difficulty for uniform distribution
- **Expected Attempts**: E[attempts] = difficulty
- **Variance**: Var[attempts] = difficulty² (geometric distribution)

### Dynamic Difficulty Adjustment

#### Height-Based Scaling

```go
func (r *LXRRunner) calculateDifficulty(blockHeight uint64) uint64 {
    baseDifficulty := uint64(1000)           // 1000 expected hashes
    heightFactor := blockHeight / 1000        // Increase every 1000 blocks
    return baseDifficulty + (heightFactor * 100)  // +100 per 1000 blocks
}
```

**Design Rationale**:
- **Simplicity**: Linear scaling avoids complex adjustment algorithms
- **Predictability**: Deterministic difficulty based on block height
- **Gradualism**: Slow increase prevents sudden computational spikes
- **Future-Proofing**: Can be replaced with more sophisticated algorithms

#### Alternative Algorithms (Future Considerations)

**Exponential Scaling**:
```go
func exponentialDifficulty(blockHeight uint64) uint64 {
    base := uint64(1000)
    exponent := blockHeight / 10000
    return base * (1 << exponent)  // Double every 10k blocks
}
```

**Time-Based Adjustment**:
```go
func timeDifficulty(targetInterval time.Duration, actualInterval time.Duration, currentDiff uint64) uint64 {
    ratio := float64(actualInterval) / float64(targetInterval)
    return uint64(float64(currentDiff) * ratio)
}
```

### Hash Rate Calculation

#### Rate Measurement

```go
func (r *LXRRunner) GetHashRate() float64 {
    r.mu.Lock()
    defer r.mu.Unlock()

    now := time.Now()
    currentHashCount := atomic.LoadUint64(&r.hashCount)
    duration := now.Sub(r.lastTime)

    // Rate limiting for frequent calls
    if duration < time.Second {
        return r.lastRate
    }

    // Calculate rate over interval
    hashDiff := currentHashCount - r.lastHashCount
    rate := float64(hashDiff) / duration.Seconds()

    // Update state for next calculation
    r.lastHashCount = currentHashCount
    r.lastTime = now
    r.lastRate = rate

    return rate
}
```

**Measurement Considerations**:
- **Atomic Operations**: Thread-safe counter incrementation
- **Rate Limiting**: Avoid calculation overhead for frequent queries
- **Moving Average**: Could be implemented for smoother reporting
- **Precision**: Float64 provides sufficient precision for typical hash rates

#### Performance Metrics

**Memory Bandwidth Requirements**:
- **1MB Table**: ~10 MB/s sustained memory bandwidth
- **32MB Table**: ~320 MB/s sustained memory bandwidth
- **1GB Table**: ~10 GB/s sustained memory bandwidth

**Expected Hash Rates** (approximate):
- **Consumer CPU**: 100-1000 H/s (1MB table)
- **Server CPU**: 1000-5000 H/s (32MB table)
- **High-end Workstation**: 5000-15000 H/s (1GB table)

## Proof Bundle Validation Algorithm

### Structural Validation

```go
func (v *ProofVerifier) validateBundle(bundle *node.ValidatorProofBundle) error {
    if bundle.ValidatorID == "" {
        return fmt.Errorf("empty validator ID")
    }

    if bundle.BlockHeight == 0 {
        return fmt.Errorf("zero block height")
    }

    if bundle.AnchorRoot == "" {
        return fmt.Errorf("empty anchor root")
    }

    if len(bundle.AnchorProof) == 0 {
        return fmt.Errorf("empty anchor proof")
    }

    // Transaction proof validation
    for i, txProof := range bundle.TxProofs {
        if txProof.TxID == "" {
            return fmt.Errorf("transaction proof %d has empty TX ID", i)
        }
        if len(txProof.Proof) == 0 {
            return fmt.Errorf("transaction proof %d has empty proof", i)
        }
        if txProof.Status == "" {
            return fmt.Errorf("transaction proof %d has empty status", i)
        }
    }

    // Timestamp reasonableness
    now := time.Now().Unix()
    if bundle.Timestamp < now-86400 || bundle.Timestamp > now+300 {
        return fmt.Errorf("unreasonable timestamp: %d", bundle.Timestamp)
    }

    return nil
}
```

### Semantic Quality Scoring

```go
func (r *LXRRunner) validateProofBundle(bundle *node.ValidatorProofBundle) (float64, string) {
    score := 1.0
    var issues []string

    // Anchor root assessment (30% weight)
    if len(bundle.AnchorRoot) == 0 {
        score -= 0.3
        issues = append(issues, "missing_anchor_root")
    }

    // Anchor proof assessment (20% weight)
    if len(bundle.AnchorProof) == 0 {
        score -= 0.2
        issues = append(issues, "missing_anchor_proof")
    }

    // Transaction proof assessment (variable weight)
    if len(bundle.TxProofs) == 0 {
        score -= 0.1
        issues = append(issues, "no_tx_proofs")
    } else {
        validTxProofs := 0
        for _, txProof := range bundle.TxProofs {
            if txProof.TxID != "" && len(txProof.Proof) > 0 {
                validTxProofs++
            }
        }

        txValidityRatio := float64(validTxProofs) / float64(len(bundle.TxProofs))
        if txValidityRatio < 0.8 {
            score -= 0.2 * (1.0 - txValidityRatio)
            issues = append(issues, "invalid_tx_proofs")
        }
    }

    // Timestamp validation (20% weight)
    now := time.Now().Unix()
    if bundle.Timestamp > now+600 { // Future timestamps
        score -= 0.2
        issues = append(issues, "future_timestamp")
    }

    // Ensure non-negative score
    if score < 0 {
        score = 0
    }

    reason := "OK"
    if len(issues) > 0 {
        reason = fmt.Sprintf("issues: %v", issues)
    }

    return score, reason
}
```

### Challenge Generation Algorithm

```go
func (r *LXRRunner) hashBundle(bundle *node.ValidatorProofBundle) []byte {
    h := sha256.New()

    // Deterministic ordering for consistent hashing
    h.Write([]byte(bundle.ValidatorID))
    binary.Write(h, binary.LittleEndian, bundle.BlockHeight)
    h.Write([]byte(bundle.AnchorRoot))
    h.Write(bundle.AnchorProof)

    // Transaction proofs in order
    for _, txProof := range bundle.TxProofs {
        h.Write([]byte(txProof.TxID))
        h.Write(txProof.Proof)
        h.Write([]byte(txProof.Status))
    }

    binary.Write(h, binary.LittleEndian, bundle.Timestamp)

    sum := h.Sum(nil)
    if len(sum) != 32 {
        panic("hashBundle must produce a 32-byte hash")
    }
    return sum
}
```

**Properties**:
- **Deterministic**: Same bundle always produces same hash
- **Avalanche Effect**: Small changes dramatically alter hash
- **Collision Resistant**: SHA256 security properties
- **Canonical**: Consistent field ordering prevents ambiguity

## P2P Discovery Algorithms

### Kademlia DHT Implementation

The miner uses LibP2P's implementation of Kademlia DHT for peer discovery:

**Distance Metric**:
```go
func XORDistance(a, b peer.ID) *big.Int {
    ab := XOR([]byte(a), []byte(b))
    return big.NewInt(0).SetBytes(ab)
}
```

**Routing Table**:
- **K-Buckets**: Each bucket contains up to k=20 peers
- **Bucket Index**: log₂(XOR distance) determines bucket
- **Replacement Policy**: LRU with liveness checks

**Bootstrap Process**:
```go
func (dht *IpfsDHT) Bootstrap(ctx context.Context) error {
    // 1. Connect to bootstrap peers
    // 2. Perform self-lookup to populate routing table
    // 3. Periodic refresh every 15 minutes
    // 4. Background maintenance of routing table
}
```

### mDNS Local Discovery

```go
func setupMDNS(ctx context.Context, h host.Host, serviceTag string) error {
    mdnsService, err := mdns.NewMdnsService(ctx, h, time.Minute, serviceTag)
    if err != nil {
        return err
    }

    mdnsService.RegisterNotifee(&discoveryNotifee{h: h})
    return nil
}
```

**Service Announcement**:
- **Service Name**: "certen-miner"
- **Interval**: 1 minute advertisements
- **Discovery**: Automatic connection to discovered local peers
- **Scope**: Local network broadcast domain only

## Cryptographic Algorithms

### Ed25519 Key Management

**Key Generation**:
```go
func GeneratePrivateKey() (crypto.PrivKey, error) {
    priv, _, err := crypto.GenerateEd25519Key(rand.Reader)
    return priv, err
}
```

**Peer ID Derivation**:
```go
func GetPeerID(keyPath string) (peer.ID, error) {
    privKey, err := LoadPrivateKey(keyPath)
    if err != nil {
        return "", err
    }

    return peer.IDFromPrivateKey(privKey)
}
```

**Properties**:
- **Algorithm**: Ed25519 (Curve25519 + EdDSA)
- **Key Size**: 32 bytes private, 32 bytes public
- **Security**: 128-bit security level
- **Performance**: Fast signature generation and verification
- **Determinism**: Same private key always generates same peer ID

### Noise Protocol Encryption

LibP2P uses Noise Protocol for transport encryption:

**Handshake Pattern**: `XX` (mutual authentication)
**Cryptographic Primitives**:
- **DH**: Curve25519 (same as Ed25519 but for ECDH)
- **Cipher**: ChaCha20-Poly1305
- **Hash**: BLAKE2s

**Security Properties**:
- **Forward Secrecy**: Ephemeral keys rotated per connection
- **Mutual Authentication**: Both peers verify identity
- **Replay Protection**: Nonce-based anti-replay
- **Post-Quantum Considerations**: Curve25519 vulnerable to quantum computers

## Performance Optimization Algorithms

### Memory Access Optimization

**Cache-Conscious Design**:
```go
// Avoid this: random stride patterns
for i := 0; i < iterations; i++ {
    index := rand.Uint64() % tableSize
    result ^= table[index]
}

// Prefer this: sequential access where possible
for i := 0; i < iterations; i++ {
    index := (baseIndex + i) % tableSize
    result ^= table[index]
}
```

**Prefetch Hints** (Future Optimization):
```go
func prefetchMemory(addr unsafe.Pointer) {
    // Platform-specific prefetch intrinsics
    runtime.Prefetch(addr)
}
```

### Concurrent Mining Strategies

**Single-Threaded Approach** (Current):
- **Advantages**: Simpler implementation, deterministic behavior
- **Disadvantages**: Underutilizes multi-core systems
- **Memory**: Single ByteMap shared across nonce iterations

**Multi-Threaded Approach** (Future):
```go
func (r *LXRRunner) parallelMining(challenge []byte, difficulty uint64, numWorkers int) *LXRProof {
    resultChan := make(chan *LXRProof, 1)

    for i := 0; i < numWorkers; i++ {
        go func(startNonce uint64) {
            for nonce := startNonce; ; nonce += uint64(numWorkers) {
                _, pow := r.lxr.LxrPoWHash(challenge, nonce)
                if checkLXRDifficulty(pow, difficulty) {
                    select {
                    case resultChan <- &LXRProof{Nonce: nonce, Solution: pow}:
                    default:
                    }
                    return
                }
            }
        }(uint64(i))
    }

    return <-resultChan
}
```

### Network Optimization

**Connection Pooling**:
```go
type ConnectionPool struct {
    mu          sync.RWMutex
    connections map[peer.ID]*ConnectionState
    maxConns    int
}

func (p *ConnectionPool) GetConnection(pid peer.ID) (*ConnectionState, error) {
    p.mu.RLock()
    if conn, exists := p.connections[pid]; exists && conn.IsHealthy() {
        p.mu.RUnlock()
        return conn, nil
    }
    p.mu.RUnlock()

    return p.createConnection(pid)
}
```

**Message Batching**:
```go
func (n *MinerNode) batchAudits(attestations []*AuditAttestation) *AuditBatch {
    return &AuditBatch{
        Attestations: attestations,
        BatchID:      generateBatchID(),
        Timestamp:    time.Now().Unix(),
    }
}
```
