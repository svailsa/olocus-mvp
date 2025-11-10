# Claude Development Assistant Configuration

## Project Context

**Project**: Olocus Protocol MVP  
**Type**: Privacy-preserving location tracking and attestation system  
**Architecture**: Distributed, on-device processing with cryptographic proofs  
**License**: Apache 2.0  

## Key Project Files

### Core Documentation
- `docs/overview.md` - Business and technical overview
- `docs/olocus_protocol_specification.md` - Complete protocol specification v1.0
- `docs/olocus_protocol_implementation_guide.md` - Implementation best practices
- `docs/olocus_protocol_release_summary.md` - Release notes and next steps

### Task Tracking
- `TASKS.md` - Documentation task tracker
- `README.md` - Project overview and setup instructions

## Technical Requirements

### Protocol Implementation
- **Passive tracking**: 95% of operations, continuous location recording
- **Active operations**: 5% of operations, user-initiated attestations
- **Battery budget**: 8-10% daily usage maximum
- **Accuracy target**: 92-95% for verified location claims
- **Fraud tolerance**: 5-8% expected fraud rate

### Core Technologies
- **Cryptography**: SHA-256, Ed25519, ECDH Curve25519
- **Serialization**: Protocol Buffers v3
- **Standards**: W3C Verifiable Credentials, RFC 3161, GeoJSON
- **Platforms**: iOS (Swift), Android (Kotlin), Web (JavaScript)

### Architecture Patterns
- **On-device storage**: All raw location data encrypted locally
- **Hash chains**: Tamper-evident append-only logs
- **Merkle trees**: Efficient proof generation
- **Zero-knowledge proofs**: Phase 2+ for selective disclosure
- **Batch processing**: Attestations grouped for efficiency

## Development Guidelines

### Code Style
- **Swift**: Follow Swift API Design Guidelines
- **Kotlin**: Follow Kotlin Coding Conventions
- **JavaScript**: ESLint with Airbnb config
- **General**: Clear variable names, comprehensive error handling

### Security Requirements
- **Never** transmit raw location coordinates (only commitments)
- **Always** encrypt sensitive data at rest (AES-256-GCM)
- **Zeroize** ephemeral keys and shared secrets after use
- **Validate** all signatures before processing
- **Hash** device fingerprint fields for privacy

### Testing Requirements
- **Unit test coverage**: Minimum 80% for core functions
- **Integration tests**: All protocol flows
- **Test vectors**: Must pass official test vectors
- **Battery testing**: Must stay within 8-10% budget
- **Fraud simulation**: Test anti-fraud measures

## OwnTracks Pattern Adoption

### Patterns to Adopt (Reimplemented)
1. **Motion-adaptive sampling**: Adjust frequency based on activity
2. **Battery preservation modes**: Tiered degradation under low battery
3. **Background task resilience**: BGTaskScheduler (iOS), WorkManager (Android)
4. **Permission UX flow**: Progressive disclosure, clear value prop
5. **Significant location changes**: Coarse tracking as fallback

### Patterns to Avoid
- Direct code copying (license incompatibility)
- Variable names or structure (maintain independence)
- GPS-only approach (use fused providers)
- Continuous high-accuracy (battery killer)

## Common Commands

### Protocol Buffer Generation
```bash
protoc --swift_out=. --kotlin_out=. --js_out=. proto/v1/*.proto
```

### Run Tests
```bash
# iOS
xcodebuild test -scheme OlocusKit

# Android  
./gradlew test

# JavaScript
npm test
```

### Check Battery Usage
```bash
# iOS Instruments
instruments -t "Energy Log" OlocusApp

# Android Battery Historian
adb bugreport > bugreport.txt
```

## Key Decisions Made

1. **GitHub first, Codeberg later**: Development on GitHub, move to Codeberg post-MVP
2. **Apache 2.0 everywhere**: No GPL code, full commercial flexibility
3. **OwnTracks-inspired, not copied**: Study patterns, reimplement independently
4. **Documentation-first**: Complete specs before coding begins
5. **Battery over accuracy**: Accept 85-90% accuracy to maintain battery budget
6. **Progressive deployment**: Universities → Fitness → Mainstream

## Frequently Needed Information

### Error Codes
- 1xxx: Friendship errors
- 2xxx: Attestation errors  
- 3xxx: Device/security errors
- 4xxx: Claim/nullifier errors
- 5xxx: Anchor/blockchain errors
- 6xxx: Network/sync errors

### Battery Cost Estimates
- GPS high accuracy: 0.05% per sample
- GPS balanced: 0.02% per sample
- WiFi scan: 0.01% per scan
- Hash computation: 0.0001% per hash
- ZKP generation: 3-4% daily (Phase 2+)

### Trust Score Weights
- Close friend: 1.0
- Acquaintance: 0.7
- Colleague: 0.5
- Device tampered: -0.5 penalty
- Low accuracy: -0.2 penalty

## Current Priorities

1. Complete database schema documentation
2. Create OpenAPI specification
3. Define test vectors for all operations
4. Document platform-specific setup guides
5. Create OwnTracks pattern analysis

## Notes for Future Sessions

- Remember battery budget is critical (8-10% daily max)
- Privacy is paramount (no raw locations transmitted)
- Documentation completeness gates development start
- OwnTracks is inspiration only (no code copying)
- Focus on MVP features first (Phase 1 only)

---

*This file helps Claude maintain context across sessions. Update it as key decisions are made or priorities change.*