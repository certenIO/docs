# Certen Miner Documentation

Comprehensive technical docs for the Certen Miners, designed to enable devs to understand, contribute to, and extend the system with current implementation fidelity

### Documentation Structure

| Document | Purpose | Target Audience |
|----------|---------|-----------------|
| [Architecture](architecture.md) | System design, package structure, data flow | All developers, architects |
| [Algorithms](algorithms.md) | LXR proof-of-work, cryptography, performance | Core developers, cryptographers |
| [Networking](networking.md) | LibP2P integration, P2P protocols, security | Network engineers, protocol developers |
| [Setup & Deployment](setup-deployment.md) | Installation, configuration, production deployment | DevOps, system administrators |
| [Development Guide](development-guide.md) | Code organization, testing, CI/CD workflows | Contributors, maintainers |

## Quick Reference

### Core Components

**CLI Interface** (`pkg/cli/`):
- **init**: Generate identity keys and configuration
- **run**: Start mining operations with P2P networking
- **status**: Display operational metrics and network state
- **validate**: Test environment and connectivity

**Mining Engine** (`pkg/lxr/`):
- **LXR Algorithm**: Memory-hard proof-of-work (from Accumulate)
- **Difficulty Scaling**: Height-based difficulty adjustment
- **Performance**: Configurable memory usage (1MB-1GB)

**P2P Networking** (`pkg/node/`):
- **LibP2P**: Full peer-to-peer networking stack
- **Discovery**: DHT, mDNS, bootstrap peers
- **Security**: Ed25519 identity, Noise protocol encryption
- **Messaging**: GossipSub for audit attestations and heartbeats

**Proof Verification** (`pkg/verifier/`):
- **Validator API**: HTTP client for proof bundle fetching
- **Caching**: In-memory LRU cache for performance
- **Validation**: Structural and semantic proof verification

### Key Algorithms

**LXR Proof-of-Work**:
1. **ByteMap Generation**: Randomized lookup table creation
2. **Challenge Mixing**: XorShift-based hash preparation
3. **Translation**: Memory-intensive computation with random access
4. **Difficulty Validation**: Leading 0xFF byte counting

**P2P Discovery**:
1. **Bootstrap**: Connect to known seed peers
2. **DHT Population**: Kademlia routing table building
3. **Peer Scoring**: Reputation-based connection management
4. **Local Discovery**: mDNS for same-network peers

### Configuration

**Default Locations**:
- Configuration: `~/.certen/miner/config.yaml`
- Identity Key: `~/.certen/miner/identity.key`
- Data Directory: `~/.certen/miner/`

**Key Parameters**:
```yaml
validator_url: "https://validator.certen.io"
audit_interval: "5s"
lxr:
  table_bits: 25  # 32MB memory usage
  loops: 8
  passes: 6
```

### Network Protocols

**Transport Layer**:
- **QUIC**: Primary transport (low latency, built-in encryption)
- **TCP**: Fallback transport (firewall compatibility)
- **WebSocket**: Future browser integration

**Application Layer**:
- **GossipSub Topics**: `certen/audit/v1`, `certen/heartbeat/v1`
- **Protocol Buffers**: Efficient binary serialization
- **Message Signing**: Ed25519 signature verification

## Architecture Decision Records

### ADR-001: LXR Algorithm Selection

**Decision**: Use Accumulate's LXR proof-of-work algorithm directly
**Rationale**:
- Proven security properties and ASIC resistance
- Existing implementation available
- Memory-hard characteristics align with decentralization goals
**Consequences**:
- Compatible with Accumulate ecosystem
- Requires significant memory for high security levels
- Performance depends on memory bandwidth

### ADR-002: LibP2P Networking Stack

**Decision**: Build on LibP2P for all networking functionality
**Rationale**:
- Industry-standard P2P networking library
- Rich feature set (DHT, GossipSub, security protocols)
- Active development and security auditing
**Consequences**:
- Large dependency tree
- Learning curve for LibP2P concepts
- Excellent long-term maintainability

### ADR-003: Protocol Buffer Messaging

**Decision**: Use Protocol Buffers for P2P message serialization
**Rationale**:
- Efficient binary encoding
- Schema evolution support
- Cross-language compatibility
**Consequences**:
- Additional build step for code generation
- Compact message size
- Type-safe message handling

### ADR-004: Ed25519 Identity System

**Decision**: Use Ed25519 for cryptographic identity
**Rationale**:
- LibP2P standard identity format
- High performance signature operations
- Proven security properties
**Consequences**:
- Modern cryptography with good security margins
- Quantum vulnerability (future consideration)
- Deterministic peer ID generation

## Security Considerations

### Threat Model

**Network Attacks**:
- **Eclipse Attacks**: Mitigated by multiple bootstrap peers and DHT diversity
- **Sybil Attacks**: Reputation scoring and connection limits
- **Message Flooding**: Rate limiting and peer reputation

**Cryptographic Security**:
- **Identity Forgery**: Ed25519 signature verification prevents impersonation
- **Transport Security**: Noise protocol provides forward secrecy
- **Message Integrity**: AEAD authentication prevents tampering

**Implementation Security**:
- **Input Validation**: Comprehensive validation of all external inputs
- **Memory Safety**: Go's memory safety prevents buffer overflows
- **Dependency Security**: Regular security auditing of dependencies

### Operational Security

**Key Management**:
- Private keys stored with restrictive file permissions (0600)
- No logging or network transmission of private key material
- Secure random number generation for key creation

**Network Security**:
- Default deny firewall policies with explicit allow rules
- TLS for validator API communications
- Encrypted P2P transport for all peer communications

## Performance Characteristics

### Resource Requirements

**Memory Usage**:
- **Base Runtime**: ~100MB for application and dependencies
- **LXR Table**: 1MB-1GB configurable (default 32MB)
- **P2P Buffers**: ~10MB for connection management and message queuing

**CPU Usage**:
- **Mining**: Single-threaded LXR computation (primary workload)
- **Networking**: Multi-threaded P2P message processing
- **Background**: Periodic DHT maintenance and health checks

**Network Usage**:
- **Audit Messages**: ~1KB per attestation (every 5 seconds default)
- **Heartbeats**: ~0.5KB per heartbeat (every 30 seconds default)
- **DHT Traffic**: ~10KB/hour for routing table maintenance
- **Validator API**: ~10KB per proof bundle fetch (every 5 seconds)

### Scalability Considerations

**Horizontal Scaling**:
- Multiple miners can run independently without coordination
- Network capacity scales with number of participants
- Validator load distributes across multiple endpoints

**Vertical Scaling**:
- Larger LXR tables increase memory requirement and security
- Faster CPUs improve hash rate linearly
- Network bandwidth affects P2P participation quality

## Testing Strategy

### Test Coverage

**Unit Tests**: Individual function and method testing
- **Target Coverage**: >90% for critical paths
- **Mock Usage**: Interfaces enable comprehensive mocking
- **Edge Cases**: Extensive boundary condition testing

**Integration Tests**: Component interaction testing
- **Network Integration**: Multi-node P2P testing
- **API Integration**: Live validator API testing
- **Protocol Integration**: End-to-end message flow testing

**Performance Tests**: Benchmark and load testing
- **Algorithm Benchmarks**: LXR performance across different parameters
- **Network Load Testing**: P2P performance under high message volumes
- **Memory Usage Testing**: Resource consumption measurement

## Contributing

### Development Setup

1. **Clone Repository**: `git clone https://github.com/certenIO/miner.git`
2. **Install Dependencies**: `make deps`
3. **Run Tests**: `make test`
4. **Build Binary**: `make build`

### Code Standards

- **Go Version**: 1.23 or higher required
- **Code Style**: Enforced via `gofumpt` and `golangci-lint`
- **Documentation**: All public APIs must be documented
- **Testing**: New features require comprehensive test coverage

### Submission Process

1. **Fork Repository**: Create personal fork on GitHub
2. **Feature Branch**: Create feature branch from `develop`
3. **Implementation**: Follow coding standards and write tests
4. **Pull Request**: Submit PR with detailed description
5. **Code Review**: Address reviewer feedback
6. **Merge**: Maintainer merges after approval

## Maintenance and Support

### Release Process

1. **Version Tagging**: Semantic versioning (v1.2.3)
2. **Cross-Compilation**: Build for all supported platforms
3. **Security Review**: Security team review for each release
4. **Documentation Update**: Keep docs synchronized with code

### Monitoring and Observability

**Health Checks**:
- Built-in status command for operational verification
- Environment validation for deployment readiness
- Connectivity testing for network functionality

**Logging**:
- Structured logging with configurable levels
- Performance metrics and hash rate reporting
- Security events and error tracking

**Future Enhancements**:
- Prometheus metrics export
- OpenTelemetry distributed tracing
- Grafana dashboard templates
