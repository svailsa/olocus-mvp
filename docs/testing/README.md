# Testing Documentation

This directory contains test plans, vectors, and methodologies for the Olocus Protocol.

## Documents

### Priority 1 (Required before development)
- **[test_vectors/](test_vectors/)** - Official test vectors for protocol operations

### Priority 2
- **[test_plan.md](test_plan.md)** - Complete test strategy and methodology
- **[battery_testing.md](battery_testing.md)** - Measurement methodology and benchmarks
- **[fraud_simulation.md](fraud_simulation.md)** - How to test anti-fraud measures
- **[performance_benchmarks.md](performance_benchmarks.md)** - CPU, memory, network targets

## Test Vector Categories

### Cryptographic Operations
- Hash chain generation
- Merkle tree construction
- Digital signatures
- ECDH key exchange

### Protocol Operations
- Visit detection (DBSCAN)
- Anchor creation
- Attestation flows
- Friendship establishment

### Data Structures
- Protocol Buffer serialization
- W3C Verifiable Credentials
- Error responses

## Testing Requirements

### Coverage Targets
- Core functions: 80% minimum
- Security functions: 100% required
- Protocol operations: 100% required

### Performance Targets
- Battery usage: 8-10% daily maximum
- Location accuracy: 92-95% verified
- Fraud rate: 5-8% maximum

## Related Documentation
- [Protocol Specification](../specification/)
- [Implementation Guide](../implementation/)
- [Security Checklist](../standards/security_checklist.md)