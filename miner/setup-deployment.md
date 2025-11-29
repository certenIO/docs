# Certen Miner Setup and Deployment

## Development Environment Setup

### Prerequisites

**System Requirements**:
- **Go**: Version 1.23 or higher
- **Git**: For source code management
- **Make**: For cross-platform build system (Linux/macOS) or equivalent
- **Protocol Buffers**: For message schema compilation (optional)

**Platform Support**:
- **Development**: Windows, macOS, Linux
- **Production**: Windows, macOS, Linux (AMD64/ARM64)
- **Containers**: Docker, Kubernetes, containerd

**Hardware Requirements**:
```
Minimum (Development):
- CPU: 2 cores, 2.0 GHz
- Memory: 4 GB RAM (includes 1MB LXR table)
- Storage: 1 GB available space
- Network: Broadband internet connection

Recommended (Production):
- CPU: 4+ cores, 3.0 GHz
- Memory: 8 GB RAM (includes 32MB LXR table)
- Storage: 10 GB available space (logs, data)
- Network: High-speed internet with public IP (or NAT traversal)

High-Performance (Mining Farm):
- CPU: 8+ cores, 4.0 GHz
- Memory: 32+ GB RAM (includes 1GB LXR table)
- Storage: 100 GB SSD (high-speed logging)
- Network: Dedicated connection, multiple IPs
```

### Source Code Setup

**Clone Repository**:
```bash
git clone https://github.com/certenIO/miner.git
cd miner
```

**Verify Go Environment**:
```bash
go version  # Should show 1.23+
go env GOPATH GOROOT
```

**Download Dependencies**:
```bash
# Disable Go workspace to avoid conflicts
export GOWORK=off

# Download and verify dependencies
go mod download
go mod verify

# Optional: vendor dependencies for offline builds
go mod vendor
```

**Build Development Binary**:
```bash
# Build for current platform
go build -o certen-miner ./cmd/miner

# Test basic functionality
./certen-miner --version
./certen-miner --help
```

### IDE Configuration

**VS Code Configuration** (`.vscode/settings.json`):
```json
{
    "go.useLanguageServer": true,
    "go.lintTool": "golangci-lint",
    "go.formatTool": "gofumpt",
    "go.testFlags": ["-v", "-race"],
    "go.buildFlags": ["-trimpath"],
    "go.toolsEnvVars": {
        "GOWORK": "off"
    },
    "files.exclude": {
        "build/": true,
        "vendor/": true
    }
}
```

**GoLand Configuration**:
- Enable Go modules (vgo) integration
- Configure GOWORK=off in environment
- Set up run configurations for different commands
- Enable race detection for testing

### Testing Setup

**Unit Tests**:
```bash
# Run all tests with race detection
go test -race -v ./...

# Run tests with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Benchmark tests
go test -bench=. -benchmem ./pkg/lxr/
```

**Integration Tests**:
```bash
# Start test validator (if available)
export VALIDATOR_URL=http://localhost:8080

# Run integration tests
go test -tags=integration -v ./...
```

## Configuration Management

### Configuration File Structure

**Default Location**: `~/.certen/miner/config.yaml`

**Complete Configuration Example**:
```yaml
# Miner identification
miner_id: "miner-550e8400-e29b-41d4-a716-446655440000"

# Validator API endpoint
validator_url: "https://validator.certen.io"

# Network binding configuration
listen_addrs:
  - "/ip4/0.0.0.0/tcp/4001"
  - "/ip4/0.0.0.0/udp/4001/quic-v1"

# Bootstrap peers for network discovery
bootstrap_peers:
  - "/dns4/bootstrap1.certen.io/tcp/4001/p2p/12D3KooWBootstrap1..."
  - "/dns4/bootstrap2.certen.io/tcp/4001/p2p/12D3KooWBootstrap2..."
  - "/ip4/203.0.113.1/tcp/4001/p2p/12D3KooWBootstrap3..."

# Mining operation intervals
audit_interval: "5s"
heartbeat_interval: "30s"

# Data and key storage
data_dir: "~/.certen/miner"
identity_key: "~/.certen/miner/identity.key"

# Logging configuration
log_level: "info"  # debug, info, warn, error

# LXR algorithm parameters
lxr:
  table_bits: 25    # 32MB memory usage (production)
  loops: 8          # Translation passes
  passes: 6         # ByteMap randomization

# P2P network configuration
p2p:
  max_connections: 100
  connection_timeout: "30s"
  nat_traversal: true
  enable_relay: true
  enable_auto_relay: true

# Advanced networking
advanced:
  enable_mdns: true
  dht_bootstrap_interval: "15m"
  peer_score_threshold: -10
  message_signing: true
```

### Environment Variable Override

**Supported Environment Variables**:
```bash
# Core configuration
export CERTEN_MINER_ID="custom-miner-id"
export CERTEN_VALIDATOR_URL="https://custom-validator.example.com"
export CERTEN_DATA_DIR="/custom/data/path"
export CERTEN_LOG_LEVEL="debug"

# Network configuration
export CERTEN_LISTEN_ADDR="/ip4/0.0.0.0/tcp/5001"
export CERTEN_BOOTSTRAP_PEERS="peer1,peer2,peer3"

# Mining parameters
export CERTEN_AUDIT_INTERVAL="10s"
export CERTEN_LXR_TABLE_BITS="24"  # 16MB for resource-constrained systems

# Security
export CERTEN_IDENTITY_KEY="/secure/path/identity.key"
```

**Environment Variable Precedence**:
1. Command line flags (highest priority)
2. Environment variables
3. Configuration file
4. Default values (lowest priority)

### Configuration Validation

**Validation Process**:
```go
func ValidateConfig(cfg *Config) error {
    // Required fields validation
    if cfg.MinerID == "" {
        return errors.New("miner_id is required")
    }

    if cfg.ValidatorURL == "" {
        return errors.New("validator_url is required")
    }

    // Network validation
    for _, addr := range cfg.ListenAddrs {
        if _, err := multiaddr.NewMultiaddr(addr); err != nil {
            return fmt.Errorf("invalid listen address %s: %w", addr, err)
        }
    }

    // Bootstrap peers validation
    for _, peer := range cfg.BootstrapPeers {
        if _, err := multiaddr.NewMultiaddr(peer); err != nil {
            return fmt.Errorf("invalid bootstrap peer %s: %w", peer, err)
        }
    }

    // LXR parameters validation
    if cfg.LXR.TableBits < 10 || cfg.LXR.TableBits > 32 {
        return errors.New("lxr.table_bits must be between 10 and 32")
    }

    // Interval validation
    if cfg.AuditInterval < time.Second {
        return errors.New("audit_interval must be at least 1 second")
    }

    return nil
}
```

**Pre-flight Checks**:
```bash
# Use validate command before running
certen-miner validate

# Expected output:
# ✅ Configuration file readable
# ✅ Identity key accessible
# ✅ Validator connectivity
# ✅ Port binding capability
# ✅ Internet connectivity
```

## Identity and Key Management

### Identity Key Generation

**Initialization Process**:
```bash
# Generate configuration and identity
certen-miner init

# Custom key location
certen-miner init --identity-key /custom/path/identity.key

# Custom data directory
certen-miner init --data-dir /custom/miner/data
```

**Manual Key Generation**:
```go
package main

import (
    "crypto/rand"
    "github.com/libp2p/go-libp2p/core/crypto"
    "github.com/certenIO/miner/pkg/identity"
)

func generateIdentity(keyPath string) error {
    // Generate Ed25519 private key
    privKey, _, err := crypto.GenerateEd25519Key(rand.Reader)
    if err != nil {
        return err
    }

    // Save with strict permissions
    return identity.SavePrivateKey(privKey, keyPath)
}
```

**Key Security Best Practices**:

1. **File Permissions**: Keys stored with 0600 permissions (owner read/write only)
2. **Backup Strategy**: Secure backup of identity keys for disaster recovery
3. **Key Rotation**: Plan for periodic key rotation (future feature)
4. **Hardware Security**: Consider hardware security modules for high-value deployments

**Key Backup Process**:
```bash
# Backup identity key
cp ~/.certen/miner/identity.key ~/secure-backup/identity-$(date +%Y%m%d).key

# Verify backup
certen-miner validate --identity-key ~/secure-backup/identity-$(date +%Y%m%d).key

# Restore from backup
cp ~/secure-backup/identity-20241129.key ~/.certen/miner/identity.key
```

### Peer ID Management

**Peer ID Derivation**:
```bash
# Show peer ID from existing key
certen-miner status | grep "Peer ID"

# Verify peer ID determinism
certen-miner init --dry-run  # Shows what peer ID would be generated
```

**Peer ID Format**:
- **Encoding**: Base58 encoding of public key hash
- **Length**: 46-52 characters (e.g., `12D3KooWExample...`)
- **Uniqueness**: Cryptographically secure unique identifier
- **Routing**: Used as address in Kademlia DHT

## Network Configuration

### Listen Address Configuration

**Address Format Examples**:
```yaml
listen_addrs:
  # IPv4 TCP
  - "/ip4/0.0.0.0/tcp/4001"

  # IPv6 TCP
  - "/ip6/::/tcp/4001"

  # IPv4 QUIC
  - "/ip4/0.0.0.0/udp/4001/quic-v1"

  # IPv6 QUIC
  - "/ip6/::/udp/4001/quic-v1"

  # Specific interface
  - "/ip4/192.168.1.100/tcp/4001"

  # WebSocket (future)
  - "/ip4/0.0.0.0/tcp/8080/ws"
```

**Port Selection Guidelines**:
- **Default Ports**: 4001 (standard libp2p port)
- **Custom Ports**: Any available port above 1024
- **Firewall Rules**: Ensure selected ports are accessible
- **Port Ranges**: Use different ports for multiple miners on same host

### Firewall Configuration

**Required Outbound Access**:
```bash
# Validator API (HTTPS)
curl -I https://validator.certen.io/health

# Bootstrap peers (TCP/UDP)
nc -zv bootstrap.certen.io 4001

# DNS resolution
nslookup bootstrap.certen.io
```

**Inbound Port Requirements**:
```bash
# Test TCP port accessibility
nc -l 4001 &  # Start listener
nc -zv <your-public-ip> 4001  # Test from external host

# Test UDP port accessibility
nc -lu 4001 &  # Start UDP listener
nc -zu <your-public-ip> 4001  # Test from external host
```

**Firewall Rules (iptables)**:
```bash
# Allow inbound miner traffic
iptables -A INPUT -p tcp --dport 4001 -j ACCEPT
iptables -A INPUT -p udp --dport 4001 -j ACCEPT

# Allow outbound to validator and bootstrap peers
iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 4001 -j ACCEPT
iptables -A OUTPUT -p udp --dport 4001 -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

### NAT Traversal Configuration

**UPnP/NAT-PMP Support**:
```yaml
p2p:
  nat_traversal: true  # Enable automatic port mapping
  upnp_timeout: "30s"  # UPnP discovery timeout
```

**Manual Port Forwarding**:
```bash
# Router configuration (example)
# Forward external port 4001 → internal IP:4001 (TCP)
# Forward external port 4001 → internal IP:4001 (UDP)
```

**Relay Configuration**:
```yaml
p2p:
  enable_relay: true       # Use relays when direct connection fails
  enable_auto_relay: true  # Automatically discover relay peers
  relay_hop_timeout: "30s"
  max_relay_connections: 4
```

## Production Deployment

### Systemd Service Configuration

**Service File** (`/etc/systemd/system/certen-miner.service`):
```ini
[Unit]
Description=Certen Independent Miner
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=certen-miner
Group=certen-miner
WorkingDirectory=/opt/certen-miner
ExecStart=/opt/certen-miner/certen-miner run
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=certen-miner

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/certen-miner

# Resource limits
LimitNOFILE=65536
LimitNPROC=32768

# Environment
Environment=CERTEN_DATA_DIR=/var/lib/certen-miner
Environment=CERTEN_LOG_LEVEL=info

[Install]
WantedBy=multi-user.target
```

**Service Management**:
```bash
# Install service
sudo cp certen-miner.service /etc/systemd/system/
sudo systemctl daemon-reload

# Enable and start
sudo systemctl enable certen-miner
sudo systemctl start certen-miner

# Monitor logs
sudo journalctl -u certen-miner -f

# Check status
sudo systemctl status certen-miner
```

### Docker Deployment

**Dockerfile**:
```dockerfile
FROM golang:1.23-alpine AS builder

# Install dependencies
RUN apk add --no-cache git make

# Set working directory
WORKDIR /src

# Copy source code
COPY . .

# Build binary
RUN make build

# Production image
FROM alpine:latest

# Install runtime dependencies
RUN apk add --no-cache ca-certificates tzdata

# Create user
RUN addgroup -g 1001 certen && \
    adduser -u 1001 -G certen -s /bin/sh -D certen

# Create directories
RUN mkdir -p /app/data && \
    chown certen:certen /app/data

# Copy binary
COPY --from=builder /src/build/certen-miner /app/certen-miner

# Switch to user
USER certen

# Set working directory
WORKDIR /app

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD /app/certen-miner status || exit 1

# Default command
CMD ["./certen-miner", "run"]
```

**Docker Compose Configuration**:
```yaml
version: '3.8'

services:
  certen-miner:
    build: .
    restart: unless-stopped

    ports:
      - "4001:4001/tcp"
      - "4001:4001/udp"

    volumes:
      - miner-data:/app/data
      - ./config.yaml:/app/config.yaml:ro

    environment:
      - CERTEN_DATA_DIR=/app/data
      - CERTEN_LOG_LEVEL=info
      - CERTEN_VALIDATOR_URL=https://validator.certen.io

    healthcheck:
      test: ["CMD", "./certen-miner", "status"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"

volumes:
  miner-data:
    driver: local
```

**Container Deployment**:
```bash
# Build image
docker build -t certen-miner:latest .

# Run container
docker run -d \
  --name certen-miner \
  --restart unless-stopped \
  -p 4001:4001/tcp \
  -p 4001:4001/udp \
  -v miner-data:/app/data \
  -e CERTEN_VALIDATOR_URL=https://validator.certen.io \
  certen-miner:latest

# View logs
docker logs -f certen-miner

# Check status
docker exec certen-miner ./certen-miner status
```

### Kubernetes Deployment

**Deployment Manifest**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: certen-miner
  namespace: mining
spec:
  replicas: 1
  selector:
    matchLabels:
      app: certen-miner
  template:
    metadata:
      labels:
        app: certen-miner
    spec:
      containers:
      - name: certen-miner
        image: certen-miner:latest
        ports:
        - containerPort: 4001
          protocol: TCP
        - containerPort: 4001
          protocol: UDP

        env:
        - name: CERTEN_DATA_DIR
          value: "/data"
        - name: CERTEN_LOG_LEVEL
          value: "info"
        - name: CERTEN_VALIDATOR_URL
          value: "https://validator.certen.io"

        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /app/config.yaml
          subPath: config.yaml
          readOnly: true

        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "8Gi"
            cpu: "2000m"

        livenessProbe:
          exec:
            command: ["./certen-miner", "status"]
          initialDelaySeconds: 30
          periodSeconds: 30

        readinessProbe:
          exec:
            command: ["./certen-miner", "status"]
          initialDelaySeconds: 10
          periodSeconds: 10

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: certen-miner-data
      - name: config
        configMap:
          name: certen-miner-config

---
apiVersion: v1
kind: Service
metadata:
  name: certen-miner
  namespace: mining
spec:
  type: LoadBalancer
  ports:
  - port: 4001
    targetPort: 4001
    protocol: TCP
    name: tcp
  - port: 4001
    targetPort: 4001
    protocol: UDP
    name: udp
  selector:
    app: certen-miner

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: certen-miner-data
  namespace: mining
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd
```

## Monitoring and Logging

### Log Configuration

**Log Levels**:
- **debug**: Detailed debugging information, performance metrics
- **info**: General operational information, connections, mining progress
- **warn**: Warning conditions, recoverable errors
- **error**: Error conditions requiring attention

**Log Format**:
```
[TIMESTAMP] [LEVEL] [COMPONENT] MESSAGE
2024-11-29T10:15:30Z [INFO] [MinerNode] Connected to peer 12D3KooW...
2024-11-29T10:15:31Z [DEBUG] [LXR] Mining progress: 1000 attempts, nonce=1000
2024-11-29T10:15:32Z [INFO] [LXR] Solution found! nonce=1337, attempts=1337
```

**Structured Logging Setup**:
```go
import (
    "github.com/sirupsen/logrus"
    "github.com/certenIO/miner/pkg/config"
)

func SetupLogging(cfg *config.Config) {
    level, err := logrus.ParseLevel(cfg.LogLevel)
    if err != nil {
        level = logrus.InfoLevel
    }

    logrus.SetLevel(level)
    logrus.SetFormatter(&logrus.JSONFormatter{
        TimestampFormat: time.RFC3339Nano,
        FieldMap: logrus.FieldMap{
            logrus.FieldKeyTime: "timestamp",
            logrus.FieldKeyLevel: "level",
            logrus.FieldKeyMsg: "message",
        },
    })
}
```

### Performance Monitoring

**Built-in Metrics**:
```bash
# Real-time status
certen-miner status

# Output example:
Certen Miner Status
==================
Miner ID: miner-550e8400-e29b-41d4-a716-446655440000
Peer ID: 12D3KooWExample...
Status: Running
Uptime: 2h15m30s

Network:
- Connected Peers: 23
- Listen Addresses: [/ip4/192.168.1.100/tcp/4001]
- NAT Status: Public

Mining:
- Hash Rate: 1,250 H/s
- Audits Completed: 1,543
- Last Block: 125,432
- Success Rate: 98.5%

Validator:
- URL: https://validator.certen.io
- Latest Height: 125,433
- Last Response: 150ms
```

**Prometheus Metrics** (Future Enhancement):
```go
// Example metrics that could be exported
var (
    hashRateGauge = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "certen_miner_hash_rate",
            Help: "Current mining hash rate in hashes per second",
        },
        []string{"miner_id"},
    )

    auditsCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "certen_miner_audits_total",
            Help: "Total number of audits performed",
        },
        []string{"miner_id", "result"},
    )

    peerCountGauge = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "certen_miner_peers",
            Help: "Current number of connected peers",
        },
        []string{"miner_id"},
    )
)
```

### Health Checks

**Health Check Endpoint** (Future Enhancement):
```go
func (n *MinerNode) HealthCheck() *HealthStatus {
    return &HealthStatus{
        Status: "healthy",
        Checks: map[string]string{
            "p2p":       n.checkP2PHealth(),
            "validator": n.checkValidatorHealth(),
            "mining":    n.checkMiningHealth(),
            "storage":   n.checkStorageHealth(),
        },
        Timestamp: time.Now(),
    }
}
```

**Health Check Script**:
```bash
#!/bin/bash
# health-check.sh

set -e

# Check if miner process is running
if ! certen-miner status > /dev/null 2>&1; then
    echo "ERROR: Miner status check failed"
    exit 1
fi

# Check peer connectivity
PEER_COUNT=$(certen-miner status | grep "Connected Peers:" | awk '{print $3}')
if [ "$PEER_COUNT" -lt 5 ]; then
    echo "WARNING: Low peer count: $PEER_COUNT"
    exit 1
fi

# Check recent mining activity
LAST_BLOCK=$(certen-miner status | grep "Last Block:" | awk '{print $3}')
if [ -z "$LAST_BLOCK" ] || [ "$LAST_BLOCK" = "0" ]; then
    echo "ERROR: No mining activity detected"
    exit 1
fi

echo "OK: All health checks passed"
exit 0
```
