# Architecture Documentation

This directory contains system architecture and design documents for the Olocus Protocol.

## Documents

### Priority 1 (Required before development)
- **[system_architecture.md](system_architecture.md)** - High-level system design with component diagrams
- **[database_schema.md](database_schema.md)** - Complete SQLite/IndexedDB schemas with migrations
- **[security_architecture.md](security_architecture.md)** - Security layers, threat model, and mitigations
- **[api_design.md](api_design.md)** - REST/WebSocket patterns and endpoint structure

### Priority 2
- **[data_flow_diagrams.md](data_flow_diagrams.md)** - Visual flow of data through passive/active protocols

## Key Architectural Decisions

### Distributed Architecture
- On-device storage and processing (95% of operations)
- Server coordination only (5% of operations)
- No centralized location database

### Privacy by Design
- Raw location data never leaves device
- Only cryptographic commitments transmitted
- Zero-knowledge proofs for selective disclosure (Phase 2+)

### Battery Efficiency
- Target: 8-10% daily battery usage
- Motion-adaptive sampling
- Smart scheduling and batching

### Security Layers
1. Device-level encryption (AES-256-GCM)
2. Hash chain integrity
3. Digital signatures (Ed25519)
4. Blockchain anchoring
5. Zero-knowledge proofs (Phase 2+)

## Related Documentation
- [Protocol Specification](../specification/)
- [Security Standards](../standards/security_checklist.md)
- [Implementation Guide](../implementation/)