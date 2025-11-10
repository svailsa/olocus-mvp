# Test Vectors

Official test vectors for the Olocus Protocol v1.0.0

## Status: 🔴 Not Started

## Test Vector Files

### Priority 1 (Required before development)
- **[hash_chain_tests.json](hash_chain_tests.json)** - Hash chain generation and verification
- **[visit_detection_tests.json](visit_detection_tests.json)** - DBSCAN clustering test cases
- **[merkle_tree_tests.json](merkle_tree_tests.json)** - Merkle tree construction and proofs
- **[friendship_flow_tests.json](friendship_flow_tests.json)** - ECDH key exchange and friendship establishment

### Additional Test Vectors
- **[anchor_tests.json](anchor_tests.json)** - Daily anchor creation
- **[attestation_tests.json](attestation_tests.json)** - Attestation generation and verification
- **[credential_tests.json](credential_tests.json)** - W3C Verifiable Credential format
- **[protobuf_tests.json](protobuf_tests.json)** - Protocol Buffer serialization

## Test Vector Format

Each test vector file follows this structure:

```json
{
  "version": "1.0.0",
  "description": "Test vectors for [operation]",
  "vectors": [
    {
      "description": "Test case description",
      "input": {
        // Input data
      },
      "expected": {
        // Expected output
      },
      "metadata": {
        "category": "valid|invalid|edge_case",
        "platform": "all|ios|android|web"
      }
    }
  ]
}
```

## Validation

All implementations MUST pass these test vectors to be considered compliant with the Olocus Protocol v1.0.0.

## References
- [Protocol Specification](../../specification/olocus_protocol_specification.md)
- [Implementation Guide](../../implementation/olocus_protocol_implementation_guide.md)