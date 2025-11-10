# Olocus Protocol Specification v1.0

**Status**: Draft  
**Version**: 1.0.0  
**Last Updated**: 2024-11-08  
**Authors**: Mark Harper

---

## Abstract

The Olocus Protocol defines a privacy-preserving location tracking and attestation system that enables users to maintain verifiable location histories while retaining full control over data disclosure. The protocol combines continuous passive tracking (95% of operations) with user-initiated active operations (5% of operations) to create a comprehensive location data infrastructure.

**Core Capabilities**:
- **Passive Tracking**: Continuous location recording with cryptographic commitments
- **Active Attestation**: Social proof through peer verification and batch processing
- **Privacy Preservation**: Zero raw data transmission, user-controlled selective disclosure
- **Monetization**: Verifiable credentials for location data marketplace

**Key Properties**:
- Tamper-evident hash chain architecture
- W3C Verifiable Credentials integration
- Zero-knowledge proof roadmap
- Cross-platform consistency guarantees
- Fraud-resistant device fingerprinting

---

## Document Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## Table of Contents

### Part I: Foundation
0. [Core Definitions & Dependencies](#section-0-core-definitions--dependencies)

### Part II: Passive Protocol (Continuous Tracking)
1. [Private Hash Chain Data Structure](#section-1-private-hash-chain-data-structure)
2. [Visit Detection & Aggregation](#section-2-visit-detection--aggregation)
3. [Anchoring & Timestamping](#section-3-anchoring--timestamping)
4. [Location Credentials & Selective Disclosure](#section-4-location-credentials--selective-disclosure)

### Part III: Active Protocol (User-Initiated Operations)
5. [Friendship Establishment Protocol](#section-5-friendship-establishment-protocol)
6. [Attestation Protocol](#section-6-attestation-protocol)
7. [Batch Attestation Processing](#section-7-batch-attestation-processing)
8. [Claim Verification & Trust Scoring](#section-8-claim-verification--trust-scoring)
9. [Nullifier Registry](#section-9-nullifier-registry)
10. [Marketplace Integration](#section-10-marketplace-integration)

### Part IV: System-Wide Concerns
11. [Data Retention & Security](#section-11-data-retention--security)
12. [Protocol Versioning & Extensibility](#section-12-protocol-versioning--extensibility)
13. [Security Considerations](#section-13-security-considerations)
14. [Interoperability Requirements](#section-14-interoperability-requirements)

### Appendices
- [Appendix A: Error Code Registry](#appendix-a-error-code-registry)
- [Appendix B: Configuration Schema](#appendix-b-configuration-schema)
- [Appendix C: Compatibility Matrix](#appendix-c-compatibility-matrix)
- [Appendix D: Privacy Impact Assessment](#appendix-d-privacy-impact-assessment)
- [Appendix E: References](#appendix-e-references)
- [Appendix F: Changelog](#appendix-f-changelog)

---

# PART I: FOUNDATION

## Section 0: Core Definitions & Dependencies

### 0.1 Referenced Standards

This specification normatively references the following standards:

| Standard | Reference | Purpose |
|----------|-----------|---------|
| **SHA-256** | FIPS 180-4, RFC 6234 | Cryptographic hashing |
| **Ed25519** | RFC 8032 | Digital signatures |
| **ECDH Curve25519** | RFC 6090, SEC1 | Key agreement for shared secrets |
| **Protocol Buffers v3** | https://protobuf.dev | Data serialization |
| **GeoJSON** | RFC 7946 | Geographic coordinates |
| **JWT/JWS** | RFC 7519, RFC 7515 | Token structure |
| **RFC 3161** | RFC 3161 | Trusted timestamping |
| **W3C Verifiable Credentials** | W3C VC v1.1 | Claims format |
| **DID Core** | W3C DID v1.0 | Decentralized identifiers |
| **AES-256-GCM** | NIST SP 800-38D | Data encryption |
| **PBKDF2** | RFC 8018 | Key derivation |
| **OAuth 2.0** | RFC 6749 | Marketplace authentication |

### 0.2 Cryptographic Primitives

#### 0.2.1 Hash Function

**Algorithm**: SHA-256  
**Output Size**: 32 bytes (256 bits)  
**Notation**: `H(x)` where x is input data

**Serialization for Hashing**:
- All data MUST be serialized using Protocol Buffers canonical encoding before hashing
- Field ordering MUST follow protobuf field numbers in ascending order
- Zero values MUST be included (no field omission)

#### 0.2.2 Digital Signature Scheme

**Algorithm**: Ed25519 (EdDSA over Curve25519)  
**Public Key Size**: 32 bytes  
**Private Key Size**: 64 bytes (32-byte seed + 32-byte derived)  
**Signature Size**: 64 bytes  
**Notation**: `Sign_SK(m)` for signing, `Verify_PK(m, sig)` for verification

#### 0.2.3 Key Agreement (ECDH)

**Algorithm**: X25519 (ECDH over Curve25519)  
**Public Key Size**: 32 bytes  
**Private Key Size**: 32 bytes  
**Shared Secret Size**: 32 bytes  
**Notation**: `X25519(SK_a, PK_b)` produces shared secret

**Security Requirements**:
- Shared secrets MUST be zeroized immediately after use
- Ephemeral keys MUST be destroyed after shared secret derivation
- Shared secrets MUST NEVER be transmitted or logged

#### 0.2.4 Random Number Generation

**Source**: Cryptographically Secure Pseudo-Random Number Generator (CSPRNG)  
**Reference**: NIST SP 800-90A (CTR_DRBG)

**Platform-Specific Implementations**:
- iOS: `SecRandomCopyBytes`
- Android: `java.security.SecureRandom`
- Web: `crypto.getRandomValues`

### 0.3 Type System

All data structures MUST use the following primitive types with explicit serialization:

```protobuf
// olocus_common.proto

syntax = "proto3";

package olocus.v1;

// Import path for use in other .proto files:
// import "olocus/v1/common.proto";

// ============================================================================
// Common Primitive Types
// ============================================================================

message Hash {
  bytes value = 1; // MUST be exactly 32 bytes (SHA-256 output)
}

message Signature {
  bytes value = 1; // MUST be exactly 64 bytes (Ed25519 signature)
}

message Timestamp {
  uint64 unix_seconds = 1; // Seconds since Unix epoch (UTC)
  uint32 nanos = 2;        // Nanosecond offset [0, 999999999]
}

message PublicKey {
  bytes ed25519 = 1;    // MUST be exactly 32 bytes (signing key)
  bytes curve25519 = 2; // MUST be exactly 32 bytes (ECDH key, optional)
}

// ============================================================================
// Geographic Types
// ============================================================================

message GeoCoordinates {
  double longitude = 1; // Decimal degrees, range [-180, 180], WGS84
  double latitude = 2;  // Decimal degrees, range [-90, 90], WGS84
  double altitude = 3;  // Meters above WGS84 ellipsoid (optional)
}

message GeoAccuracy {
  double horizontal = 1; // Meters, 95% confidence radius
  double vertical = 2;   // Meters, 95% confidence (optional)
}

// ============================================================================
// Error Handling
// ============================================================================

message Error {
  uint32 code = 1;              // See Appendix A: Error Code Registry
  string message = 2;           // Technical error message
  string user_friendly = 3;     // User-facing message
  repeated string details = 4;  // Additional context
  Timestamp occurred_at = 5;
}

// ============================================================================
// Configuration Types
// ============================================================================

message BatchConfig {
  uint32 max_batch_size = 1;        // Default: 20
  uint32 min_batch_threshold = 2;   // Default: 3
  uint64 batch_window_ms = 3;       // Default: 300000 (5 minutes)
}

message ClaimConfig {
  uint32 min_attestations = 1;          // K in K-of-N, default: 3
  uint32 attestation_expiry_hours = 2;  // Default: 168 (7 days)
}

message SamplingPolicy {
  uint32 interval_seconds = 1;    // Sampling interval
  string accuracy = 2;            // "high" | "balanced" | "low"
  uint32 min_displacement = 3;    // Meters
}
```

### 0.4 Geographic Coordinate System

**Standard**: WGS84 (EPSG:4326)  
**Reference**: GeoJSON RFC 7946

All geographic coordinates MUST use the WGS84 datum with decimal degrees.

### 0.5 Time Source

**Primary Source**: Network Time Protocol (NTP)  
**Reference**: RFC 5905  
**NTP Pool**: pool.ntp.org or regional equivalent  
**Fallback**: Device system time  
**Accuracy Requirement**: ±5 seconds for anchoring operations

**Implementation Requirements**:
- Devices MUST synchronize with NTP at least once per 24 hours
- Devices MUST record time uncertainty in metadata
- Anchoring operations MUST fail if time uncertainty exceeds 60 seconds

### 0.6 User Identity

#### 0.6.1 Decentralized Identifier (DID)

**Format**: `did:olocus:<user_id_hash>`  
**Standard**: W3C DID Core v1.0

```protobuf
message UserIdentity {
  string did = 1;             // Format: "did:olocus:" + base58(H(PubKey_User))
  PublicKey public_key = 2;   // Ed25519 public key
  Timestamp created_at = 3;   // DID creation timestamp
}
```

**DID Generation**:
```
1. Generate Ed25519 keypair (SK_user, PK_user)
2. Compute user_id_hash = H(PK_user)
3. Encode user_id_hash as base58
4. DID = "did:olocus:" + base58(user_id_hash)
```

#### 0.6.2 Key Management

**Storage Requirements**:
- Private keys MUST be stored in device secure enclave (iOS) or Keystore (Android)
- Private keys MUST NOT be exportable in plaintext
- Backup MUST use BIP-39 mnemonic phrases (24 words)

### 0.7 Encryption

**Algorithm**: AES-256-GCM  
**Reference**: NIST SP 800-38D  
**Key Size**: 256 bits  
**IV/Nonce Size**: 96 bits (12 bytes)  
**Authentication Tag Size**: 128 bits (16 bytes)

**Key Derivation**:
```protobuf
message KeyDerivation {
  string algorithm = 1;   // MUST be "PBKDF2-SHA256"
  uint32 iterations = 2;  // MUST be >= 100,000
  bytes salt = 3;         // MUST be 16 bytes from CSPRNG
  bytes derived_key = 4;  // 32 bytes (256 bits)
}
```

---

# PART II: PASSIVE PROTOCOL (Continuous Tracking)

## Section 1: Private Hash Chain Data Structure

### 1.1 Hash Chain Architecture

Each user MUST maintain a private, append-only hash chain of location data.

```protobuf
message LocationHashChain {
  Hash chain_id = 1;
  GenesisBlock genesis_block = 2;
  Hash current_head = 3;              // Hash of most recent block
  Timestamp last_anchor_timestamp = 4;
  Hash last_anchor_block_hash = 5;
  uint32 protocol_version = 6;        // MUST be 1 for this spec
}

message GenesisBlock {
  Timestamp created_at = 1;
  Hash user_commitment = 2;         // H(user_did || genesis_timestamp)
  DeviceFingerprint device_fingerprint = 3;
}

message DeviceFingerprint {
  string os = 1;                    // "iOS" | "Android" | "Web"
  Hash os_version_hash = 2;         // H(os_version) for privacy
  Hash device_model_hash = 3;       // H(device_model) for privacy
  string app_version = 4;           // Olocus app version
  Hash installation_id = 5;         // H(platform_id || installation_timestamp)
}
```

**Chain ID Generation**:
```
chain_id = H(PK_user || installation_id || genesis_timestamp)
  where:
    PK_user: 32-byte Ed25519 public key
    installation_id: H(platform_id || installation_timestamp)
    genesis_timestamp: Unix timestamp at chain creation
    ||: Concatenation operator
```

**Genesis Block Generation** (IMPROVED):
```
1. Generate user_commitment = H(user_did || genesis_timestamp)
2. Collect device_fingerprint with hashed OS/model fields
3. Create GenesisBlock with commitment and fingerprint
4. genesis_hash = H(GenesisBlock)
```

### 1.2 Location Block Structure

A **LocationBlock** is the atomic unit of the hash chain, containing a single location sample with metadata.

```protobuf
message LocationBlock {
  // Header (metadata about block structure)
  LocationBlockHeader header = 1;
  
  // Payload (actual location data)
  LocationBlockPayload payload = 2;
  
  // Signature (JWS-style)
  Signature signature = 3; // Sign(SK_user, H(header || payload))
}

message LocationBlockHeader {
  string type = 1;      // MUST be "OLOCUS_LOCATION_BLOCK"
  string algorithm = 2; // MUST be "SHA256"
  string version = 3;   // MUST be "1.0"
}

message LocationBlockPayload {
  uint64 block_index = 1;      // Sequential counter, starts at 0
  Timestamp timestamp = 2;      // When sample was taken
  GeoCoordinates coordinates = 3;
  GeoAccuracy accuracy = 4;
  Hash previous_hash = 5;       // H(previous LocationBlock)
  
  // Optional motion data
  MotionState motion_state = 6;
  double speed = 7;             // Meters per second (optional)
  double bearing = 8;           // Degrees from true north, 0-360 (optional)
  
  // Device state
  DeviceState device_state = 9;
}

enum MotionState {
  MOTION_UNKNOWN = 0;
  MOTION_STATIONARY = 1;
  MOTION_WALKING = 2;
  MOTION_RUNNING = 3;
  MOTION_CYCLING = 4;
  MOTION_VEHICLE = 5;
}

message DeviceState {
  uint32 battery_level = 1;  // Percentage [0, 100]
  bool low_power_mode = 2;
  string network_type = 3;    // "wifi" | "cellular" | "offline"
  uint32 time_uncertainty_ms = 4; // Milliseconds
}
```

**Block Hash Computation**:
```
block_hash = H(serialize(header) || serialize(payload))
  where:
    serialize() uses Protocol Buffers canonical encoding
    H() is SHA-256
    || is byte concatenation
```
```

### 1.3 Hash Chain Extension Algorithm

**Block Creation**:
```
create_location_block(coordinates, timestamp, motion_state):
  1. Retrieve current_head from chain
  2. Increment block_index
  3. Construct LocationBlockPayload:
     - Set block_index, timestamp, coordinates, accuracy
     - Set previous_hash = current_head
     - Include motion_state, device_state
  4. Construct LocationBlockHeader (type, algorithm, version)
  5. Compute block_hash = H(header || payload)
  6. Sign block: signature = Sign(SK_user, block_hash)
  7. Assemble LocationBlock(header, payload, signature)
  8. Update chain: current_head = block_hash
  9. Store encrypted block in local database
  10. Return LocationBlock
```

### 1.4 Chain Integrity Verification

```
verify_chain_segment(blocks):
  1. For each block in blocks:
     a. Verify signature: Verify(PK_user, block_hash, block.signature)
     b. Verify previous_hash links to prior block
     c. Verify block_index is sequential
     d. Verify timestamp is monotonically increasing
  2. If all checks pass: return VALID
  3. Else: return INVALID with error details
```

---

## Section 2: Visit Detection & Aggregation

### 2.1 Visit Definition

A **LocusVisit** represents an aggregated stay at a specific location, derived from multiple LocationBlocks.

```protobuf
message LocusVisit {
  Hash visit_id = 1;                // H(random_256_bits)
  GeoCoordinates center = 2;        // Centroid of visit cluster
  double radius_meters = 3;         // 95% confidence radius
  
  // Temporal bounds
  Timestamp arrival_time = 4;
  Timestamp departure_time = 5;
  uint64 duration_seconds = 6;      // departure - arrival
  
  // Visit characteristics
  VisitType visit_type = 7;
  double confidence = 8;            // [0.0, 1.0]
  
  // Block references
  repeated Hash block_hashes = 9;   // Hashes of constituent LocationBlocks
  Hash blocks_merkle_root = 10;     // Merkle root of block_hashes
  
  // Privacy-preserving commitments
  Hash visit_commitment = 11;       // H(visit_id || center || arrival_time)
  
  uint32 protocol_version = 12;     // MUST be 1
}

enum VisitType {
  VISIT_UNKNOWN = 0;
  VISIT_HOME = 1;
  VISIT_WORK = 2;
  VISIT_TRANSIT = 3;
  VISIT_LEISURE = 4;
  VISIT_SHOPPING = 5;
  VISIT_DINING = 6;
  VISIT_OTHER = 7;
}
```

**Merkle Tree Construction Rules**:

The `blocks_merkle_root` MUST be computed using the following algorithm:

```
compute_merkle_root(block_hashes):
  1. Sort leaves by block_index (ascending order)
  2. If list is empty: return H("EMPTY_TREE")
  3. If list has single element: return H(element)
  4. Create leaf layer: leaves = [H(block_hash) for block_hash in block_hashes]
  5. While len(leaves) > 1:
     a. If len(leaves) is odd: duplicate last leaf
     b. Create parent layer: []
     c. For each pair (left, right):
        - parent = H(left || right)  // Canonical byte concatenation
        - Append parent to parent layer
     d. leaves = parent layer
  6. Return leaves[0] as merkle_root
```

**Verification**:
- Leaf ordering MUST be deterministic (by block_index)
- Hash function MUST be SHA-256
- Concatenation MUST be direct byte concatenation (no separators)
- Empty trees MUST use `H("EMPTY_TREE")` sentinel value
```

### 2.2 Visit Detection Algorithm

**DBSCAN-Based Clustering** (RECOMMENDED):

```
detect_visits(location_blocks, epsilon=50, min_samples=3):
  1. Extract (lat, lon, timestamp) from each block
  2. Compute distance matrix using Haversine formula
  3. Apply DBSCAN clustering with:
     - epsilon: max distance between points (default 50m)
     - min_samples: minimum cluster size (default 3)
  4. For each cluster:
     a. Compute centroid (center of mass)
     b. Compute radius (max distance from centroid)
     c. Identify arrival_time (first block timestamp)
     d. Identify departure_time (last block timestamp)
     e. Calculate duration = departure - arrival
     f. Classify visit_type using heuristics
     g. Assign confidence score based on cluster quality
  5. Generate visit_id = H(random_256_bits)
  6. Compute blocks_merkle_root from constituent block_hashes
  7. Compute visit_commitment = H(visit_id || center || arrival_time)
  8. Construct LocusVisit message
  9. Return list of LocusVisits
```

### 2.3 Visit Type Classification

**Rule-Based Heuristics**:

```
classify_visit_type(visit, historical_visits):
  # Home: Frequent nighttime visits (10pm - 6am)
  if is_frequent_location(visit.center, historical_visits) and 
     overlaps_nighttime(visit.arrival_time, visit.departure_time):
    return VISIT_HOME with confidence = 0.9
  
  # Work: Frequent daytime visits on weekdays (9am - 5pm)
  if is_frequent_location(visit.center, historical_visits) and
     is_weekday(visit.arrival_time) and
     overlaps_work_hours(visit.arrival_time, visit.departure_time):
    return VISIT_WORK with confidence = 0.85
  
  # Transit: Short duration (< 30 minutes), high speed before/after
  if visit.duration_seconds < 1800 and
     high_speed_adjacent_blocks(visit.block_hashes):
    return VISIT_TRANSIT with confidence = 0.7
  
  # Dining: 30-120 minute duration, dinner hours (6pm - 10pm)
  if 1800 <= visit.duration_seconds <= 7200 and
     overlaps_dinner_hours(visit.arrival_time, visit.departure_time):
    return VISIT_DINING with confidence = 0.6
  
  # Default
  return VISIT_OTHER with confidence = 0.5
```

### 2.4 Visit Storage

```protobuf
message EncryptedVisit {
  bytes ciphertext = 1;             // AES-256-GCM encrypted LocusVisit
  bytes nonce = 2;                  // 12-byte nonce (unique per visit)
  bytes authentication_tag = 3;     // 16-byte auth tag
  string key_id = 4;                // Reference to encryption key
  Hash visit_commitment = 5;        // Stored unencrypted for indexing
}
```

---

## Section 3: Anchoring & Timestamping

### 3.1 Daily Anchor Structure

```protobuf
message DailyAnchor {
  Hash anchor_id = 1;
  Hash chain_head_hash = 2;         // Hash of last block in 24h period
  Hash visits_merkle_root = 3;      // Merkle root of all visits in period
  Timestamp period_start = 4;       // Start of 24h period (UTC midnight)
  Timestamp period_end = 5;         // End of 24h period
  
  // Timestamping
  bytes tsa_token = 6;              // RFC 3161 timestamp token
  string tsa_url = 7;               // TSA server URL
  
  // Blockchain reference (Phase 1: optional, Phase 2+: required)
  BlockchainReference blockchain_ref = 8;
  
  // Anchor signature
  Signature anchor_signature = 9;   // Sign(SK_user, H(anchor))
  
  uint32 protocol_version = 10;     // MUST be 1
}

message BlockchainReference {
  string chain_id = 1;              // "ethereum" | "polkadot" | "near"
  string transaction_hash = 2;
  uint64 block_number = 3;
  Timestamp block_timestamp = 4;
  bytes anchor_data = 5;            // Data committed on-chain
}
```

**Anchor Submission Window**:
- DailyAnchors MUST be created within 48 hours of `period_end`
- Anchors submitted after 48 hours SHOULD be rejected by validators
- This prevents retroactive anchoring attacks

**Anchor Frequency**:
- One anchor per 24-hour period (UTC-aligned)
- `period_start` MUST be UTC midnight (00:00:00)
- `period_end` MUST be next UTC midnight (24:00:00)
```

### 3.2 Anchoring Process (Phase 1: RFC 3161 TSA)

```
create_daily_anchor(chain, visits, period_start, period_end):
  1. Collect all LocationBlocks created in [period_start, period_end]
  2. Verify chain integrity for collected blocks
  3. Extract chain_head_hash = last block's hash
  4. Collect all LocusVisits in time period
  5. Compute visits_merkle_root from visit_commitment values
  6. Construct DailyAnchor:
     - anchor_id = H(random_256_bits)
     - chain_head_hash, visits_merkle_root
     - period_start, period_end
  7. Generate RFC 3161 timestamp token:
     a. Compute anchor_hash = H(DailyAnchor without tsa_token)
     b. Send TSA request to tsa_url with anchor_hash
     c. Receive tsa_token (signed timestamp)
     d. Verify tsa_token signature
  8. (Optional Phase 1) Publish to blockchain:
     a. Construct blockchain transaction with anchor_hash
     b. Broadcast transaction
     c. Store blockchain_ref when confirmed
  9. Sign anchor: anchor_signature = Sign(SK_user, H(DailyAnchor))
  10. Store DailyAnchor (unencrypted, contains no raw coordinates)
  11. Return DailyAnchor
```

### 3.3 Anchor Verification

```
verify_daily_anchor(anchor):
  1. Verify anchor.anchor_signature using user's public key
  2. Verify RFC 3161 tsa_token:
     a. Extract timestamp from token
     b. Verify TSA signature
     c. Verify timestamp matches period
  3. If blockchain_ref exists:
     a. Query blockchain for transaction
     b. Verify anchor_hash matches on-chain data
     c. Verify block_timestamp is within acceptable window
  4. Return verification result
```

---

## Section 4: Location Credentials & Selective Disclosure

### 4.1 Location Credential Structure (W3C Verifiable Credential)

```protobuf
message LocationCredential {
  // W3C VC Standard Fields
  string context = 1;               // MUST be "https://www.w3.org/2018/credentials/v1"
  string type = 2;                  // MUST include "VerifiableCredential", "LocationCredential"
  string issuer_did = 3;            // User's DID (self-issued)
  Timestamp issuance_date = 4;
  Timestamp expiration_date = 5;    // Optional
  
  // Credential Subject (Location Claim)
  LocationClaim credential_subject = 6;
  
  // Proof
  CredentialProof proof = 7;
  
  // Anchor Reference
  AnchorReference anchor = 8;
  
  uint32 protocol_version = 9;      // MUST be 1
}

message LocationClaim {
  string subject_did = 1;           // Claimant's DID
  Hash visit_id = 2;                // References LocusVisit
  
  // Selective disclosure fields (oneof)
  oneof spatial_disclosure {
    GeoCoordinates exact_coordinates = 3;
    Hash coordinates_commitment = 4; // H(coordinates)
    SpatialProof spatial_proof = 5;  // ZK proof (Phase 2+)
  }
  
  // Temporal claims (always revealed)
  Timestamp visit_start = 6;
  Timestamp visit_end = 7;
  uint64 duration_seconds = 8;
  
  // Visit characteristics
  VisitType visit_type = 9;
  double confidence = 10;
  
  // Optional: Place information
  string place_name = 11;           // Optional, user-provided
  string place_category = 12;       // Optional, e.g., "restaurant"
}

message CredentialProof {
  string type = 1;                  // MUST be "Ed25519Signature2020"
  Timestamp created = 2;
  string verification_method = 3;   // DID URL to public key
  bytes proof_value = 4;            // Signature bytes
}

message AnchorReference {
  Hash anchor_id = 1;               // References DailyAnchor
  Hash chain_head_hash = 2;
  Hash visits_merkle_root = 3;
  MerkleProof merkle_proof = 4;     // Proves visit inclusion in anchor
  BlockchainReference blockchain_ref = 5; // Optional in Phase 1
}

message MerkleProof {
  Hash leaf = 1;                    // visit_commitment
  repeated Hash path = 2;           // Sibling hashes from leaf to root
  repeated bool directions = 3;     // true = right, false = left
}
```

### 4.2 Spatial Proof (Zero-Knowledge, Phase 2+)

```protobuf
message SpatialProof {
  oneof proof_type {
    ProximityProof proximity = 1;
    RegionMembershipProof region = 2;
  }
}

message ProximityProof {
  GeoCoordinates reference_point = 1;
  double max_distance_meters = 2;
  bytes zkp_data = 3;               // Zero-knowledge proof
  repeated Hash public_inputs = 4;
  string circuit_id = 5;            // e.g., "proximity_v1"
}

message RegionMembershipProof {
  string region_id = 1;             // e.g., "US-CA-SF", "postal_code_94102"
  bytes zkp_data = 2;
  repeated Hash public_inputs = 3;
  string circuit_id = 4;            // e.g., "region_v1"
}
```

### 4.3 Credential Generation Algorithm

```
create_location_credential(visit, disclosure_level):
  1. Verify visit is anchored (exists in DailyAnchor)
  2. Construct LocationClaim:
     - subject_did = current user's DID
     - visit_id = visit.visit_id
     - If disclosure_level == "exact":
         Use visit.center as exact_coordinates
       Else if disclosure_level == "commitment":
         Use H(visit.center) as coordinates_commitment
       Else if disclosure_level == "zkp" (Phase 2+):
         Generate spatial_proof
     - Include temporal claims: visit_start, visit_end, duration
     - Include visit_type, confidence
  3. Build anchor reference:
     - Find DailyAnchor covering visit time window
     - Include chain_head_hash, visits_merkle_root
     - Generate merkle_proof proving visit inclusion
  4. Construct CredentialProof:
     - proof_value = Sign(SK_user, H(credential))
  5. Assemble LocationCredential
  6. Return LocationCredential
```

### 4.4 Credential Verification

```
verify_location_credential(credential):
  1. Verify W3C VC structure compliance
  2. Verify credential.proof.proof_value using issuer's public key
  3. Verify issuance_date <= current_time < expiration_date
  4. Verify anchor exists on blockchain (if blockchain_ref present):
     - Query blockchain for anchor at credential.anchor.blockchain_ref
     - Verify anchor.visits_merkle_root matches credential
  5. Verify merkle_proof:
     - Compute leaf = H(visit_commitment)
     - Verify merkle_path leads to visits_merkle_root
  6. If spatial_proof exists (Phase 2+), verify ZK proof
  7. Return verification result
```

### 4.5 Nullifier-Based Double-Claim Prevention

```protobuf
message ClaimNullifier {
  Hash nullifier = 1;               // H(visit_id || user_did || salt)
  Timestamp claimed_at = 2;
  string marketplace = 3;           // Where claim was published
  Hash claim_id = 4;
  string claimant_did = 5;
}

message NullifierRegistry {
  repeated ClaimNullifier nullifiers = 1;
}
```

**Double-Claim Prevention** (IMPROVED):
```
can_create_claim(visit_id, user_did, user_salt):
  nullifier = H(visit_id || user_did || user_salt)
  return !NullifierRegistry.contains(nullifier)

publish_claim(credential, user_salt):
  nullifier = H(credential.visit_id || credential.issuer_did || user_salt)
  NullifierRegistry.add(nullifier, current_timestamp, marketplace_id)
```

---

# PART III: ACTIVE PROTOCOL (User-Initiated Operations)

## Section 5: Friendship Establishment Protocol

### 5.1 Friendship Credential Structure

```protobuf
message FriendshipCredential {
  Hash credential_id = 1;           // H(random_256_bits)
  
  // Participants (ordered: requester first, acceptor second)
  string participant_a_did = 2;     // Requester's DID
  string participant_b_did = 3;     // Acceptor's DID
  
  // Shared secret commitment (NEVER the secret itself)
  Hash shared_secret_commitment = 4; // H(shared_secret)
  
  // Establishment metadata
  Timestamp established_at = 5;
  Timestamp expires_at = 6;         // MAY be null for indefinite
  
  // Optional: Relationship type
  FriendshipType type = 7;
  
  // Mutual signatures
  Signature participant_a_signature = 8;
  Signature participant_b_signature = 9;
  
  uint32 protocol_version = 10;     // MUST be 1
}

enum FriendshipType {
  FRIENDSHIP_UNSPECIFIED = 0;
  FRIENDSHIP_CLOSE = 1;             // High trust
  FRIENDSHIP_ACQUAINTANCE = 2;      // Medium trust
  FRIENDSHIP_COLLEAGUE = 3;         // Professional context
}
```

### 5.2 Friendship Establishment Process

**Phase 1: Request Initiation**

```protobuf
message FriendshipRequest {
  Hash request_id = 1;
  string requester_did = 2;
  string target_did = 3;
  
  // Requester's ECDH public key (ephemeral)
  PublicKey ephemeral_public_key = 4;
  
  // Request metadata
  Timestamp created_at = 5;
  Timestamp expires_at = 6;         // Request valid for 7 days default
  FriendshipType proposed_type = 7;
  
  // Optional: Out-of-band verification code
  string verification_code = 8;     // 8-digit code + checksum for in-person verification
  
  // Request signature
  Signature requester_signature = 9; // Sign(SK_requester, H(request))
  
  uint32 protocol_version = 10;     // MUST be 1
}
```

**Phase 2: Request Acceptance**

```protobuf
message FriendshipResponse {
  Hash request_id = 1;              // References original request
  bool accepted = 2;
  
  // Acceptor's ECDH public key (ephemeral, only if accepted)
  PublicKey ephemeral_public_key = 3;
  
  // Response metadata
  Timestamp responded_at = 4;
  
  // Response signature
  Signature acceptor_signature = 5; // Sign(SK_acceptor, H(response))
  
  uint32 protocol_version = 6;      // MUST be 1
}
```

### 5.3 Shared Secret Derivation (IMPROVED)

**Requirements**:
1. Both parties MUST compute shared secret using ECDH
2. Shared secret MUST be derived from ephemeral keys (not long-term DIDs)
3. Shared secret MUST NEVER be transmitted or stored on servers
4. Only commitment H(shared_secret) MAY be transmitted
5. **Shared secret MUST be zeroized immediately after deriving commitment**
6. **Ephemeral private keys MUST be zeroized after shared secret derivation**

**Derivation Algorithm**:
```
1. Requester generates ephemeral keypair (SK_r_eph, PK_r_eph)
2. Requester sends PK_r_eph in FriendshipRequest
3. Acceptor generates ephemeral keypair (SK_a_eph, PK_a_eph)
4. Acceptor sends PK_a_eph in FriendshipResponse
5. Both compute: shared_secret = X25519(SK_own_eph, PK_other_eph)
6. Both compute: commitment = H(shared_secret)
7. Both store FriendshipCredential with commitment (not secret)
8. CRITICAL: zeroize(shared_secret)  // MUST
9. CRITICAL: zeroize(SK_eph)         // MUST
```

### 5.4 Friendship Validation Rules

A valid FriendshipCredential MUST satisfy:

1. **Mutual Consent**: Both participant_a_signature and participant_b_signature MUST be valid
2. **Temporal Validity**: established_at < current_time < expires_at (if expires_at is set)
3. **Commitment Integrity**: shared_secret_commitment MUST equal H(shared_secret)
4. **Uniqueness**: credential_id MUST be globally unique
5. **Ordering**: participant_a_did < participant_b_did (lexicographic)

---

## Section 6: Attestation Protocol

### 6.1 Attestation Request Structure

```protobuf
message AttestationRequest {
  Hash request_id = 1;
  
  // Claim being attested
  Hash claim_id = 2;                // References LocationCredential
  string claimant_did = 3;
  
  // Requesting attestation from
  string attester_did = 4;
  
  // Spatial-temporal requirements
  AttestationRequirements requirements = 5;
  
  // Request metadata
  Timestamp created_at = 6;
  Timestamp expires_at = 7;         // Default: 7 days
  
  // Request signature
  Signature claimant_signature = 8;
  
  uint32 protocol_version = 9;      // MUST be 1
}

message AttestationRequirements {
  double max_distance_meters = 1;   // MUST be > 0
  uint64 min_time_overlap_seconds = 2; // MUST be > 0
  bool require_friendship = 3;      // If true, MUST have FriendshipCredential
}
```

### 6.2 Individual Attestation Structure

```protobuf
message Attestation {
  Hash attestation_id = 1;
  Hash request_id = 2;              // References AttestationRequest
  Hash claim_id = 3;
  
  // Attester information
  string attester_did = 4;
  Hash attester_visit_id = 5;       // References attester's LocusVisit
  Hash attester_visit_commitment = 6; // H(LocusVisit)
  
  // Friendship proof
  FriendshipProof friendship_proof = 7;
  
  // Spatial-temporal proof
  SpatialTemporalProof overlap_proof = 8;
  
  // Device attestation
  DeviceFingerprint device_fingerprint = 9;
  
  // Attestation metadata
  Timestamp created_at = 10;
  
  // Attestation signature
  Signature attester_signature = 11; // Sign(SK_attester, H(attestation))
  
  uint32 protocol_version = 12;     // MUST be 1
}
```

### 6.3 Friendship Proof

```protobuf
message FriendshipProof {
  oneof proof_type {
    FriendshipCommitmentProof commitment_proof = 1; // Phase 1
    FriendshipZKProof zk_proof = 2;                 // Phase 2+
  }
}

message FriendshipCommitmentProof {
  Hash shared_secret_commitment = 1; // H(shared_secret)
  Hash credential_id = 2;            // References FriendshipCredential
  
  // Optional: Server-assisted proof (Phase 1)
  ServerFriendshipProof server_proof = 3;
}

message ServerFriendshipProof {
  string server_url = 1;
  bytes proof_data = 2;             // Server-generated proof
  Timestamp proof_timestamp = 3;
  Signature server_signature = 4;
}

message FriendshipZKProof {
  bytes zkp_data = 1;               // Circom/snarkjs proof
  repeated Hash public_inputs = 2;
  string circuit_id = 3;            // e.g., "friendship_v2"
}
```

### 6.4 Spatial-Temporal Proof

```protobuf
message SpatialTemporalProof {
  oneof proof_type {
    SpatialTemporalCommitment commitment = 1; // Commitment-based
    SpatialTemporalZKP zkp = 2;               // ZK proof (Phase 2+)
  }
}

message SpatialTemporalCommitment {
  // Attester's visit bounds (revealed)
  GeoCoordinates attester_center = 1;
  double attester_radius = 2;
  Timestamp attester_arrival = 3;
  Timestamp attester_departure = 4;
  
  // Computed overlap
  double distance_meters = 5;       // Distance between centers
  uint64 time_overlap_seconds = 6;  // Seconds of temporal overlap
  
  // Commitment to attester's visit
  Hash attester_visit_commitment = 7;
}

message SpatialTemporalZKP {
  bytes zkp_data = 1;
  repeated Hash public_inputs = 2;
  string circuit_id = 3;            // e.g., "colocation_v1"
}
```

### 6.5 Attestation Validation

```
validate_attestation(attestation, claim):
  1. Verify attestation.attester_signature
  2. Verify attestation.request_id references valid AttestationRequest
  3. Verify request has not expired
  4. If requirements.require_friendship:
     a. Verify friendship_proof
     b. Verify shared_secret_commitment matches stored credential
  5. Verify overlap_proof:
     a. If commitment-based:
        - Verify distance_meters <= max_distance_meters
        - Verify time_overlap_seconds >= min_time_overlap_seconds
     b. If ZKP-based:
        - Verify ZK proof validates
  6. Check device_fingerprint for tampering flags
  7. Return validation result
```

---

## Section 7: Batch Attestation Processing

### 7.1 Batch Attestation Structure

```protobuf
message BatchAttestation {
  Hash batch_id = 1;
  string attester_did = 2;
  
  // Batch of attestations
  repeated AttestationBatchEntry entries = 3;
  
  // Merkle tree of attestation_ids
  Hash attestations_merkle_root = 4;
  
  // Batch metadata
  Timestamp created_at = 5;
  uint32 batch_size = 6;
  
  // Batch signature (single signature covers all entries)
  Signature batch_signature = 7;
  
  uint32 protocol_version = 8;      // MUST be 1
}

message AttestationBatchEntry {
  Hash attestation_id = 1;
  Hash claim_id = 2;
  string claimant_did = 3;
  Hash attester_visit_commitment = 4;
  SpatialTemporalProof overlap_proof = 5;
  
  // Merkle proof for this entry
  MerkleProof merkle_proof = 6;
}
```

**Batch Size Constraints**:
- Batch size MUST be between `config.min_batch_threshold` and `config.max_batch_size`
- Batch size MUST NOT exceed 50 attestations (hard limit for network efficiency)
- If more than 50 attestations are pending, create multiple batches

**Batch Window**:
- Batches SHOULD be created within `config.batch_window_ms` (default: 5 minutes)
- Implementations MAY flush earlier if `max_batch_size` is reached
```

### 7.2 Batch Creation Algorithm

```
create_batch_attestation(attestations, config):
  1. Verify len(attestations) >= config.min_batch_threshold
  2. Verify len(attestations) <= config.max_batch_size
  3. Generate batch_id = H(random_256_bits)
  4. For each attestation:
     a. Create AttestationBatchEntry
     b. Include attestation_id, claim_id, claimant_did
     c. Include attester_visit_commitment, overlap_proof
  5. Construct Merkle tree from attestation_ids
  6. Compute attestations_merkle_root
  7. Generate merkle_proof for each entry
  8. Sign batch: batch_signature = Sign(SK_attester, H(batch))
  9. Assemble BatchAttestation
  10. Return BatchAttestation
```

### 7.3 Batch Verification

```
verify_batch_attestation(batch):
  1. Verify batch.batch_signature using attester's public key
  2. For each entry in batch.entries:
     a. Verify merkle_proof leads to attestations_merkle_root
     b. Verify overlap_proof (spatial-temporal validation)
  3. Return batch verification result
```

---

## Section 8: Claim Verification & Trust Scoring

### 8.1 Verified Claim Structure

```protobuf
message VerifiedClaim {
  // Base claim
  LocationCredential base_claim = 1;
  
  // Attestations
  repeated Attestation attestations = 2;
  
  // Optional: Batch attestations
  repeated BatchAttestation batch_attestations = 3;
  
  // Trust scoring
  TrustScore trust_score = 4;
  
  // Verification status
  VerificationStatus status = 5;
  
  Timestamp verified_at = 6;
  uint32 protocol_version = 7;      // MUST be 1
}

enum VerificationStatus {
  VERIFICATION_UNKNOWN = 0;
  VERIFICATION_PENDING = 1;
  VERIFICATION_INSUFFICIENT = 2;    // Not enough attestations
  VERIFICATION_SUFFICIENT = 3;      // Meets minimum threshold
  VERIFICATION_STRONG = 4;          // Exceeds threshold significantly
  VERIFICATION_SUSPICIOUS = 5;      // Fraud flags detected
}
```

### 8.2 Trust Scoring Algorithm (CONFIGURABLE)

```protobuf
message TrustScore {
  double aggregate_score = 1;       // [0.0, 1.0]
  uint32 attestation_count = 2;
  repeated AttestationWeight weights = 3;
  TrustWeightConfig config = 4;
}

message AttestationWeight {
  Hash attestation_id = 1;
  double weight = 2;                // [0.0, 1.0]
  repeated string penalty_reasons = 3;
}

message TrustWeightConfig {
  double close_friend_weight = 1;   // Default: 1.0
  double acquaintance_weight = 2;   // Default: 0.7
  double colleague_weight = 3;      // Default: 0.5
  double device_tampered_penalty = 4; // Default: -0.5
  double low_accuracy_penalty = 5;  // Default: -0.2
}
```

**Trust Score Computation Formula**:

The aggregate trust score MUST be computed using the following formula:

```
TrustScore = (Σ(friendship_weight_i × attestation_i) 
           - (tamper_penalty × device_tampered_count)
           - (accuracy_penalty × low_accuracy_count)) / attestation_count

where:
  friendship_weight_i = weight based on friendship type (close/acquaintance/colleague)
  attestation_i = 1 if attestation is valid, 0 otherwise
  device_tampered_count = number of attestations from tampered devices
  low_accuracy_count = number of attestations with distance > 100m
  attestation_count = total number of attestations
  
Bounds: TrustScore ∈ [0.0, 1.0] (clamped)
```

**Scoring Algorithm**:
```
compute_trust_score(claim, attestations, config):
  weights = []
  for attestation in attestations:
    weight = 1.0
    
    # Friendship type modifier
    if attestation.friendship_proof.friendship_type == CLOSE:
      weight *= config.close_friend_weight
    elif attestation.friendship_proof.friendship_type == ACQUAINTANCE:
      weight *= config.acquaintance_weight
    elif attestation.friendship_proof.friendship_type == COLLEAGUE:
      weight *= config.colleague_weight
    
    # Device trust penalty
    if attestation.device_fingerprint.is_tampered:
      weight += config.device_tampered_penalty
      penalty_reasons.append("device_tampered")
    
    # Spatial accuracy penalty
    if attestation.overlap_proof.distance_meters > 100:
      weight += config.low_accuracy_penalty
      penalty_reasons.append("low_spatial_accuracy")
    
    # Time overlap bonus
    if attestation.overlap_proof.time_overlap_seconds > 3600:
      weight *= 1.2
    
    weights.append(AttestationWeight(attestation.id, weight, penalty_reasons))
  
  aggregate_score = sum(w.weight for w in weights) / len(weights)
  return TrustScore(aggregate_score, len(attestations), weights, config)
```

### 8.3 Verification Algorithm

```
verify_claim(claim, attestations, config):
  1. Verify claim.base_claim.proof (W3C VC validation)
  2. Verify claim links to anchored LocusVisit (check merkle_proof)
  3. For each attestation:
     a. Verify attestation.attester_signature
     b. Verify attestation.friendship_proof
     c. Verify attestation.overlap_proof
     d. Check attestation.device_fingerprint for fraud flags
     e. Compute attestation_weight
  4. Compute aggregate_trust_score = weighted_sum(attestation_weights)
  5. Determine status:
     if aggregate_trust_score >= config.strong_threshold:
       status = VERIFICATION_STRONG
     elif aggregate_trust_score >= config.sufficient_threshold:
       status = VERIFICATION_SUFFICIENT
     else:
       status = VERIFICATION_INSUFFICIENT
  6. Check nullifier registry for double-claim
  7. Construct VerifiedClaim
  8. Return VerifiedClaim
```

---

## Section 9: Nullifier Registry

### 9.1 Nullifier Structure

```protobuf
message ClaimNullifier {
  Hash nullifier = 1;               // H(visit_id || claimant_did || salt)
  Hash claim_id = 2;
  string claimant_did = 3;
  Timestamp claimed_at = 4;
  string marketplace = 5;           // Where claim was published
  Hash transaction_id = 6;          // If on-chain
}
```

### 9.2 Double-Claim Prevention

**Requirements**:

1. Before publishing a claim, implementation MUST:
   - Compute nullifier = H(visit_id || claimant_did || user_salt)
   - Query NullifierRegistry for existing nullifier
   - If exists: REJECT with error "CLAIM_ALREADY_PUBLISHED"

2. When publishing a claim, implementation MUST:
   - Add nullifier to NullifierRegistry
   - Record timestamp and marketplace

3. Nullifiers MUST be globally unique across all marketplaces

**Nullifier Registry Interface**:
```protobuf
service NullifierRegistry {
  rpc CheckNullifier(NullifierCheckRequest) returns (NullifierCheckResponse);
  rpc RegisterNullifier(NullifierRegistration) returns (NullifierRegistrationResponse);
}

message NullifierCheckRequest {
  Hash nullifier = 1;
}

message NullifierCheckResponse {
  bool exists = 1;
  ClaimNullifier existing_record = 2; // If exists=true
}

message NullifierRegistration {
  ClaimNullifier nullifier = 1;
  Signature registrant_signature = 2;
}

message NullifierRegistrationResponse {
  bool success = 1;
  string error_message = 2;         // If success=false
}
```

---

## Section 10: Marketplace Integration

### 10.1 Marketplace Listing

```protobuf
message MarketplaceListing {
  Hash listing_id = 1;
  VerifiedClaim claim = 2;
  
  // Pricing
  uint64 price_usd_cents = 3;
  string accepted_currencies = 4;   // e.g., "OLOCUS_TOKEN,USDC,ETH"
  
  // Listing metadata
  Timestamp listed_at = 5;
  Timestamp expires_at = 6;
  string marketplace_url = 7;
  
  // Selective disclosure options
  bool reveal_exact_coordinates = 8;
  bool reveal_place_name = 9;
  
  // Listing signature
  Signature seller_signature = 10;
  
  uint32 protocol_version = 11;     // MUST be 1
}
```

### 10.2 Purchase Transaction

```protobuf
message PurchaseTransaction {
  Hash transaction_id = 1;
  Hash listing_id = 2;
  
  // Buyer information
  string buyer_did = 3;
  
  // Payment
  uint64 amount_paid_usd_cents = 4;
  string payment_currency = 5;
  string payment_tx_id = 6;         // Blockchain transaction ID
  
  // Data delivery
  bytes encrypted_claim_data = 7;   // Encrypted with buyer's public key
  
  // Transaction metadata
  Timestamp purchased_at = 8;
  
  // Signatures
  Signature buyer_signature = 9;
  Signature seller_signature = 10;
  
  uint32 protocol_version = 11;     // MUST be 1
}
```

---

# PART IV: SYSTEM-WIDE CONCERNS

## Section 11: Data Retention & Security

### 11.1 Encryption Requirements

**At-Rest Encryption**:
- All LocationBlocks MUST be encrypted using AES-256-GCM
- All LocusVisits MUST be encrypted using AES-256-GCM
- All FriendshipCredentials MUST encrypt shared_secret using device-bound key
- DailyAnchors MAY be stored unencrypted (contain no raw coordinates)

```protobuf
message EncryptedLocationBlock {
  bytes ciphertext = 1;             // AES-256-GCM encrypted LocationBlock
  bytes nonce = 2;                  // 12-byte nonce (MUST be unique per block)
  bytes authentication_tag = 3;     // 16-byte authentication tag
  string key_id = 4;                // Reference to encryption key in keychain
}
```

**Key Derivation**:
```
master_key = device_secure_enclave.generate_key()
encryption_key = PBKDF2(
  password = master_key,
  salt = installation_id || "OLOCUS_ENCRYPTION",
  iterations = 100000,
  output_length = 32
)
```

### 11.2 Data Retention Policy

```protobuf
message RetentionPolicy {
  uint32 raw_blocks_days = 1;       // Default: 90 days
  uint32 visits_days = 2;           // Default: indefinite (0 = forever)
  uint32 anchors_days = 3;          // Default: indefinite
  bool auto_prune_enabled = 4;
  Timestamp last_prune_timestamp = 5;
}
```

**Pruning Rules**:
1. LocationBlocks older than `raw_blocks_days` MAY be deleted
2. Before deleting blocks, implementation MUST verify:
   - Blocks are included in a LocusVisit (visits_merkle_root preserved)
   - Corresponding DailyAnchor exists on blockchain
3. LocusVisits MUST be retained if not yet claimed or within expiration window
4. DailyAnchors MUST never be pruned (needed for verification)

### 11.3 Device Compromise Mitigation

**Remote Revocation**:
```protobuf
message DeviceRevocation {
  Hash chain_id = 1;                // Chain to revoke
  Timestamp revoked_at = 2;
  string reason = 3;                // "device_lost" | "device_stolen" | "compromised"
  Signature user_signature = 4;     // Signed by user's key (from backup device)
  Hash nullifier = 5;               // Broadcast to prevent future anchors
}
```

**Recovery Procedure**:
1. User reports device compromise from backup device
2. User signs DeviceRevocation with backup key
3. Revocation broadcast to all TSAs / blockchain nodes
4. Nullifier added to global revocation list
5. Future anchors from compromised chain_id rejected
6. User restores from encrypted backup on new device

**Backup Requirements**:
- Users MUST maintain encrypted backups of:
  - BIP-39 seed phrase (for key recovery)
  - Latest DailyAnchor (for chain continuity proof)
- Backups MAY be stored in user-controlled cloud storage (iCloud, Google Drive)
- Backups MUST be encrypted with user-derived password

---

## Section 12: Protocol Versioning & Extensibility

### 12.1 Semantic Versioning

Protocol versions follow Semantic Versioning 2.0.0:
- **MAJOR**: Breaking changes (incompatible with previous version)
- **MINOR**: Backwards-compatible additions
- **PATCH**: Backwards-compatible bug fixes

Current Version: **1.0.0**

### 12.2 Version Negotiation

```protobuf
message ProtocolVersion {
  uint32 major = 1;
  uint32 minor = 2;
  uint32 patch = 3;
}
```

**Compatibility Rules**:
- Verifiers MUST accept any version with same MAJOR version
- Verifiers MAY reject blocks with unknown MAJOR version
- Verifiers SHOULD ignore unknown fields (forward compatibility)

### 12.3 Extension Mechanism

```protobuf
message ExtensionField {
  string extension_id = 1;          // e.g., "olocus.neural_interface.v1"
  bytes extension_data = 2;         // Arbitrary protobuf message
}
```

Future protocol versions MAY add:
- New MotionState enum values (e.g., MOTION_FLYING, MOTION_CYCLING)
- New VisitType enum values
- New device state fields (e.g., neural interface status)
- New cryptographic primitives (e.g., post-quantum signatures)

---

## Section 13: Security Considerations

### 13.1 Threat Model

**Protected Against**:
- ✅ Location data interception (encrypted at rest)
- ✅ Unauthorized location disclosure (user-controlled claims)
- ✅ Timestamp manipulation (RFC 3161 TSA / blockchain immutability)
- ✅ Double-spending location claims (nullifier registry with salted hashes)
- ✅ Chain history tampering (hash chain immutability)
- ✅ Shared secret exposure (zeroization after use)

**Not Protected Against**:
- ❌ Device compromise with biometric unlock (physical access)
- ❌ Malicious applications with location permissions (OS-level threat)
- ❌ Side-channel attacks on device (out of scope)
- ❌ Coercion to reveal data (user cooperation assumed)

### 13.2 Privacy Analysis

**Privacy Guarantees**:
- Location coordinates never transmitted (only commitments)
- Visit aggregation hides granular movement patterns
- Selective disclosure allows minimal information sharing
- Nullifiers use unlinkable identifiers (salted hashes)
- Device fingerprints use hashed OS/model fields

**Privacy Limitations**:
- DailyAnchors reveal user is actively using protocol (metadata leak)
- Merkle proof size reveals approximate number of visits (information leak)
- Repeated claims from same user may enable profiling

### 13.3 Cryptographic Security

Implementations MUST:

1. **Key Security**:
   - Store private keys in device secure enclave/keystore
   - Never expose private keys via APIs
   - Rotate ephemeral keys after each use
   - Zeroize shared secrets immediately after use

2. **Commitment Security**:
   - Use SHA-256 for all commitments
   - Never reveal preimage unless explicitly authorized
   - Validate commitments before accepting

3. **Signature Security**:
   - Verify all signatures before processing
   - Reject messages with invalid signatures
   - Use nonces to prevent replay attacks

### 13.4 Privacy Requirements

Implementations MUST:

1. **Data Minimization**:
   - Transmit only commitments, never raw data
   - Use selective disclosure for claims
   - Minimize metadata leakage

2. **Social Graph Privacy**:
   - Phase 1: Friendship edges visible to server (acceptable)
   - Phase 2+: Use ZKPs to hide friendship graph
   - Never expose friend lists publicly

3. **Location Privacy**:
   - Never transmit raw coordinates except in purchased claims
   - Use ZKPs for spatial proofs when possible
   - Allow users to control disclosure granularity

### 13.5 Fraud Detection Requirements

Implementations MUST:

1. **Device Fingerprinting**:
   - Collect fingerprints at all critical operations
   - Hash non-essential fields (OS version, device model)
   - Detect tampered devices (jailbreak/root)
   - Flag suspicious device patterns

2. **Behavioral Analysis**:
   - Track co-location patterns
   - Detect impossible movement
   - Analyze friendship graph topology

3. **Trust Scoring**:
   - Weight attestations by device trust
   - Penalize flagged devices
   - Provide transparency on trust calculations

### 13.6 Audit Requirements

Implementations SHOULD:
- Undergo annual third-party security audits
- Publish cryptographic test vectors for interoperability
- Maintain public bug bounty programs
- Document all known vulnerabilities

---

## Section 14: Interoperability Requirements

### 14.1 Cross-Platform Consistency

All implementations MUST produce identical outputs for:
- Hash computations (given identical input data)
- Signature generation (given identical keys and messages)
- Merkle tree construction (given identical leaf sets)
- Protobuf serialization (canonical encoding)

### 14.2 Test Vectors

Implementations MUST pass official test vectors published at:
`https://olocus.io/protocol/v1/test-vectors`

Test vectors MUST include:
- Sample LocationBlock with expected hash
- Sample LocusVisit with expected Merkle root
- Sample DailyAnchor with expected signature
- Sample LocationCredential with expected verification result
- Sample FriendshipRequest/Response with expected shared secret
- Sample Attestation with expected trust score
- Sample BatchAttestation with expected merkle root

**Inline Minimal Example** (Friendship Establishment):
```protobuf
message TestVector_Friendship {
  FriendshipRequest request = 1;
  FriendshipResponse response = 2;
  bytes expected_shared_secret = 3; // 32 bytes (for test only)
  Hash expected_commitment = 4;
}
```

---

# APPENDICES

## Appendix A: Error Code Registry

| Code | Name | User-Friendly Message |
|------|------|---------------------|
| 1001 | FRIENDSHIP_EXPIRED | "Friend request has expired. Please request again." |
| 1002 | VERIFICATION_CODE_MISMATCH | "Verification code incorrect. Please try again." |
| 1003 | FRIENDSHIP_ALREADY_EXISTS | "You are already friends with this user." |
| 2001 | BATCH_TOO_SMALL | "Not enough attestations to create batch." |
| 2002 | OVERLAP_INSUFFICIENT | "Insufficient spatial-temporal overlap for attestation." |
| 2003 | ATTESTATION_EXPIRED | "Attestation request has expired." |
| 3001 | DEVICE_TAMPERED | "Device integrity check failed. Cannot create attestation." |
| 3002 | TIME_UNCERTAINTY_HIGH | "Device time is too uncertain. Sync with NTP." |
| 4001 | NULLIFIER_EXISTS | "This location claim has already been published." |
| 4002 | CLAIM_EXPIRED | "Location claim has expired." |
| 5001 | ANCHOR_VERIFICATION_FAILED | "Failed to verify daily anchor on blockchain." |
| 5002 | MERKLE_PROOF_INVALID | "Merkle proof verification failed." |
| 6001 | TSA_UNAVAILABLE | "Timestamp authority is unavailable. Try again later." |
| 6002 | BLOCKCHAIN_CONFIRMATION_TIMEOUT | "Blockchain confirmation timed out." |

---

## Appendix B: Configuration Schema

```json
{
  "sampling": {
    "stationary": { "interval": 300, "accuracy": "balanced", "minDisplacement": 0 },
    "walking": { "interval": 60, "accuracy": "high", "minDisplacement": 10 },
    "vehicle": { "interval": 120, "accuracy": "balanced", "minDisplacement": 50 },
    "running": { "interval": 45, "accuracy": "high", "minDisplacement": 20 }
  },
  "battery": {
    "daily_budget_percent": 9.5,
    "low_battery_threshold": 30,
    "critical_battery_threshold": 20
  },
  "tsa": {
    "primary": "https://tsa1.olocus.io",
    "backups": ["https://tsa2.olocus.io", "https://tsa3.olocus.io"]
  },
  "batch": {
    "max_batch_size": 20,
    "min_batch_threshold": 3,
    "batch_window_ms": 300000
  },
  "trust": {
    "close_friend_weight": 1.0,
    "acquaintance_weight": 0.7,
    "colleague_weight": 0.5,
    "device_tampered_penalty": -0.5,
    "low_accuracy_penalty": -0.2,
    "sufficient_threshold": 0.6,
    "strong_threshold": 0.8
  }
}
```

---

## Appendix C: Compatibility Matrix

| Feature | v1.0 | v1.1 | v2.0 |
|--------|------|------|------|
| ECDH Curve25519 | Yes | Yes | Yes |
| ZKP (Phase 2) | No | Optional | Required |
| Batch Attestation | Yes | Yes | Yes |
| Nullifier Registry | Yes (salted) | Yes | Yes |
| RFC 3161 TSA | Yes | Yes | Deprecated |
| Blockchain Anchoring | Optional | Required | Required |
| Device Fingerprint Hashing | Yes | Yes | Yes |
| Shared Secret Zeroization | Yes | Yes | Yes |

---

## Appendix D: Privacy Impact Assessment

| Data | Collected? | Stored? | Transmitted? | Retention | Notes |
|------|-----------|---------|-------------|-----------|-------|
| Raw GPS | Yes | Encrypted | No | 90 days (configurable) | Never leaves device except in purchased claims |
| Device Model | Yes | Hashed | Hashed | 90 days | Only hash transmitted |
| OS Version | Yes | Hashed | Hashed | 90 days | Only hash transmitted |
| Installation ID | Yes | Yes | Yes | Indefinite | Derived, not PII |
| Visit Center | Yes | Encrypted | Commitment only | Indefinite | Only commitment or ZKP transmitted |
| Friendship Graph | Yes | Encrypted | Commitment only | Indefinite | Phase 2+ uses ZKP |
| Attestation Graph | Yes | Encrypted | Yes | Indefinite | Required for trust scoring |

---

## Appendix E: References

1. **FIPS 180-4**: Secure Hash Standard (SHA-256)
2. **RFC 8032**: Edwards-Curve Digital Signature Algorithm (EdDSA)
3. **RFC 6090**: Fundamental ECC Algorithms
4. **SEC1**: Elliptic Curve Cryptography
5. **RFC 7946**: The GeoJSON Format
6. **RFC 3161**: Time-Stamp Protocol (TSP)
7. **RFC 7519**: JSON Web Token (JWT)
8. **RFC 2119**: Key words for RFCs
9. **RFC 6749**: OAuth 2.0 Authorization Framework
10. **W3C VC**: Verifiable Credentials Data Model v1.1
11. **W3C DID**: Decentralized Identifiers v1.0
12. **NIST SP 800-38D**: Galois/Counter Mode (GCM)

---

## Appendix F: Changelog

### Version 1.0.0 (2024-11-08)
- Initial consolidated specification release
- Combined passive and active protocols into single document
- Integrated feedback: zeroization requirements, hashed device fingerprints, salted nullifiers
- Added comprehensive error code registry
- Added configuration schema and compatibility matrix
- Added privacy impact assessment
- Improved genesis block generation with user commitment
- Added configurable trust scoring with TrustWeightConfig
- Added inline test vector examples
- Strengthened security requirements around key management

---

**End of Olocus Protocol Specification v1.0**
