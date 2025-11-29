# Certen Miner Networking

## LibP2P Integration Architecture

### Core LibP2P Components

The Certen miner builds upon the LibP2P networking stack, implementing a complete peer-to-peer communication system:

```
┌─────────────────────────────────────────────────────────────┐
│                    LibP2P Host Stack                       │
├─────────────────────────────────────────────────────────────┤
│  Application Layer                                          │
│  ├─ GossipSub (Message Routing)  ├─ Kademlia DHT          │
│  ├─ Audit Protocol              ├─ Heartbeat Protocol      │
├─────────────────────────────────────────────────────────────┤
│  Security Layer                                             │
│  ├─ Noise Protocol              ├─ Ed25519 Authentication   │
├─────────────────────────────────────────────────────────────┤
│  Transport Layer                                            │
│  ├─ QUIC (Primary)              ├─ TCP (Fallback)          │
│  ├─ WebRTC (Future)             ├─ WebSocket (Future)       │
├─────────────────────────────────────────────────────────────┤
│  Network Layer                                              │
│  ├─ IPv4/IPv6                   ├─ Multiaddr Routing        │
│  ├─ NAT Traversal               ├─ Relay Protocol           │
└─────────────────────────────────────────────────────────────┘
```

### Host Configuration

**LibP2P Host Creation**:
```go
func createLibP2PHost(config Config, logger *log.Logger) (host.Host, error) {
    // Load identity key for peer authentication
    privKey, err := identity.LoadPrivateKey(config.IdentityKey)
    if err != nil {
        return nil, fmt.Errorf("failed to load identity key: %w", err)
    }

    // Configure LibP2P host with security and transport options
    h, err := libp2p.New(
        // Network binding configuration
        libp2p.ListenAddrStrings(config.ListenAddrs...),

        // Identity and security
        libp2p.Identity(privKey),
        libp2p.Security(noise.ID, noise.New),

        // Transport protocols
        libp2p.DefaultTransports,  // TCP, QUIC, WebSocket

        // NAT traversal and connectivity
        libp2p.NATPortMap(),      // UPnP/NAT-PMP port mapping
        libp2p.EnableRelay(),     // Circuit relay support
        libp2p.EnableAutoRelay(), // Automatic relay discovery

        // Connection management
        libp2p.ConnectionManager(connmgr.NewConnManager(
            100,    // Low water mark
            400,    // High water mark
            time.Minute, // Grace period
        )),
    )

    if err != nil {
        return nil, fmt.Errorf("failed to create LibP2P host: %w", err)
    }

    logger.Printf("LibP2P host created with Peer ID: %s", h.ID())
    return h, nil
}
```

**Configuration Parameters**:
- **Listen Addresses**: Multiaddr format for network binding
- **Identity**: Ed25519 private key for peer authentication
- **Security**: Noise protocol for transport encryption
- **Transports**: QUIC (primary), TCP (fallback), WebSocket (web compatibility)
- **NAT Traversal**: Automatic port mapping and relay discovery

### Security Architecture

#### Ed25519 Identity System

**Key Properties**:
- **Algorithm**: EdDSA with Curve25519
- **Key Size**: 32 bytes private key, 32 bytes public key
- **Security Level**: 128-bit equivalent security
- **Performance**: Fast signing and verification operations

**Identity Generation**:
```go
func GeneratePrivateKey() (crypto.PrivKey, error) {
    // Use crypto/rand for secure random generation
    priv, _, err := crypto.GenerateEd25519Key(rand.Reader)
    if err != nil {
        return nil, fmt.Errorf("failed to generate Ed25519 key: %w", err)
    }
    return priv, nil
}
```

**Peer ID Derivation**:
```go
func GetPeerID(keyPath string) (peer.ID, error) {
    privKey, err := LoadPrivateKey(keyPath)
    if err != nil {
        return "", err
    }

    // Peer ID is deterministically derived from public key
    peerID, err := peer.IDFromPrivateKey(privKey)
    if err != nil {
        return "", fmt.Errorf("failed to derive peer ID: %w", err)
    }

    return peerID, nil
}
```

**Security Properties**:
- **Unforgeable Identity**: Private key possession proves identity
- **Deterministic**: Same private key always yields same peer ID
- **Collision Resistant**: Extremely low probability of peer ID collision
- **Forward Compatible**: LibP2P standard identity format

#### Noise Protocol Transport Encryption

**Handshake Pattern**: `XX` (mutual authentication with unknown static keys)

**Protocol Flow**:
1. **Initiator → Responder**: Ephemeral public key + encrypted static key
2. **Responder → Initiator**: Ephemeral public key + encrypted static key + authentication
3. **Initiator → Responder**: Final authentication message
4. **Secure Channel**: Established with forward secrecy

**Cryptographic Primitives**:
- **DH Function**: Curve25519 (same curve as Ed25519, different usage)
- **Cipher**: ChaCha20-Poly1305 AEAD
- **Hash Function**: BLAKE2s (256-bit output)

**Implementation**:
```go
import "github.com/libp2p/go-libp2p/p2p/security/noise"

// Noise security transport configuration
libp2p.Security(noise.ID, noise.New)
```

**Security Guarantees**:
- **Confidentiality**: All traffic encrypted with ephemeral keys
- **Integrity**: AEAD authentication prevents tampering
- **Forward Secrecy**: Compromise of long-term keys doesn't affect past sessions
- **Mutual Authentication**: Both peers verify each other's identity

## Peer Discovery Mechanisms

### Bootstrap Peer Strategy

**Bootstrap Configuration**:
```go
type Config struct {
    BootstrapPeers []string `yaml:"bootstrap_peers"`
    // Example:
    // - "/dns4/bootstrap1.certen.io/tcp/4001/p2p/12D3KooW..."
    // - "/ip4/203.0.113.1/tcp/4001/p2p/12D3KooW..."
    // - "/ip6/2001:db8::1/tcp/4001/p2p/12D3KooW..."
}
```

**Bootstrap Process**:
```go
func (n *MinerNode) connectToBootstrapPeers() error {
    for _, addr := range n.config.BootstrapPeers {
        multiAddr, err := ma.NewMultiaddr(addr)
        if err != nil {
            n.logger.Printf("Invalid bootstrap address %s: %v", addr, err)
            continue
        }

        addrInfo, err := peer.AddrInfoFromP2pAddr(multiAddr)
        if err != nil {
            n.logger.Printf("Failed to parse bootstrap peer %s: %v", addr, err)
            continue
        }

        // Connect with timeout
        ctx, cancel := context.WithTimeout(n.ctx, 30*time.Second)
        if err := n.host.Connect(ctx, *addrInfo); err != nil {
            n.logger.Printf("Failed to connect to bootstrap peer %s: %v",
                addrInfo.ID, err)
        } else {
            n.logger.Printf("Connected to bootstrap peer: %s", addrInfo.ID)
        }
        cancel()
    }

    return nil
}
```

**Bootstrap Peer Requirements**:
- **High Availability**: 99%+ uptime for network bootstrapping
- **Geographic Distribution**: Multiple regions for global accessibility
- **Scalability**: Ability to handle thousands of concurrent connections
- **Version Compatibility**: Support for protocol evolution

### Kademlia DHT Implementation

**DHT Configuration**:
```go
func setupDHT(ctx context.Context, h host.Host, logger *log.Logger) (*dht.IpfsDHT, error) {
    // Create Kademlia DHT with default configuration
    kademliaDHT, err := dht.New(ctx, h, dht.Mode(dht.ModeAuto))
    if err != nil {
        return nil, fmt.Errorf("failed to create DHT: %w", err)
    }

    // Start DHT bootstrap process
    if err = kademliaDHT.Bootstrap(ctx); err != nil {
        return nil, fmt.Errorf("failed to bootstrap DHT: %w", err)
    }

    // Periodic bootstrap maintenance
    go func() {
        ticker := time.NewTicker(15 * time.Minute)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                if err := kademliaDHT.Bootstrap(ctx); err != nil {
                    logger.Printf("DHT bootstrap maintenance failed: %v", err)
                }
            }
        }
    }()

    return kademliaDHT, nil
}
```

**Kademlia Algorithm Details**:

**Distance Metric**:
```go
func XORDistance(a, b peer.ID) *big.Int {
    // XOR distance between 160-bit peer IDs
    aBytes := []byte(a)
    bBytes := []byte(b)

    result := make([]byte, len(aBytes))
    for i := range aBytes {
        result[i] = aBytes[i] ^ bBytes[i]
    }

    return big.NewInt(0).SetBytes(result)
}
```

**Routing Table Structure**:
- **K-Buckets**: 160 buckets (one per bit position in peer ID space)
- **Bucket Capacity**: k=20 peers per bucket
- **Replacement Policy**: Least recently used (LRU) with liveness checks
- **Split Policy**: Buckets split when full and contain own peer ID

**DHT Operations**:

1. **FindPeer**: Locate contact information for specific peer ID
2. **FindProviders**: Discover peers providing specific content
3. **Provide**: Announce availability of content to the network
4. **GetValue**: Retrieve stored values by key
5. **PutValue**: Store key-value pairs in the DHT

**Bootstrap Algorithm**:
```go
func (dht *IpfsDHT) Bootstrap(ctx context.Context) error {
    // 1. Connect to seed peers (bootstrap peers)
    dht.connectToSeeds(ctx)

    // 2. Perform self-lookup to populate routing table
    _, err := dht.FindPeer(ctx, dht.PeerID())
    if err != nil {
        return fmt.Errorf("self-lookup failed: %w", err)
    }

    // 3. Random walks to fill routing table
    for i := 0; i < 3; i++ {
        randomID := generateRandomPeerID()
        dht.FindPeer(ctx, randomID)
    }

    return nil
}
```

### mDNS Local Discovery

**mDNS Service Configuration**:
```go
func setupMDNS(ctx context.Context, h host.Host, serviceTag string) error {
    // Create mDNS service for local network discovery
    mdnsService, err := mdns.NewMdnsService(ctx, h, time.Minute, serviceTag)
    if err != nil {
        return fmt.Errorf("failed to create mDNS service: %w", err)
    }

    // Register notifee for handling discovered peers
    mdnsService.RegisterNotifee(&discoveryNotifee{
        h:      h,
        logger: logger,
    })

    return nil
}

type discoveryNotifee struct {
    h      host.Host
    logger *log.Logger
}

func (n *discoveryNotifee) HandlePeerFound(pi peer.AddrInfo) {
    // Ignore self-discovery
    if pi.ID == n.h.ID() {
        return
    }

    n.logger.Printf("mDNS discovered peer: %s", pi.ID)

    // Attempt connection to discovered peer
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := n.h.Connect(ctx, pi); err != nil {
        n.logger.Printf("Failed to connect to discovered peer %s: %v", pi.ID, err)
    } else {
        n.logger.Printf("Connected to mDNS discovered peer: %s", pi.ID)
    }
}
```

**mDNS Properties**:
- **Service Name**: "_certen-miner._tcp.local"
- **Advertisement Interval**: 1 minute
- **Discovery Scope**: Local broadcast domain only
- **Protocol**: DNS-SD over multicast UDP
- **Port**: 5353 (standard mDNS port)

**Use Cases**:
- **Development**: Automatic peer discovery in local networks
- **Private Networks**: Isolated network deployments
- **LAN Mining**: Local mining pools without internet connectivity
- **Testing**: Simplified network setup for integration tests

## Transport Protocols

### QUIC Transport (Primary)

**QUIC Advantages**:
- **Low Latency**: Reduced handshake overhead compared to TCP+TLS
- **Built-in Encryption**: Integrated transport security
- **Multiplexing**: Multiple streams without head-of-line blocking
- **Connection Migration**: Survive IP address changes

**QUIC Configuration**:
```go
// QUIC listen address format
"/ip4/0.0.0.0/udp/4001/quic-v1"

// LibP2P automatically configures QUIC transport
libp2p.DefaultTransports // Includes QUIC
```

**Stream Management**:
```go
func (n *MinerNode) openAuditStream(peerID peer.ID) (network.Stream, error) {
    ctx, cancel := context.WithTimeout(n.ctx, 10*time.Second)
    defer cancel()

    // Open new stream for audit protocol
    stream, err := n.host.NewStream(ctx, peerID, "/certen/audit/1.0.0")
    if err != nil {
        return nil, fmt.Errorf("failed to open audit stream: %w", err)
    }

    return stream, nil
}
```

**Performance Characteristics**:
- **Connection Establishment**: 0-RTT for resumed connections
- **Bandwidth**: Efficient congestion control (BBR)
- **Latency**: 20-30% reduction vs TCP+TLS in high-latency networks
- **CPU Usage**: Optimized implementations available

### TCP Transport (Fallback)

**TCP Configuration**:
```go
// TCP listen address format
"/ip4/0.0.0.0/tcp/4001"

// TCP with Noise encryption for compatibility
libp2p.Transport(tcp.NewTCPTransport)
libp2p.Security(noise.ID, noise.New)
```

**Connection Management**:
```go
type ConnectionState struct {
    Stream       network.Stream
    Created      time.Time
    LastActivity time.Time
    BytesSent    uint64
    BytesRecv    uint64
}

func (cs *ConnectionState) IsHealthy() bool {
    // Check for recent activity
    if time.Since(cs.LastActivity) > 5*time.Minute {
        return false
    }

    // Check stream state
    return cs.Stream.Stat().Direction != network.DirUnknown
}
```

**TCP Optimization**:
- **Keep-Alive**: TCP keep-alive for connection health monitoring
- **Nagle Algorithm**: Disabled for low-latency mining communications
- **Buffer Sizes**: Optimized send/receive buffers
- **Congestion Control**: BBR or CUBIC depending on OS support

### WebSocket Transport (Future)

**WebSocket Use Cases**:
- **Browser Integration**: Web-based mining interfaces
- **Firewall Traversal**: Port 80/443 compatibility
- **Proxy Support**: HTTP proxy compatibility
- **Load Balancing**: Standard HTTP load balancer support

**Planned Implementation**:
```go
// WebSocket address format
"/ip4/127.0.0.1/tcp/8080/ws"

// WebSocket transport configuration
libp2p.Transport(websocket.New)
```

## GossipSub Message Routing

### Topic-Based Messaging

**Topic Configuration**:
```go
const (
    AuditTopic     = "certen/audit/v1"
    HeartbeatTopic = "certen/heartbeat/v1"
)

func (n *MinerNode) setupGossipSub() error {
    // Create GossipSub router
    ps, err := pubsub.NewGossipSub(n.ctx, n.host,
        pubsub.WithPeerExchange(true),
        pubsub.WithFloodPublish(true),
        pubsub.WithMessageSigning(true),
        pubsub.WithStrictSignatureVerification(true),
    )
    if err != nil {
        return fmt.Errorf("failed to create GossipSub: %w", err)
    }

    n.pubsub = ps

    // Join audit topic
    auditTopic, err := ps.Join(AuditTopic)
    if err != nil {
        return fmt.Errorf("failed to join audit topic: %w", err)
    }

    n.auditSub, err = auditTopic.Subscribe()
    if err != nil {
        return fmt.Errorf("failed to subscribe to audit topic: %w", err)
    }

    // Join heartbeat topic
    heartbeatTopic, err := ps.Join(HeartbeatTopic)
    if err != nil {
        return fmt.Errorf("failed to join heartbeat topic: %w", err)
    }

    n.heartbeatSub, err = heartbeatTopic.Subscribe()
    if err != nil {
        return fmt.Errorf("failed to subscribe to heartbeat topic: %w", err)
    }

    return nil
}
```

### Message Processing

**Audit Message Handler**:
```go
func (n *MinerNode) runAuditMessageLoop() {
    defer n.wg.Done()

    for {
        select {
        case <-n.ctx.Done():
            return
        default:
            // Receive message from topic
            msg, err := n.auditSub.Next(n.ctx)
            if err != nil {
                if n.ctx.Err() != nil {
                    return // Context cancelled
                }
                n.logger.Printf("Error receiving audit message: %v", err)
                continue
            }

            // Skip own messages
            if msg.ReceivedFrom == n.host.ID() {
                continue
            }

            // Process message in goroutine to avoid blocking
            go n.handleAuditMessage(msg)
        }
    }
}

func (n *MinerNode) handleAuditMessage(msg *pubsub.Message) {
    // Deserialize protocol buffer message
    var minerMsg miningpb.MinerMessage
    if err := proto.Unmarshal(msg.Data, &minerMsg); err != nil {
        n.logger.Printf("Failed to unmarshal audit message: %v", err)
        return
    }

    // Validate message signature
    if !n.validateMessageSignature(&minerMsg, msg.ReceivedFrom) {
        n.logger.Printf("Invalid signature from peer %s", msg.ReceivedFrom)
        return
    }

    // Process based on message type
    switch msgType := minerMsg.Msg.(type) {
    case *miningpb.MinerMessage_AuditBatch:
        n.processAuditBatch(msgType.AuditBatch, msg.ReceivedFrom)
    case *miningpb.MinerMessage_Heartbeat:
        n.processHeartbeat(msgType.Heartbeat, msg.ReceivedFrom)
    default:
        n.logger.Printf("Unknown message type from peer %s", msg.ReceivedFrom)
    }
}
```

### Message Publishing

**Audit Attestation Publishing**:
```go
func (n *MinerNode) publishAuditAttestation(attestation *miningpb.AuditAttestation) error {
    // Create batch with single attestation
    batch := &miningpb.AuditBatch{
        Attestations: []*miningpb.AuditAttestation{attestation},
        BatchId:      generateUUID(),
        Timestamp:    time.Now().Unix(),
    }

    // Wrap in miner message
    minerMsg := &miningpb.MinerMessage{
        Msg: &miningpb.MinerMessage_AuditBatch{
            AuditBatch: batch,
        },
        MessageId: generateUUID(),
        SenderId:  n.host.ID().String(),
        Timestamp: time.Now().Unix(),
    }

    // Sign message
    if err := n.signMessage(minerMsg); err != nil {
        return fmt.Errorf("failed to sign message: %w", err)
    }

    // Serialize to protocol buffer
    data, err := proto.Marshal(minerMsg)
    if err != nil {
        return fmt.Errorf("failed to marshal message: %w", err)
    }

    // Publish to audit topic
    if err := n.auditTopic.Publish(n.ctx, data); err != nil {
        return fmt.Errorf("failed to publish audit message: %w", err)
    }

    n.logger.Printf("Published audit attestation: block %d, score %.3f",
        attestation.BlockHeight, attestation.Score)

    return nil
}
```

### GossipSub Protocol Parameters

**Mesh Configuration**:
- **D**: 6 (target mesh degree)
- **D_low**: 4 (minimum mesh degree before grafting)
- **D_high**: 12 (maximum mesh degree before pruning)
- **D_lazy**: 6 (degree for lazy propagation)

**Timing Parameters**:
- **Heartbeat Interval**: 1 second
- **Fanout TTL**: 60 seconds
- **GossipRetransmission**: 3 times
- **GossipHistory**: 5 heartbeat intervals

**Score Parameters**:
```go
pubsub.WithPeerScore(
    &pubsub.PeerScoreParams{
        TopicScoreCap:    100,
        AppSpecificScore: n.calculatePeerScore,
        DecayInterval:    time.Second,
        DecayToZero:      0.01,
    },
    &pubsub.PeerScoreThresholds{
        GossipThreshold:   -10,
        PublishThreshold:  -50,
        GraylistThreshold: -100,
    },
)
```

## Connection Management

### Connection Pool Implementation

**Connection State Tracking**:
```go
type ConnectionManager struct {
    mu          sync.RWMutex
    connections map[peer.ID]*ConnectionInfo
    maxConns    int
    connTimeout time.Duration
}

type ConnectionInfo struct {
    Stream       network.Stream
    Reputation   float64
    LastSeen     time.Time
    MessageCount uint64
    BytesTransferred uint64
    Errors       uint32
}

func (cm *ConnectionManager) GetConnection(peerID peer.ID) (*ConnectionInfo, error) {
    cm.mu.RLock()
    if conn, exists := cm.connections[peerID]; exists {
        if conn.IsHealthy() {
            cm.mu.RUnlock()
            return conn, nil
        }
    }
    cm.mu.RUnlock()

    // Create new connection
    return cm.createConnection(peerID)
}
```

### Peer Reputation System

**Reputation Calculation**:
```go
func (n *MinerNode) calculatePeerReputation(peerID peer.ID) float64 {
    n.mu.RLock()
    info, exists := n.peers[peerID]
    if !exists {
        n.mu.RUnlock()
        return 0.5 // Neutral reputation for new peers
    }
    n.mu.RUnlock()

    reputation := 0.5 // Base reputation

    // Activity boost (up to +0.3)
    timeSinceLastSeen := time.Since(info.LastSeen)
    if timeSinceLastSeen < time.Hour {
        reputation += 0.3 * (1.0 - float64(timeSinceLastSeen)/float64(time.Hour))
    }

    // Message quality boost (up to +0.2)
    if info.MessageCount > 0 {
        errorRate := float64(info.Errors) / float64(info.MessageCount)
        reputation += 0.2 * (1.0 - errorRate)
    }

    // Bandwidth contribution (up to +0.1)
    if info.BytesTransferred > 1024*1024 { // 1MB threshold
        reputation += 0.1
    }

    // Clamp to [0.0, 1.0] range
    if reputation < 0.0 {
        reputation = 0.0
    } else if reputation > 1.0 {
        reputation = 1.0
    }

    return reputation
}
```

### NAT Traversal and Relay

**NAT Traversal Configuration**:
```go
libp2p.NATPortMap() // Enable UPnP/NAT-PMP

// Automatic relay configuration
libp2p.EnableAutoRelay(
    autorelay.WithCircuitV1Support(),
    autorelay.WithMaxCandidates(4),
    autorelay.WithBootDelay(30*time.Second),
)
```

**Circuit Relay Protocol**:
```go
func (n *MinerNode) enableRelayService() error {
    // Enable relay service for helping other peers
    _, err := relay.New(n.host, relay.WithResources(relay.Resources{
        Limit: &relay.ResourceLimitConfig{
            Duration: 2 * time.Minute,
            Data:     1 << 17, // 128KB
        },
        ReservationTTL: time.Hour,
        MaxReservations: 128,
        MaxCircuits:     16,
        BufferSize:      2048,
    }))

    return err
}
```
