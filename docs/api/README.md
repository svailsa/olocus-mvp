# API Documentation

This directory contains API specifications and protocols for the Olocus system.

## Documents

### Priority 1 (Required before development)
- **[openapi.yaml](openapi.yaml)** - Complete OpenAPI 3.0 specification
- **[error_responses.md](error_responses.md)** - Standardized error format and codes

### Priority 2
- **[websocket_protocol.md](websocket_protocol.md)** - Real-time sync protocol for co-signing
- **[rate_limiting.md](rate_limiting.md)** - API throttling rules and quotas

## API Overview

### Endpoints Categories

#### Public Endpoints
- Health check
- Protocol version
- Public key exchange

#### Authenticated Endpoints
- Anchor submission
- Attestation requests
- Friendship management
- Marketplace operations

#### WebSocket Channels
- Real-time co-signing
- Attestation coordination
- Sync notifications

## Error Code Ranges
- `1xxx` - Friendship errors
- `2xxx` - Attestation errors
- `3xxx` - Device/security errors
- `4xxx` - Claim/nullifier errors
- `5xxx` - Anchor/blockchain errors
- `6xxx` - Network/sync errors

## Related Documentation
- [Protocol Specification](../specification/)
- [System Architecture](../architecture/system_architecture.md)
- [Security Checklist](../standards/security_checklist.md)