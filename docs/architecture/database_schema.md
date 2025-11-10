# Database Schema

## Status: 🔴 Not Started

## Overview

Complete database schemas for client-side storage across all platforms.

## Contents to be Added

### 1. SQLite Schema (iOS/Android)

#### Core Tables
- `chains` - Hash chain metadata
- `location_blocks` - Encrypted location blocks
- `visits` - Aggregated visits
- `daily_anchors` - Blockchain/TSA anchors
- `friendships` - Friendship credentials
- `attestations` - Received attestations
- `battery_usage` - Battery tracking

#### Indexes and Constraints
- Performance indexes
- Foreign key relationships
- Unique constraints

### 2. IndexedDB Schema (Web)

#### Object Stores
- Chain store
- Block store
- Visit store
- Anchor store
- Friendship store

#### Indexes
- Query optimization
- Composite indexes

### 3. Migration Strategy

#### Version Management
- Schema versioning
- Migration scripts
- Rollback procedures

#### Data Migration
- Forward compatibility
- Backward compatibility
- Data validation

### 4. Storage Optimization

#### Pruning Strategy
- 90-day retention for blocks
- Indefinite retention for visits
- Anchor preservation

#### Performance Targets
- Query performance
- Storage limits
- Indexing strategy

## References
- [Implementation Guide](../implementation/olocus_protocol_implementation_guide.md)
- [Protocol Specification](../specification/olocus_protocol_specification.md)