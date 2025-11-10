# Olocus Protocol Implementation Guide v1.0

**Status**: Draft  
**Version**: 1.0.0  
**Last Updated**: 2024-11-08  
**Related Specification**: olocus-protocol-specification-v1.0.md

---

## Purpose of This Document

This implementation guide provides **best practices, optimization strategies, and practical recommendations** for building Olocus Protocol-compliant applications. Unlike the specification (which defines MUST/MUST NOT requirements), this guide uses SHOULD/RECOMMENDED/MAY to describe proven approaches.

**This document covers**:
- Battery optimization techniques (target: 8-10% daily usage)
- Platform-specific implementations (iOS, Android, Web)
- Visit detection algorithms
- Network efficiency strategies
- Friendship management UX patterns
- Attestation batching strategies
- Device fingerprinting implementation
- Performance benchmarking
- Troubleshooting common issues

**Target Battery Budget**:
- **Passive Operations**: 7-8% daily
- **Active Operations**: 1-2% daily
- **Total**: 8-10% daily battery usage

---

## Table of Contents

### Part I: Passive Implementation (Continuous Tracking)
1. [Background Location Tracking](#1-background-location-tracking)
2. [Battery Optimization](#2-battery-optimization)
3. [Visit Detection Algorithms](#3-visit-detection-algorithms)
4. [Hash Chain Management](#4-hash-chain-management)
5. [Storage Management](#5-storage-management)
6. [Anchoring & Timestamping](#6-anchoring--timestamping)

### Part II: Active Implementation (User-Initiated Operations)
7. [Friendship Management](#7-friendship-management)
8. [Attestation Batching Strategies](#8-attestation-batching-strategies)
9. [Device Fingerprinting](#9-device-fingerprinting)
10. [Fraud Detection Heuristics](#10-fraud-detection-heuristics)
11. [User Experience Patterns](#11-user-experience-patterns)

### Part III: Cross-Platform Implementation
12. [Platform-Specific Guidance](#12-platform-specific-guidance)
13. [Network Synchronization](#13-network-synchronization)
14. [Performance Optimization](#14-performance-optimization)
15. [Testing & Validation](#15-testing--validation)
16. [Troubleshooting](#16-troubleshooting)

---

# PART I: PASSIVE IMPLEMENTATION (Continuous Tracking)

## 1. Background Location Tracking

### 1.1 Recommended Sampling Strategy

**Adaptive Sampling Based on Motion State**:

```javascript
const SamplingPolicy = {
  // When user is stationary
  stationary: {
    interval: 300,              // 5 minutes
    accuracy: 'balanced',       // ~50m accuracy
    minDisplacement: 0          // Any movement triggers update
  },
  
  // When user is walking
  walking: {
    interval: 60,               // 1 minute
    accuracy: 'high',           // ~30m accuracy
    minDisplacement: 10         // 10m minimum displacement
  },
  
  // When user is in vehicle
  vehicle: {
    interval: 120,              // 2 minutes
    accuracy: 'balanced',       // ~100m accuracy (sufficient for roads)
    minDisplacement: 50         // 50m minimum displacement
  },
  
  // When running
  running: {
    interval: 45,               // 45 seconds
    accuracy: 'high',           // ~30m accuracy
    minDisplacement: 20         // 20m minimum displacement
  }
};
```

### 1.2 Battery Preservation Modes

```javascript
const BatteryPreservation = {
  // Normal mode (battery > 30%)
  normal: {
    useGPS: true,
    useWiFi: true,
    useCellular: true,
    samplingMultiplier: 1.0
  },
  
  // Low battery mode (20% <= battery <= 30%)
  lowBattery: {
    useGPS: false,              // Disable GPS
    useWiFi: true,
    useCellular: true,
    samplingMultiplier: 2.0     // Half the frequency
  },
  
  // Critical battery mode (battery < 20%)
  critical: {
    useGPS: false,
    useWiFi: true,
    useCellular: true,
    samplingMultiplier: 4.0     // Quarter frequency
  },
  
  // Ultra-low mode (battery < 10%)
  ultraLow: {
    useGPS: false,
    useWiFi: false,
    useCellular: true,          // Cell tower only
    samplingMultiplier: 10.0,   // Every 50 minutes when stationary
    significantLocationOnly: true
  }
};
```

### 1.3 Platform-Specific APIs

#### iOS Implementation (Swift)

```swift
import CoreLocation

class LocationManager: NSObject, CLLocationManagerDelegate {
    let locationManager = CLLocationManager()
    var currentSamplingPolicy: SamplingPolicy?
    
    func startTracking() {
        locationManager.delegate = self
        locationManager.allowsBackgroundLocationUpdates = true
        locationManager.pausesLocationUpdatesAutomatically = false
        locationManager.activityType = .other
        
        // Start with significant location changes (battery efficient)
        locationManager.startMonitoringSignificantLocationChanges()
        
        // Also monitor visits (iOS Visit Monitoring)
        locationManager.startMonitoringVisits()
        
        // For high-accuracy when needed
        applyNormalSampling()
    }
    
    func applyNormalSampling() {
        locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters
        locationManager.distanceFilter = 50 // meters
        locationManager.startUpdatingLocation()
    }
    
    func applyLowBatterySampling() {
        // Switch to lower accuracy
        locationManager.desiredAccuracy = kCLLocationAccuracyKilometer
        locationManager.distanceFilter = 200
        locationManager.stopUpdatingLocation()
        // Rely on significant location changes
    }
    
    func locationManager(_ manager: CLLocationManager, 
                        didUpdateLocations locations: [CLLocation]) {
        for location in locations {
            Task {
                await processLocationSample(location)
            }
        }
    }
    
    func locationManager(_ manager: CLLocationManager, 
                        didVisit visit: CLVisit) {
        // iOS detected a visit - use this to supplement our algorithm
        Task {
            await processIOSVisit(visit)
        }
    }
    
    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        switch manager.authorizationStatus {
        case .authorizedAlways:
            startTracking()
        case .notDetermined:
            manager.requestAlwaysAuthorization()
        default:
            // Handle denied/restricted
            break
        }
    }
    
    private func processLocationSample(_ location: CLLocation) async {
        // Battery check
        let batteryLevel = UIDevice.current.batteryLevel
        if batteryLevel < 0.20 {
            applyLowBatterySampling()
        }
        
        // Create LocationBlock
        let block = try await createLocationBlock(
            coordinates: GeoCoordinates(
                longitude: location.coordinate.longitude,
                latitude: location.coordinate.latitude,
                altitude: location.altitude
            ),
            accuracy: GeoAccuracy(
                horizontal: location.horizontalAccuracy,
                vertical: location.verticalAccuracy
            ),
            timestamp: Timestamp(date: location.timestamp),
            motionState: await detectMotionState(),
            speed: location.speed,
            bearing: location.course
        )
        
        // Store encrypted block
        try await storeLocationBlock(block)
        
        // Update hash chain
        try await updateHashChain(blockHash: block.hash)
    }
}
```

#### Android Implementation (Kotlin)

```kotlin
import com.google.android.gms.location.*
import kotlinx.coroutines.flow.MutableStateFlow

class LocationTracker(private val context: Context) {
    private val fusedLocationClient = LocationServices.getFusedLocationProviderClient(context)
    private val activityRecognitionClient = ActivityRecognition.getClient(context)
    private val currentMotionState = MutableStateFlow(MotionState.MOTION_UNKNOWN)
    
    fun startTracking() {
        startLocationUpdates()
        startActivityRecognition()
    }
    
    private fun startLocationUpdates() {
        val locationRequest = LocationRequest.create().apply {
            // Balanced power and accuracy
            priority = LocationRequest.PRIORITY_BALANCED_POWER_ACCURACY
            interval = 300_000  // 5 minutes
            fastestInterval = 60_000  // 1 minute minimum
            maxWaitTime = 600_000  // Batch updates for 10 minutes
            smallestDisplacement = 50f  // 50 meters
        }
        
        val locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.locations.forEach { location ->
                    processLocationSample(location)
                }
            }
        }
        
        fusedLocationClient.requestLocationUpdates(
            locationRequest,
            locationCallback,
            Looper.getMainLooper()
        )
    }
    
    private fun startActivityRecognition() {
        val request = ActivityRecognitionRequest.Builder()
            .setDetectionInterval(60_000) // 1 minute
            .build()
        
        activityRecognitionClient.requestActivityRecognitionUpdates(
            request,
            getPendingIntent()
        )
    }
    
    private fun processLocationSample(location: Location) {
        // Battery check
        val batteryManager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        val batteryLevel = batteryManager.getIntProperty(
            BatteryManager.BATTERY_PROPERTY_CAPACITY
        )
        
        if (batteryLevel < 20) {
            adjustForLowBattery()
        }
        
        // Create LocationBlock
        lifecycleScope.launch {
            val block = createLocationBlock(
                coordinates = GeoCoordinates(
                    longitude = location.longitude,
                    latitude = location.latitude,
                    altitude = location.altitude
                ),
                accuracy = GeoAccuracy(
                    horizontal = location.accuracy.toDouble(),
                    vertical = null
                ),
                timestamp = Timestamp.fromMillis(location.time),
                motionState = currentMotionState.value,
                speed = location.speed.toDouble(),
                bearing = location.bearing.toDouble()
            )
            
            // Store encrypted block
            storeLocationBlock(block)
            
            // Update hash chain
            updateHashChain(block.hash)
        }
    }
    
    // Use Geofencing for visit detection
    fun setupGeofences(locations: List<LatLng>) {
        val geofencingClient = LocationServices.getGeofencingClient(context)
        val geofences = locations.map { location ->
            Geofence.Builder()
                .setRequestId(location.toString())
                .setCircularRegion(location.latitude, location.longitude, 50f)
                .setExpirationDuration(Geofence.NEVER_EXPIRE)
                .setTransitionTypes(
                    Geofence.GEOFENCE_TRANSITION_ENTER or 
                    Geofence.GEOFENCE_TRANSITION_EXIT
                )
                .build()
        }
        
        val geofencingRequest = GeofencingRequest.Builder()
            .addGeofences(geofences)
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
            .build()
        
        geofencingClient.addGeofences(geofencingRequest, getPendingIntent())
    }
}
```

#### Web Implementation (JavaScript)

```javascript
class WebLocationTracker {
  constructor() {
    this.watchId = null;
    this.batteryManager = null;
    this.samplingPolicy = 'normal';
  }
  
  async startTracking() {
    if (!navigator.geolocation) {
      throw new Error('Geolocation not supported');
    }
    
    // Get battery info if available
    if ('getBattery' in navigator) {
      this.batteryManager = await navigator.getBattery();
      this.batteryManager.addEventListener('levelchange', () => {
        this.adjustSamplingForBattery();
      });
    }
    
    // Start watching position
    this.watchId = navigator.geolocation.watchPosition(
      (position) => this.handlePosition(position),
      (error) => this.handleError(error),
      {
        enableHighAccuracy: true,
        maximumAge: 30000,
        timeout: 27000
      }
    );
  }
  
  async handlePosition(position) {
    const block = await this.createLocationBlock({
      coordinates: {
        longitude: position.coords.longitude,
        latitude: position.coords.latitude,
        altitude: position.coords.altitude || 0
      },
      accuracy: {
        horizontal: position.coords.accuracy,
        vertical: position.coords.altitudeAccuracy || null
      },
      timestamp: {
        unix_seconds: Math.floor(position.timestamp / 1000),
        nanos: (position.timestamp % 1000) * 1000000
      },
      motionState: await this.detectMotionState(),
      speed: position.coords.speed || 0,
      bearing: position.coords.heading || 0
    });
    
    // Store in IndexedDB
    await this.storeLocationBlock(block);
    
    // Update hash chain
    await this.updateHashChain(block.hash);
  }
  
  adjustSamplingForBattery() {
    if (!this.batteryManager) return;
    
    const level = this.batteryManager.level;
    
    if (level < 0.10) {
      this.samplingPolicy = 'ultraLow';
      // Stop high-frequency tracking
      if (this.watchId) {
        navigator.geolocation.clearWatch(this.watchId);
      }
    } else if (level < 0.20) {
      this.samplingPolicy = 'critical';
    } else if (level < 0.30) {
      this.samplingPolicy = 'lowBattery';
    } else {
      this.samplingPolicy = 'normal';
    }
  }
  
  stopTracking() {
    if (this.watchId) {
      navigator.geolocation.clearWatch(this.watchId);
      this.watchId = null;
    }
  }
}
```

---

## 2. Battery Optimization

### 2.1 Battery Budget Tracking

```javascript
class BatteryBudgetTracker {
  constructor(dailyBudgetPercent = 9.5) {
    this.dailyBudgetPercent = dailyBudgetPercent;
    this.usageLog = [];
  }
  
  async recordUsage(category, costPercent) {
    const now = Date.now();
    this.usageLog.push({
      timestamp: now,
      category,
      costPercent
    });
    
    // Prune old entries (older than 24 hours)
    const cutoff = now - 86400000;
    this.usageLog = this.usageLog.filter(e => e.timestamp > cutoff);
    
    // Check if over budget
    const totalUsage = this.getTotalUsage();
    if (totalUsage > this.dailyBudgetPercent) {
      await this.triggerThrottling();
    }
  }
  
  getTotalUsage() {
    return this.usageLog.reduce((sum, e) => sum + e.costPercent, 0);
  }
  
  getRemainingBudget() {
    return this.dailyBudgetPercent - this.getTotalUsage();
  }
  
  async triggerThrottling() {
    // Reduce sampling frequency
    await adjustSamplingPolicy('ultraLow');
    
    // Delay non-critical operations
    await postponeAnchoring();
    
    // Notify user
    await showNotification({
      title: 'Olocus Battery Optimization',
      body: 'Reducing tracking frequency to preserve battery'
    });
  }
}

// Usage categories and estimated costs
const BatteryCosts = {
  GPS_HIGH_ACCURACY: 0.05,      // 0.05% per sample
  GPS_BALANCED: 0.02,           // 0.02% per sample
  WIFI_SCAN: 0.01,              // 0.01% per scan
  CELL_TOWER: 0.005,            // 0.005% per sample
  MOTION_DETECTION: 0.001,      // 0.001% per check
  VISIT_DETECTION: 0.01,        // 0.01% per detection cycle
  HASH_COMPUTATION: 0.0001,     // 0.0001% per hash
  ENCRYPTION: 0.0005,           // 0.0005% per encryption
  NETWORK_SYNC: 0.02,           // 0.02% per sync
  ANCHORING: 0.05               // 0.05% per anchor
};
```

### 2.2 Smart Scheduling

```javascript
class SmartScheduler {
  async scheduleOperation(operation, priority = 'normal') {
    // Check battery level
    const batteryLevel = await getBatteryLevel();
    
    // Check network conditions
    const networkType = await getNetworkType();
    
    // Check device state
    const isCharging = await isDeviceCharging();
    const isIdle = await isDeviceIdle();
    
    // Decision matrix
    if (priority === 'critical') {
      // Execute immediately regardless of conditions
      return await operation();
    }
    
    if (priority === 'high') {
      if (batteryLevel > 0.20) {
        return await operation();
      } else {
        // Schedule for when charging
        return await scheduleForCharging(operation);
      }
    }
    
    if (priority === 'normal') {
      // Execute if conditions are good
      if (batteryLevel > 0.30 && (networkType === 'wifi' || isCharging)) {
        return await operation();
      } else {
        return await scheduleForOptimalConditions(operation);
      }
    }
    
    if (priority === 'low') {
      // Only execute when charging and idle
      if (isCharging && isIdle) {
        return await operation();
      } else {
        return await scheduleForCharging(operation);
      }
    }
  }
  
  async scheduleForCharging(operation) {
    return new Promise((resolve) => {
      const checkCharging = async () => {
        if (await isDeviceCharging()) {
          const result = await operation();
          resolve(result);
        } else {
          setTimeout(checkCharging, 300000); // Check every 5 minutes
        }
      };
      checkCharging();
    });
  }
}

// Usage example
const scheduler = new SmartScheduler();

// Critical: Location sample
await scheduler.scheduleOperation(
  () => captureLocationSample(),
  'critical'
);

// High: Daily anchor
await scheduler.scheduleOperation(
  () => createDailyAnchor(),
  'high'
);

// Normal: Visit detection
await scheduler.scheduleOperation(
  () => detectVisits(),
  'normal'
);

// Low: Pruning old data
await scheduler.scheduleOperation(
  () => pruneOldLocationBlocks(),
  'low'
);
```

### 2.3 Batching Strategies

```javascript
class OperationBatcher {
  constructor() {
    this.pendingOperations = {
      hashing: [],
      encryption: [],
      network: [],
      storage: []
    };
  }
  
  async addHashingOperation(data, callback) {
    this.pendingOperations.hashing.push({ data, callback });
    
    if (this.pendingOperations.hashing.length >= 10) {
      await this.flushHashing();
    }
  }
  
  async flushHashing() {
    const batch = this.pendingOperations.hashing.splice(0);
    
    // Process all hashes in single crypto operation
    const hashes = await Promise.all(
      batch.map(op => crypto.subtle.digest('SHA-256', op.data))
    );
    
    // Invoke callbacks
    batch.forEach((op, i) => op.callback(hashes[i]));
  }
  
  async addNetworkOperation(request, callback) {
    this.pendingOperations.network.push({ request, callback });
    
    // Wait for batch window or size threshold
    if (this.pendingOperations.network.length >= 5) {
      await this.flushNetwork();
    } else {
      setTimeout(() => this.flushNetwork(), 5000); // 5 second window
    }
  }
  
  async flushNetwork() {
    const batch = this.pendingOperations.network.splice(0);
    
    // Send all requests in single HTTP call
    const responses = await fetch('/api/batch', {
      method: 'POST',
      body: JSON.stringify(batch.map(op => op.request))
    }).then(r => r.json());
    
    // Invoke callbacks
    batch.forEach((op, i) => op.callback(responses[i]));
  }
}
```

---

## 3. Visit Detection Algorithms

### 3.1 DBSCAN-Based Clustering

```python
import numpy as np
from sklearn.cluster import DBSCAN
from scipy.spatial.distance import cdist

class VisitDetector:
    def __init__(self, epsilon=50, min_samples=5):
        """
        DBSCAN parameters for visit detection.
        
        Args:
            epsilon: Maximum distance between points (meters). Default: 50m
                    - Urban areas: 30-50m (tighter clustering)
                    - Suburban areas: 50-100m (wider clustering)
                    - Rural areas: 100-200m (sparse data points)
            
            min_samples: Minimum cluster size. Default: 5
                    - Higher accuracy mode: 5-7 samples (~15-20 minutes)
                    - Balanced mode: 3-5 samples (~10-15 minutes)
                    - Quick detection: 2-3 samples (~5-10 minutes)
        
        Rationale:
            - 50m epsilon captures typical building footprints
            - 5 min_samples ensures minimum 15 minutes dwell time
            - Balances false positives vs. detection sensitivity
        """
        self.epsilon = epsilon
        self.min_samples = min_samples
    
    def detect_visits(self, location_blocks):
        """
        Detect visits from a list of LocationBlocks using DBSCAN clustering.
        
        Returns: List of LocusVisit objects
        """
        if len(location_blocks) < self.min_samples:
            return []
        
        # Extract coordinates and timestamps
        coords = np.array([
            [block.payload.coordinates.latitude, 
             block.payload.coordinates.longitude]
            for block in location_blocks
        ])
        timestamps = np.array([
            block.payload.timestamp.unix_seconds 
            for block in location_blocks
        ])
        
        # Compute distance matrix using Haversine
        distances = self._haversine_distance_matrix(coords)
        
        # Apply DBSCAN
        clustering = DBSCAN(
            eps=self.epsilon, 
            min_samples=self.min_samples,
            metric='precomputed'
        )
        labels = clustering.fit_predict(distances)
        
        # Extract visits from clusters
        visits = []
        for label in set(labels):
            if label == -1:  # Noise points
                continue
            
            cluster_mask = labels == label
            cluster_coords = coords[cluster_mask]
            cluster_timestamps = timestamps[cluster_mask]
            cluster_blocks = [b for i, b in enumerate(location_blocks) 
                            if cluster_mask[i]]
            
            visit = self._create_visit(
                cluster_coords, 
                cluster_timestamps, 
                cluster_blocks
            )
            visits.append(visit)
        
        return visits
    
    def _haversine_distance_matrix(self, coords):
        """
        Compute pairwise Haversine distances between coordinates.
        """
        n = len(coords)
        distances = np.zeros((n, n))
        
        for i in range(n):
            for j in range(i+1, n):
                dist = self._haversine_distance(coords[i], coords[j])
                distances[i, j] = dist
                distances[j, i] = dist
        
        return distances
    
    def _haversine_distance(self, coord1, coord2):
        """
        Calculate Haversine distance between two coordinates (meters).
        """
        R = 6371000  # Earth radius in meters
        
        lat1, lon1 = np.radians(coord1)
        lat2, lon2 = np.radians(coord2)
        
        dlat = lat2 - lat1
        dlon = lon2 - lon1
        
        a = np.sin(dlat/2)**2 + np.cos(lat1) * np.cos(lat2) * np.sin(dlon/2)**2
        c = 2 * np.arcsin(np.sqrt(a))
        
        return R * c
    
    def _create_visit(self, coords, timestamps, blocks):
        """
        Create a LocusVisit from cluster data.
        """
        # Compute centroid
        center_lat = np.mean(coords[:, 0])
        center_lon = np.mean(coords[:, 1])
        
        # Compute radius (95th percentile distance from centroid)
        center = np.array([center_lat, center_lon])
        distances_from_center = [
            self._haversine_distance(center, coord) 
            for coord in coords
        ]
        radius = np.percentile(distances_from_center, 95)
        
        # Temporal bounds
        arrival_time = np.min(timestamps)
        departure_time = np.max(timestamps)
        duration = departure_time - arrival_time
        
        # Classify visit type
        visit_type, confidence = self._classify_visit_type(
            center, arrival_time, departure_time, duration
        )
        
        # Generate visit ID
        visit_id = generate_hash(random_bytes(32))
        
        # Compute blocks Merkle root
        block_hashes = [b.hash for b in blocks]
        blocks_merkle_root = compute_merkle_root(block_hashes)
        
        # Compute visit commitment
        visit_commitment = hash_combine(
            visit_id, 
            f"{center_lat},{center_lon}", 
            str(arrival_time)
        )
        
        return LocusVisit(
            visit_id=visit_id,
            center=GeoCoordinates(center_lon, center_lat),
            radius_meters=radius,
            arrival_time=Timestamp.from_unix(arrival_time),
            departure_time=Timestamp.from_unix(departure_time),
            duration_seconds=duration,
            visit_type=visit_type,
            confidence=confidence,
            block_hashes=block_hashes,
            blocks_merkle_root=blocks_merkle_root,
            visit_commitment=visit_commitment
        )
    
    def _classify_visit_type(self, center, arrival, departure, duration):
        """
        Classify visit type using heuristics.
        """
        # Check against historical visits
        if self._is_frequent_location(center):
            if self._overlaps_nighttime(arrival, departure):
                return VisitType.VISIT_HOME, 0.9
            
            if self._is_weekday(arrival) and self._overlaps_work_hours(arrival, departure):
                return VisitType.VISIT_WORK, 0.85
        
        # Transit: short duration
        if duration < 1800:  # 30 minutes
            return VisitType.VISIT_TRANSIT, 0.7
        
        # Dining: 30-120 minutes, dinner hours
        if 1800 <= duration <= 7200 and self._overlaps_dinner_hours(arrival, departure):
            return VisitType.VISIT_DINING, 0.6
        
        # Shopping: 60-180 minutes
        if 3600 <= duration <= 10800:
            return VisitType.VISIT_SHOPPING, 0.5
        
        return VisitType.VISIT_OTHER, 0.5
```

### 3.2 Incremental Visit Detection

```javascript
class IncrementalVisitDetector {
  constructor() {
    this.activeVisit = null;
    this.recentBlocks = [];
    this.epsilon = 50; // meters
    this.minDuration = 600; // 10 minutes
  }
  
  async processNewBlock(block) {
    this.recentBlocks.push(block);
    
    // Keep only last hour of blocks
    const oneHourAgo = Date.now() - 3600000;
    this.recentBlocks = this.recentBlocks.filter(
      b => b.payload.timestamp.unix_seconds * 1000 > oneHourAgo
    );
    
    if (!this.activeVisit) {
      // Check if we should start a new visit
      if (this._isStationary()) {
        this.activeVisit = {
          startBlock: block,
          blocks: [block],
          startTime: block.payload.timestamp
        };
      }
    } else {
      // Check if current block extends the active visit
      const distanceFromStart = this._haversineDistance(
        this.activeVisit.startBlock.payload.coordinates,
        block.payload.coordinates
      );
      
      if (distanceFromStart <= this.epsilon) {
        // Extend visit
        this.activeVisit.blocks.push(block);
      } else {
        // End visit if duration meets threshold
        const duration = block.payload.timestamp.unix_seconds - 
                        this.activeVisit.startTime.unix_seconds;
        
        if (duration >= this.minDuration) {
          const visit = await this._finalizeVisit();
          this.activeVisit = null;
          return visit;
        } else {
          // Too short, discard
          this.activeVisit = null;
        }
      }
    }
    
    return null;
  }
  
  _isStationary() {
    if (this.recentBlocks.length < 3) return false;
    
    // Check if last 3 blocks are within epsilon
    const recent = this.recentBlocks.slice(-3);
    const center = this._computeCentroid(recent);
    
    for (const block of recent) {
      const dist = this._haversineDistance(center, block.payload.coordinates);
      if (dist > this.epsilon) return false;
    }
    
    return true;
  }
  
  async _finalizeVisit() {
    const blocks = this.activeVisit.blocks;
    
    // Compute centroid
    const center = this._computeCentroid(blocks);
    
    // Compute radius
    const distances = blocks.map(b => 
      this._haversineDistance(center, b.payload.coordinates)
    );
    const radius = this._percentile(distances, 95);
    
    // Create visit
    const visit = await createLocusVisit({
      center,
      radius,
      arrival_time: blocks[0].payload.timestamp,
      departure_time: blocks[blocks.length - 1].payload.timestamp,
      blocks: blocks
    });
    
    // Store visit
    await storeVisit(visit);
    
    return visit;
  }
}
```

---

## 4. Hash Chain Management

### 4.1 Chain Initialization

```javascript
class HashChainManager {
  async initializeChain(userDID, deviceInfo) {
    // Generate installation ID
    const installationId = await this._generateInstallationId();
    
    // Create genesis block
    const genesisTimestamp = Timestamp.now();
    const userCommitment = await this._hashCombine(
      userDID,
      genesisTimestamp.unix_seconds.toString()
    );
    
    const genesisBlock = {
      created_at: genesisTimestamp,
      user_commitment: userCommitment,
      device_fingerprint: await this._createDeviceFingerprint(deviceInfo)
    };
    
    const genesisHash = await this._hashMessage(genesisBlock);
    
    // Create chain ID
    const chainId = await this._hashCombine(
      userDID,
      installationId,
      genesisTimestamp.unix_seconds.toString()
    );
    
    // Initialize chain
    const chain = {
      chain_id: chainId,
      genesis_block: genesisBlock,
      current_head: genesisHash,
      last_anchor_timestamp: null,
      last_anchor_block_hash: null,
      protocol_version: 1
    };
    
    await this._storeChain(chain);
    
    return chain;
  }
  
  async _generateInstallationId() {
    // Platform-specific unique identifier + timestamp
    const platformId = await this._getPlatformId();
    const timestamp = Date.now();
    return await this._hashCombine(platformId, timestamp.toString());
  }
  
  async _createDeviceFingerprint(deviceInfo) {
    return {
      os: deviceInfo.os,
      os_version_hash: await this._hashString(deviceInfo.os_version),
      device_model_hash: await this._hashString(deviceInfo.device_model),
      app_version: deviceInfo.app_version,
      installation_id: await this._generateInstallationId()
    };
  }
}
```

### 4.2 Block Appending

```javascript
class BlockAppender {
  async appendBlock(chain, locationData) {
    // Get current head
    const currentHead = chain.current_head;
    
    // Get next block index
    const blockIndex = await this._getNextBlockIndex(chain.chain_id);
    
    // Create payload
    const payload = {
      block_index: blockIndex,
      timestamp: Timestamp.now(),
      coordinates: locationData.coordinates,
      accuracy: locationData.accuracy,
      previous_hash: currentHead,
      motion_state: locationData.motion_state,
      speed: locationData.speed,
      bearing: locationData.bearing,
      device_state: await this._getDeviceState()
    };
    
    // Create header
    const header = {
      type: "OLOCUS_LOCATION_BLOCK",
      algorithm: "SHA256",
      version: "1.0"
    };
    
    // Compute block hash
    const blockHash = await this._hashBlock(header, payload);
    
    // Sign block
    const signature = await this._signHash(blockHash);
    
    // Assemble block
    const block = {
      header,
      payload,
      signature
    };
    
    // Encrypt and store
    await this._storeEncryptedBlock(chain.chain_id, block);
    
    // Update chain head
    chain.current_head = blockHash;
    await this._updateChain(chain);
    
    return block;
  }
  
  async _getDeviceState() {
    return {
      battery_level: await getBatteryLevel(),
      low_power_mode: await isLowPowerMode(),
      network_type: await getNetworkType(),
      time_uncertainty_ms: await getTimeUncertainty()
    };
  }
}
```

### 4.3 Chain Verification

```javascript
class ChainVerifier {
  async verifyChainSegment(chain, startIndex, endIndex) {
    const blocks = await this._loadBlocks(chain.chain_id, startIndex, endIndex);
    
    let previousHash = startIndex === 0 ? 
                       chain.genesis_block.genesis_hash : 
                       blocks[0].payload.previous_hash;
    
    for (let i = 0; i < blocks.length; i++) {
      const block = blocks[i];
      
      // Verify signature
      const blockHash = await this._hashBlock(block.header, block.payload);
      const signatureValid = await this._verifySignature(
        blockHash,
        block.signature,
        chain.user_public_key
      );
      
      if (!signatureValid) {
        return {
          valid: false,
          error: `Invalid signature at block ${startIndex + i}`,
          block_index: startIndex + i
        };
      }
      
      // Verify previous_hash link
      if (block.payload.previous_hash !== previousHash) {
        return {
          valid: false,
          error: `Broken chain at block ${startIndex + i}`,
          block_index: startIndex + i
        };
      }
      
      // Verify block_index is sequential
      if (block.payload.block_index !== startIndex + i) {
        return {
          valid: false,
          error: `Non-sequential block index at ${startIndex + i}`,
          block_index: startIndex + i
        };
      }
      
      // Verify timestamp is monotonically increasing
      if (i > 0 && block.payload.timestamp.unix_seconds <= 
                   blocks[i-1].payload.timestamp.unix_seconds) {
        return {
          valid: false,
          error: `Non-monotonic timestamp at block ${startIndex + i}`,
          block_index: startIndex + i
        };
      }
      
      previousHash = blockHash;
    }
    
    return { valid: true };
  }
}
```

### 4.4 Merkle Tree Implementation

```javascript
class MerkleTree {
  /**
   * Build a Merkle tree from block hashes.
   * Follows Olocus Protocol Specification Section 2.1.
   */
  static async build(blockHashes) {
    if (blockHashes.length === 0) {
      // Empty tree sentinel
      return {
        root: await hashString("EMPTY_TREE"),
        leaves: [],
        tree: []
      };
    }
    
    if (blockHashes.length === 1) {
      // Single element - hash it once
      const leaf = await hashBytes(blockHashes[0]);
      return {
        root: leaf,
        leaves: [leaf],
        tree: [[leaf]]
      };
    }
    
    // Create leaf layer by hashing each block hash
    let currentLayer = await Promise.all(
      blockHashes.map(hash => hashBytes(hash))
    );
    
    const tree = [currentLayer];
    
    // Build tree bottom-up
    while (currentLayer.length > 1) {
      const nextLayer = [];
      
      // Process pairs
      for (let i = 0; i < currentLayer.length; i += 2) {
        const left = currentLayer[i];
        const right = i + 1 < currentLayer.length ? 
                      currentLayer[i + 1] : 
                      currentLayer[i]; // Duplicate last if odd
        
        // Parent = H(left || right)
        const parent = await hashBytes(concatenateBytes(left, right));
        nextLayer.push(parent);
      }
      
      tree.push(nextLayer);
      currentLayer = nextLayer;
    }
    
    return {
      root: currentLayer[0],
      leaves: tree[0],
      tree: tree
    };
  }
  
  /**
   * Generate Merkle proof for a specific leaf index.
   */
  static getProof(tree, leafIndex) {
    const proof = {
      leaf: tree.leaves[leafIndex],
      path: [],
      directions: [] // true = right, false = left
    };
    
    let currentIndex = leafIndex;
    
    // Walk up the tree
    for (let level = 0; level < tree.tree.length - 1; level++) {
      const currentLayer = tree.tree[level];
      const isRightNode = currentIndex % 2 === 1;
      const siblingIndex = isRightNode ? currentIndex - 1 : currentIndex + 1;
      
      // Add sibling to proof path
      if (siblingIndex < currentLayer.length) {
        proof.path.push(currentLayer[siblingIndex]);
        proof.directions.push(!isRightNode); // Sibling is on opposite side
      }
      
      currentIndex = Math.floor(currentIndex / 2);
    }
    
    return proof;
  }
  
  /**
   * Verify a Merkle proof.
   */
  static async verifyProof(proof, expectedRoot) {
    let currentHash = proof.leaf;
    
    for (let i = 0; i < proof.path.length; i++) {
      const sibling = proof.path[i];
      const isRight = proof.directions[i];
      
      if (isRight) {
        // Sibling on right
        currentHash = await hashBytes(concatenateBytes(currentHash, sibling));
      } else {
        // Sibling on left
        currentHash = await hashBytes(concatenateBytes(sibling, currentHash));
      }
    }
    
    return bytesEqual(currentHash, expectedRoot);
  }
}

// Helper functions
function concatenateBytes(a, b) {
  const result = new Uint8Array(a.length + b.length);
  result.set(a, 0);
  result.set(b, a.length);
  return result;
}

function bytesEqual(a, b) {
  if (a.length !== b.length) return false;
  for (let i = 0; i < a.length; i++) {
    if (a[i] !== b[i]) return false;
  }
  return true;
}

async function hashBytes(data) {
  return new Uint8Array(
    await crypto.subtle.digest('SHA-256', data)
  );
}

async function hashString(str) {
  return hashBytes(new TextEncoder().encode(str));
}
```

**Python Implementation**:

```python
import hashlib
from typing import List, Tuple

class MerkleTree:
    @staticmethod
    def build(block_hashes: List[bytes]) -> dict:
        """Build Merkle tree from block hashes."""
        if not block_hashes:
            return {
                'root': hashlib.sha256(b"EMPTY_TREE").digest(),
                'leaves': [],
                'tree': []
            }
        
        if len(block_hashes) == 1:
            leaf = hashlib.sha256(block_hashes[0]).digest()
            return {
                'root': leaf,
                'leaves': [leaf],
                'tree': [[leaf]]
            }
        
        # Create leaf layer
        current_layer = [hashlib.sha256(h).digest() for h in block_hashes]
        tree = [current_layer[:]]
        
        # Build tree bottom-up
        while len(current_layer) > 1:
            next_layer = []
            
            for i in range(0, len(current_layer), 2):
                left = current_layer[i]
                right = current_layer[i + 1] if i + 1 < len(current_layer) else left
                
                # Parent = H(left || right)
                parent = hashlib.sha256(left + right).digest()
                next_layer.append(parent)
            
            tree.append(next_layer)
            current_layer = next_layer
        
        return {
            'root': current_layer[0],
            'leaves': tree[0],
            'tree': tree
        }
    
    @staticmethod
    def get_proof(tree: dict, leaf_index: int) -> dict:
        """Generate Merkle proof for leaf at index."""
        proof = {
            'leaf': tree['leaves'][leaf_index],
            'path': [],
            'directions': []
        }
        
        current_index = leaf_index
        
        for level in range(len(tree['tree']) - 1):
            current_layer = tree['tree'][level]
            is_right_node = current_index % 2 == 1
            sibling_index = current_index - 1 if is_right_node else current_index + 1
            
            if sibling_index < len(current_layer):
                proof['path'].append(current_layer[sibling_index])
                proof['directions'].append(not is_right_node)
            
            current_index //= 2
        
        return proof
    
    @staticmethod
    def verify_proof(proof: dict, expected_root: bytes) -> bool:
        """Verify Merkle proof."""
        current_hash = proof['leaf']
        
        for sibling, is_right in zip(proof['path'], proof['directions']):
            if is_right:
                current_hash = hashlib.sha256(current_hash + sibling).digest()
            else:
                current_hash = hashlib.sha256(sibling + current_hash).digest()
        
        return current_hash == expected_root
```

---

## 5. Storage Management

### 5.1 Database Schema (SQLite)

```sql
-- Hash chain metadata
CREATE TABLE chains (
  chain_id BLOB PRIMARY KEY,
  user_did TEXT NOT NULL,
  genesis_block BLOB NOT NULL,
  current_head BLOB NOT NULL,
  last_anchor_timestamp INTEGER,
  last_anchor_block_hash BLOB,
  protocol_version INTEGER NOT NULL,
  created_at INTEGER NOT NULL
);

-- Encrypted location blocks
CREATE TABLE location_blocks (
  block_hash BLOB PRIMARY KEY,
  chain_id BLOB NOT NULL,
  block_index INTEGER NOT NULL,
  ciphertext BLOB NOT NULL,
  nonce BLOB NOT NULL,
  authentication_tag BLOB NOT NULL,
  key_id TEXT NOT NULL,
  timestamp INTEGER NOT NULL,
  FOREIGN KEY (chain_id) REFERENCES chains(chain_id),
  UNIQUE(chain_id, block_index)
);

CREATE INDEX idx_blocks_chain_timestamp ON location_blocks(chain_id, timestamp);
CREATE INDEX idx_blocks_chain_index ON location_blocks(chain_id, block_index);

-- Encrypted visits
CREATE TABLE visits (
  visit_id BLOB PRIMARY KEY,
  chain_id BLOB NOT NULL,
  visit_commitment BLOB NOT NULL,
  ciphertext BLOB NOT NULL,
  nonce BLOB NOT NULL,
  authentication_tag BLOB NOT NULL,
  key_id TEXT NOT NULL,
  arrival_time INTEGER NOT NULL,
  departure_time INTEGER NOT NULL,
  visit_type INTEGER,
  confidence REAL,
  FOREIGN KEY (chain_id) REFERENCES chains(chain_id)
);

CREATE INDEX idx_visits_chain_time ON visits(chain_id, arrival_time);
CREATE INDEX idx_visits_commitment ON visits(visit_commitment);

-- Daily anchors (unencrypted, no PII)
CREATE TABLE daily_anchors (
  anchor_id BLOB PRIMARY KEY,
  chain_id BLOB NOT NULL,
  chain_head_hash BLOB NOT NULL,
  visits_merkle_root BLOB NOT NULL,
  period_start INTEGER NOT NULL,
  period_end INTEGER NOT NULL,
  tsa_token BLOB,
  tsa_url TEXT,
  blockchain_tx_hash TEXT,
  blockchain_block_number INTEGER,
  anchor_signature BLOB NOT NULL,
  created_at INTEGER NOT NULL,
  FOREIGN KEY (chain_id) REFERENCES chains(chain_id)
);

CREATE INDEX idx_anchors_chain_period ON daily_anchors(chain_id, period_start);

-- Friendship credentials
CREATE TABLE friendships (
  credential_id BLOB PRIMARY KEY,
  participant_a_did TEXT NOT NULL,
  participant_b_did TEXT NOT NULL,
  shared_secret_encrypted BLOB NOT NULL,
  shared_secret_commitment BLOB NOT NULL,
  established_at INTEGER NOT NULL,
  expires_at INTEGER,
  friendship_type INTEGER,
  participant_a_signature BLOB NOT NULL,
  participant_b_signature BLOB NOT NULL
);

CREATE INDEX idx_friendships_participants ON friendships(participant_a_did, participant_b_did);

-- Battery usage log
CREATE TABLE battery_usage (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp INTEGER NOT NULL,
  category TEXT NOT NULL,
  cost_percent REAL NOT NULL
);

CREATE INDEX idx_battery_timestamp ON battery_usage(timestamp);
```

### 5.2 Encryption Key Management

```javascript
class KeyManager {
  async initializeKeys() {
    // Generate master key in secure enclave
    const masterKey = await this._generateMasterKey();
    
    // Derive encryption key
    const salt = await this._generateSalt();
    const encryptionKey = await this._deriveKey(masterKey, salt, 'encryption');
    
    // Store key reference (not the key itself)
    await this._storeKeyMetadata({
      key_id: 'master_encryption_key_v1',
      algorithm: 'AES-256-GCM',
      derived_from: 'secure_enclave',
      salt: salt,
      created_at: Date.now()
    });
    
    return encryptionKey;
  }
  
  async encryptLocationBlock(block, key) {
    // Serialize block
    const plaintext = this._serializeProtobuf(block);
    
    // Generate unique nonce
    const nonce = crypto.getRandomValues(new Uint8Array(12));
    
    // Encrypt with AES-256-GCM
    const ciphertext = await crypto.subtle.encrypt(
      {
        name: 'AES-GCM',
        iv: nonce,
        tagLength: 128
      },
      key,
      plaintext
    );
    
    // Extract authentication tag (last 16 bytes)
    const ciphertextArray = new Uint8Array(ciphertext);
    const authTag = ciphertextArray.slice(-16);
    const actualCiphertext = ciphertextArray.slice(0, -16);
    
    return {
      ciphertext: actualCiphertext,
      nonce: nonce,
      authentication_tag: authTag,
      key_id: 'master_encryption_key_v1'
    };
  }
  
  async decryptLocationBlock(encrypted, key) {
    // Reconstruct full ciphertext with auth tag
    const fullCiphertext = new Uint8Array([
      ...encrypted.ciphertext,
      ...encrypted.authentication_tag
    ]);
    
    // Decrypt
    const plaintext = await crypto.subtle.decrypt(
      {
        name: 'AES-GCM',
        iv: encrypted.nonce,
        tagLength: 128
      },
      key,
      fullCiphertext
    );
    
    // Deserialize protobuf
    return this._deserializeProtobuf(plaintext, LocationBlock);
  }
}
```

### 5.3 Pruning Strategy

```javascript
class StoragePruner {
  async pruneOldBlocks(chain, retentionPolicy) {
    const cutoffTime = Date.now() - (retentionPolicy.raw_blocks_days * 86400000);
    
    // Find blocks older than retention period
    const oldBlocks = await db.query(`
      SELECT block_hash, block_index, timestamp
      FROM location_blocks
      WHERE chain_id = ? AND timestamp < ?
      ORDER BY block_index ASC
    `, [chain.chain_id, cutoffTime]);
    
    for (const block of oldBlocks) {
      // Verify block is included in a visit
      const included = await this._isBlockInVisit(block.block_hash);
      
      if (!included) {
        // Don't prune if not in visit yet
        continue;
      }
      
      // Verify corresponding anchor exists
      const anchorExists = await this._hasAnchor(chain.chain_id, block.timestamp);
      
      if (!anchorExists) {
        // Don't prune if not anchored
        continue;
      }
      
      // Safe to delete
      await db.execute(`
        DELETE FROM location_blocks WHERE block_hash = ?
      `, [block.block_hash]);
      
      console.log(`Pruned block ${block.block_index} from ${new Date(block.timestamp)}`);
    }
  }
  
  async _isBlockInVisit(blockHash) {
    // Check if block_hash appears in any visit's block_hashes
    const visits = await db.query(`
      SELECT visit_id, ciphertext, nonce, authentication_tag, key_id
      FROM visits
    `);
    
    for (const visit of visits) {
      const decrypted = await decryptVisit(visit);
      if (decrypted.block_hashes.some(h => h.equals(blockHash))) {
        return true;
      }
    }
    
    return false;
  }
  
  async _hasAnchor(chainId, timestamp) {
    const anchor = await db.queryOne(`
      SELECT anchor_id FROM daily_anchors
      WHERE chain_id = ? AND period_start <= ? AND period_end >= ?
    `, [chainId, timestamp, timestamp]);
    
    return anchor !== null;
  }
}
```

---

## 6. Anchoring & Timestamping

### 6.1 RFC 3161 TSA Client

```javascript
class TSAClient {
  constructor(tsaUrl = 'https://tsa1.olocus.io') {
    this.tsaUrl = tsaUrl;
    this.backupUrls = [
      'https://tsa2.olocus.io',
      'https://tsa3.olocus.io',
      'https://freetsa.org/tsr', // Public fallback
    ];
  }
  
  async requestTimestamp(dataHash) {
    // Construct TSA request (RFC 3161 format)
    const request = this._constructTSARequest(dataHash);
    
    // Try primary TSA with exponential backoff
    try {
      return await this._sendRequestWithRetry(this.tsaUrl, request, 3);
    } catch (error) {
      console.warn(`Primary TSA failed after retries: ${error.message}`);
    }
    
    // Try backup TSAs
    for (const url of this.backupUrls) {
      try {
        console.log(`Trying backup TSA: ${url}`);
        return await this._sendRequestWithRetry(url, request, 2);
      } catch (error) {
        console.warn(`Backup TSA ${url} failed: ${error.message}`);
      }
    }
    
    throw new Error('All TSA servers unavailable');
  }
  
  async _sendRequestWithRetry(url, request, maxRetries = 3) {
    let lastError;
    
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        return await this._sendRequest(url, request);
      } catch (error) {
        lastError = error;
        
        if (attempt < maxRetries - 1) {
          // Exponential backoff: 1s, 2s, 4s
          const delayMs = Math.pow(2, attempt) * 1000;
          console.log(`TSA request failed (attempt ${attempt + 1}/${maxRetries}), retrying in ${delayMs}ms...`);
          await sleep(delayMs);
        }
      }
    }
    
    throw lastError;
  }
  
  _constructTSARequest(dataHash) {
    // ASN.1 DER encoding for RFC 3161 TimeStampReq
    // Implementation depends on crypto library
    return {
      version: 1,
      messageImprint: {
        hashAlgorithm: 'SHA-256',
        hashedMessage: dataHash
      },
      reqPolicy: null,
      nonce: crypto.getRandomValues(new Uint8Array(8)),
      certReq: true
    };
  }
  
  async _sendRequest(url, request) {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/timestamp-query'
      },
      body: this._encodeRequest(request)
    });
    
    if (!response.ok) {
      throw new Error(`TSA returned ${response.status}`);
    }
    
    const token = await response.arrayBuffer();
    
    // Verify token
    const valid = await this._verifyToken(token);
    if (!valid) {
      throw new Error('Invalid TSA token');
    }
    
    return token;
  }
  
  async _verifyToken(token) {
    // Parse and verify TSA signature
    // Implementation depends on crypto library
    const parsed = this._parseTSAToken(token);
    
    // Verify signature
    const signatureValid = await this._verifyTSASignature(
      parsed.signature,
      parsed.signedData,
      parsed.certificate
    );
    
    return signatureValid;
  }
}
```

### 6.2 Daily Anchor Creation

```javascript
class AnchorCreator {
  constructor() {
    this.maxPendingAnchors = 7; // Maximum 7 days of unsynced anchors
  }
  
  async createDailyAnchor(chain, periodStart, periodEnd) {
    // Check pending anchor queue
    const pendingCount = await this._getPendingAnchorCount(chain.chain_id);
    
    if (pendingCount >= this.maxPendingAnchors) {
      throw new Error(
        `Too many pending anchors (${pendingCount}). ` +
        `Please sync before creating new anchors.`
      );
    }
    
    // Collect all blocks in period
    const blocks = await db.query(`
      SELECT block_hash FROM location_blocks
      WHERE chain_id = ? AND timestamp >= ? AND timestamp < ?
      ORDER BY block_index ASC
    `, [chain.chain_id, periodStart, periodEnd]);
    
    if (blocks.length === 0) {
      console.log('No blocks to anchor in this period');
      return null;
    }
    
    // Get chain head hash
    const chainHeadHash = blocks[blocks.length - 1].block_hash;
    
    // Collect all visits in period
    const visits = await db.query(`
      SELECT visit_commitment FROM visits
      WHERE chain_id = ? AND arrival_time >= ? AND departure_time < ?
    `, [chain.chain_id, periodStart, periodEnd]);
    
    // Compute visits merkle root
    const visitCommitments = visits.map(v => v.visit_commitment);
    const visitsMerkleRoot = computeMerkleRoot(visitCommitments);
    
    // Construct anchor (without tsa_token yet)
    const anchorId = generateHash(randomBytes(32));
    const anchor = {
      anchor_id: anchorId,
      chain_head_hash: chainHeadHash,
      visits_merkle_root: visitsMerkleRoot,
      period_start: periodStart,
      period_end: periodEnd
    };
    
    // Compute anchor hash
    const anchorHash = await hashMessage(anchor);
    
    // Request TSA timestamp (with retry logic)
    try {
      const tsaClient = new TSAClient();
      const tsaToken = await tsaClient.requestTimestamp(anchorHash);
      
      anchor.tsa_token = tsaToken;
      anchor.tsa_url = tsaClient.tsaUrl;
    } catch (error) {
      console.warn('TSA request failed, anchor will be queued:', error);
      // Continue without TSA token - will retry during sync
    }
    
    // Optionally publish to blockchain (Phase 1: optional)
    if (shouldPublishToBlockchain() && isOnline()) {
      try {
        const blockchainRef = await this._publishToBlockchain(anchorHash);
        anchor.blockchain_ref = blockchainRef;
      } catch (error) {
        console.warn('Blockchain publish failed, will retry during sync:', error);
        // Continue - will retry during sync
      }
    }
    
    // Sign anchor
    const anchorSignature = await signHash(anchorHash);
    anchor.anchor_signature = anchorSignature;
    
    // Store anchor with sync status
    await storeAnchor(chain.chain_id, anchor, {
      synced: anchor.tsa_token && anchor.blockchain_ref ? 1 : 0,
      needs_tsa: !anchor.tsa_token,
      needs_blockchain: !anchor.blockchain_ref
    });
    
    // Update chain
    chain.last_anchor_timestamp = periodEnd;
    chain.last_anchor_block_hash = chainHeadHash;
    await updateChain(chain);
    
    return anchor;
  }
  
  async _getPendingAnchorCount(chainId) {
    const result = await db.queryOne(`
      SELECT COUNT(*) as count FROM daily_anchors
      WHERE chain_id = ? AND synced = 0
    `, [chainId]);
    
    return result.count;
  }
  
  /**
   * Retry syncing pending anchors when online.
   */
  async syncPendingAnchors(chainId) {
    const pending = await db.query(`
      SELECT * FROM daily_anchors
      WHERE chain_id = ? AND synced = 0
      ORDER BY period_start ASC
      LIMIT ?
    `, [chainId, this.maxPendingAnchors]);
    
    console.log(`Syncing ${pending.length} pending anchors...`);
    
    for (const anchor of pending) {
      let updated = false;
      
      // Retry TSA if needed
      if (anchor.needs_tsa) {
        try {
          const tsaClient = new TSAClient();
          const anchorHash = await hashMessage(anchor);
          const tsaToken = await tsaClient.requestTimestamp(anchorHash);
          
          await db.execute(`
            UPDATE daily_anchors
            SET tsa_token = ?, tsa_url = ?, needs_tsa = 0
            WHERE anchor_id = ?
          `, [tsaToken, tsaClient.tsaUrl, anchor.anchor_id]);
          
          updated = true;
        } catch (error) {
          console.error(`Failed to get TSA for anchor ${anchor.anchor_id}:`, error);
        }
      }
      
      // Retry blockchain if needed
      if (anchor.needs_blockchain && shouldPublishToBlockchain()) {
        try {
          const anchorHash = await hashMessage(anchor);
          const blockchainRef = await this._publishToBlockchain(anchorHash);
          
          await db.execute(`
            UPDATE daily_anchors
            SET blockchain_tx_hash = ?, blockchain_block_number = ?, needs_blockchain = 0
            WHERE anchor_id = ?
          `, [blockchainRef.transaction_hash, blockchainRef.block_number, anchor.anchor_id]);
          
          updated = true;
        } catch (error) {
          console.error(`Failed to publish anchor ${anchor.anchor_id} to blockchain:`, error);
        }
      }
      
      // Mark as synced if both TSA and blockchain succeeded
      if (updated && !anchor.needs_tsa && !anchor.needs_blockchain) {
        await db.execute(`
          UPDATE daily_anchors SET synced = 1 WHERE anchor_id = ?
        `, [anchor.anchor_id]);
      }
    }
  }
  
  async _publishToBlockchain(anchorHash) {
    // Phase 1: Ethereum mainnet or Polkadot
    // Phase 2+: Layer 2 or zkRollup
    
    const tx = await blockchainClient.publishAnchor({
      data: anchorHash,
      from: getUserAddress(),
      gas: 100000
    });
    
    await tx.wait(); // Wait for confirmation
    
    return {
      chain_id: 'ethereum',
      transaction_hash: tx.hash,
      block_number: tx.blockNumber,
      block_timestamp: tx.timestamp,
      anchor_data: anchorHash
    };
  }
}
```

---

# PART II: ACTIVE IMPLEMENTATION (User-Initiated Operations)

## 7. Friendship Management

### 7.1 In-Person Friend Request Flow

```javascript
class FriendshipManager {
  // User A initiates request
  async createFriendRequest(targetDID) {
    // Generate ephemeral ECDH keypair
    const ephemeralKeypair = await this._generateECDHKeypair();
    
    const request = {
      request_id: generateHash(randomBytes(32)),
      requester_did: await getCurrentUserDID(),
      target_did: targetDID,
      ephemeral_public_key: ephemeralKeypair.publicKey,
      created_at: Date.now(),
      expires_at: Date.now() + (7 * 86400 * 1000), // 7 days
      verification_code: this._generateVerificationCode(), // 8 digits + checksum
      proposed_type: 'FRIENDSHIP_CLOSE'
    };
    
    // Sign request
    request.requester_signature = await signMessage(
      await hashRequest(request),
      await getUserPrivateKey()
    );
    
    // Store ephemeral private key temporarily (for shared secret derivation)
    // CRITICAL: Will be zeroized after shared secret derivation
    await this._storeEphemeralKey(request.request_id, ephemeralKeypair.privateKey);
    
    return request;
  }
  
  // Generate 8-digit code with checksum for in-person verification
  _generateVerificationCode() {
    const random = Math.floor(10000000 + Math.random() * 90000000);
    const checksum = random % 10;
    return random.toString() + checksum.toString();
  }
  
  // Generate QR code for easy scanning
  async generateQRCode(request) {
    // Encode request as compact JSON
    const payload = {
      rid: base64url(request.request_id),
      req: request.requester_did,
      epk: base64url(request.ephemeral_public_key),
      vc: request.verification_code,
      exp: request.expires_at,
      sig: base64url(request.requester_signature)
    };
    
    // Generate QR code
    return await QRCode.generate(JSON.stringify(payload), {
      errorCorrectionLevel: 'M',
      type: 'svg',
      width: 300
    });
  }
  
  // User B scans QR and accepts
  async acceptFriendRequest(scannedRequest) {
    // Verify request signature
    const isValid = await verifySignature(
      scannedRequest.requester_signature,
      scannedRequest.requester_did,
      await hashRequest(scannedRequest)
    );
    
    if (!isValid) {
      throw new Error(ErrorCode.VERIFICATION_FAILED, 'Invalid request signature');
    }
    
    // Verify not expired
    if (Date.now() > scannedRequest.expires_at) {
      throw new Error(ErrorCode.FRIENDSHIP_EXPIRED, 'Request expired');
    }
    
    // Prompt user for verification code (in-person verification)
    const userEnteredCode = await promptForVerificationCode();
    if (userEnteredCode !== scannedRequest.verification_code) {
      throw new Error(ErrorCode.VERIFICATION_CODE_MISMATCH, 'Verification code mismatch');
    }
    
    // Generate own ephemeral keypair
    const ephemeralKeypair = await this._generateECDHKeypair();
    
    // Compute shared secret
    const sharedSecret = await this._computeECDH(
      ephemeralKeypair.privateKey,
      scannedRequest.ephemeral_public_key
    );
    
    // Compute commitment
    const commitment = await hashBytes(sharedSecret);
    
    // CRITICAL: Zeroize shared secret immediately
    zeroize(sharedSecret);
    
    // CRITICAL: Zeroize ephemeral private key
    zeroize(ephemeralKeypair.privateKey);
    
    // Create response
    const response = {
      request_id: scannedRequest.request_id,
      accepted: true,
      ephemeral_public_key: ephemeralKeypair.publicKey,
      responded_at: Date.now()
    };
    
    response.acceptor_signature = await signMessage(
      await hashResponse(response),
      await getUserPrivateKey()
    );
    
    // Store friendship credential locally
    const credential = {
      credential_id: generateHash(randomBytes(32)),
      participant_a_did: scannedRequest.requester_did,
      participant_b_did: await getCurrentUserDID(),
      shared_secret_commitment: commitment,
      established_at: Date.now(),
      type: 'FRIENDSHIP_CLOSE',
      participant_a_signature: scannedRequest.requester_signature,
      participant_b_signature: response.acceptor_signature
    };
    
    // Encrypt commitment with device-bound key for storage
    await storeFriendshipCredential(credential);
    
    // Send response to requester
    await sendFriendshipResponse(response);
    
    return credential;
  }
  
  async _generateECDHKeypair() {
    // Generate Curve25519 keypair
    const keypair = await crypto.subtle.generateKey(
      {
        name: 'X25519',
        namedCurve: 'X25519'
      },
      true,
      ['deriveBits']
    );
    
    return keypair;
  }
  
  async _computeECDH(privateKey, publicKey) {
    // Derive shared secret using X25519
    const sharedSecret = await crypto.subtle.deriveBits(
      {
        name: 'ECDH',
        public: publicKey
      },
      privateKey,
      256 // 32 bytes
    );
    
    return new Uint8Array(sharedSecret);
  }
}

// Zeroization helper
function zeroize(buffer) {
  if (buffer instanceof Uint8Array) {
    buffer.fill(0);
  } else if (buffer instanceof ArrayBuffer) {
    new Uint8Array(buffer).fill(0);
  } else if (typeof buffer === 'object' && buffer.privateKey) {
    // Handle CryptoKey objects - mark for garbage collection
    buffer.privateKey = null;
  }
}
```

### 7.2 Friendship Storage

```javascript
async function storeFriendshipCredential(credential) {
  // Encrypt shared_secret_commitment with device-bound key
  const encryptionKey = await getDeviceEncryptionKey();
  const encrypted = await encryptData(credential.shared_secret_commitment, encryptionKey);
  
  await db.execute(`
    INSERT INTO friendships (
      credential_id,
      participant_a_did,
      participant_b_did,
      shared_secret_encrypted,
      shared_secret_commitment,
      established_at,
      expires_at,
      friendship_type,
      participant_a_signature,
      participant_b_signature
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `, [
    credential.credential_id,
    credential.participant_a_did,
    credential.participant_b_did,
    encrypted.ciphertext,
    credential.shared_secret_commitment,
    credential.established_at,
    credential.expires_at || null,
    credential.type,
    credential.participant_a_signature,
    credential.participant_b_signature
  ]);
}
```

---

## 8. Attestation Batching Strategies

### 8.1 Batch Creation

```javascript
class AttestationBatcher {
  constructor(config = {}) {
    this.maxBatchSize = config.maxBatchSize || 20;
    this.minBatchThreshold = config.minBatchThreshold || 3;
    this.batchWindowMs = config.batchWindowMs || 300000; // 5 minutes
    this.pendingAttestations = [];
    this.batchTimer = null;
  }
  
  async addAttestation(attestation) {
    this.pendingAttestations.push(attestation);
    
    // Check if we should flush immediately
    if (this.pendingAttestations.length >= this.maxBatchSize) {
      await this.flushBatch();
    } else if (this.pendingAttestations.length === 1) {
      // Start batch window timer
      this.batchTimer = setTimeout(
        () => this.flushBatch(),
        this.batchWindowMs
      );
    }
  }
  
  async flushBatch() {
    if (this.batchTimer) {
      clearTimeout(this.batchTimer);
      this.batchTimer = null;
    }
    
    if (this.pendingAttestations.length < this.minBatchThreshold) {
      console.log('Batch too small, waiting for more attestations');
      return;
    }
    
    const batch = this.pendingAttestations.splice(0, this.maxBatchSize);
    
    try {
      const batchAttestation = await this._createBatch(batch);
      await this._publishBatch(batchAttestation);
      
      // Record battery usage
      await batteryTracker.recordUsage('batch_attestation', 0.05);
    } catch (error) {
      console.error('Failed to create batch:', error);
      // Re-add attestations to queue
      this.pendingAttestations.unshift(...batch);
    }
  }
  
  async _createBatch(attestations) {
    const batchId = generateHash(randomBytes(32));
    const attesterDID = await getCurrentUserDID();
    
    // Create batch entries
    const entries = attestations.map(att => ({
      attestation_id: att.attestation_id,
      claim_id: att.claim_id,
      claimant_did: att.claimant_did,
      attester_visit_commitment: att.attester_visit_commitment,
      overlap_proof: att.overlap_proof
    }));
    
    // Compute Merkle tree of attestation_ids
    const attestationIds = entries.map(e => e.attestation_id);
    const merkleTree = buildMerkleTree(attestationIds);
    const attestationsMerkleRoot = merkleTree.root;
    
    // Generate Merkle proof for each entry
    entries.forEach((entry, i) => {
      entry.merkle_proof = merkleTree.getProof(i);
    });
    
    // Create batch
    const batch = {
      batch_id: batchId,
      attester_did: attesterDID,
      entries: entries,
      attestations_merkle_root: attestationsMerkleRoot,
      created_at: Date.now(),
      batch_size: entries.length
    };
    
    // Sign batch
    batch.batch_signature = await signMessage(
      await hashMessage(batch),
      await getUserPrivateKey()
    );
    
    return batch;
  }
  
  async _publishBatch(batch) {
    // Send to attestation server with exponential backoff
    let retries = 0;
    const maxRetries = 3;
    
    while (retries < maxRetries) {
      try {
        const response = await fetch('/api/attestations/batch', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(batch)
        });
        
        if (response.ok) {
          console.log(`Published batch ${batch.batch_id} with ${batch.batch_size} attestations`);
          return;
        } else {
          throw new Error(`Server returned ${response.status}`);
        }
      } catch (error) {
        retries++;
        if (retries >= maxRetries) {
          throw error;
        }
        
        // Exponential backoff
        const delay = Math.pow(2, retries) * 1000;
        console.warn(`Batch publish failed, retrying in ${delay}ms...`);
        await sleep(delay);
      }
    }
  }
}
```

### 8.2 Attestation Request Processing

```javascript
class AttestationProcessor {
  async processAttestationRequest(request) {
    // Verify request signature
    const valid = await verifySignature(
      request.claimant_signature,
      request.claimant_did,
      await hashMessage(request)
    );
    
    if (!valid) {
      throw new Error('Invalid attestation request signature');
    }
    
    // Check if expired
    if (Date.now() > request.expires_at) {
      throw new Error(ErrorCode.ATTESTATION_EXPIRED);
    }
    
    // Check friendship requirement
    if (request.requirements.require_friendship) {
      const friendship = await findFriendship(
        await getCurrentUserDID(),
        request.claimant_did
      );
      
      if (!friendship) {
        throw new Error('Friendship required but not found');
      }
    }
    
    // Find matching visit from our history
    const matchingVisit = await this._findMatchingVisit(
      request.claim_id,
      request.requirements
    );
    
    if (!matchingVisit) {
      throw new Error(ErrorCode.OVERLAP_INSUFFICIENT);
    }
    
    // Create attestation
    const attestation = await this._createAttestation(
      request,
      matchingVisit
    );
    
    // Add to batch
    await attestationBatcher.addAttestation(attestation);
    
    return attestation;
  }
  
  async _findMatchingVisit(claimId, requirements) {
    // Get claim details
    const claim = await fetchClaim(claimId);
    
    // Search our visits for spatial-temporal overlap
    const ourVisits = await db.query(`
      SELECT * FROM visits
      WHERE arrival_time <= ? AND departure_time >= ?
    `, [claim.visit_end, claim.visit_start]);
    
    for (const visit of ourVisits) {
      const decrypted = await decryptVisit(visit);
      
      // Check spatial overlap
      const distance = haversineDistance(
        decrypted.center,
        claim.center
      );
      
      if (distance > requirements.max_distance_meters) {
        continue;
      }
      
      // Check temporal overlap
      const overlapStart = Math.max(
        decrypted.arrival_time,
        claim.visit_start
      );
      const overlapEnd = Math.min(
        decrypted.departure_time,
        claim.visit_end
      );
      const overlapSeconds = overlapEnd - overlapStart;
      
      if (overlapSeconds < requirements.min_time_overlap_seconds) {
        continue;
      }
      
      // Found match
      return {
        visit: decrypted,
        distance,
        overlapSeconds
      };
    }
    
    return null;
  }
  
  async _createAttestation(request, matchingVisit) {
    const attestationId = generateHash(randomBytes(32));
    
    // Create friendship proof
    const friendshipProof = await this._createFriendshipProof(
      request.claimant_did
    );
    
    // Create spatial-temporal proof
    const overlapProof = {
      commitment: {
        attester_center: matchingVisit.visit.center,
        attester_radius: matchingVisit.visit.radius,
        attester_arrival: matchingVisit.visit.arrival_time,
        attester_departure: matchingVisit.visit.departure_time,
        distance_meters: matchingVisit.distance,
        time_overlap_seconds: matchingVisit.overlapSeconds,
        attester_visit_commitment: matchingVisit.visit.visit_commitment
      }
    };
    
    // Create device fingerprint
    const deviceFingerprint = await createDeviceFingerprint();
    
    // Assemble attestation
    const attestation = {
      attestation_id: attestationId,
      request_id: request.request_id,
      claim_id: request.claim_id,
      attester_did: await getCurrentUserDID(),
      attester_visit_id: matchingVisit.visit.visit_id,
      attester_visit_commitment: matchingVisit.visit.visit_commitment,
      friendship_proof: friendshipProof,
      overlap_proof: overlapProof,
      device_fingerprint: deviceFingerprint,
      created_at: Date.now()
    };
    
    // Sign attestation
    attestation.attester_signature = await signMessage(
      await hashMessage(attestation),
      await getUserPrivateKey()
    );
    
    return attestation;
  }
}
```

---

## 9. Device Fingerprinting

### 9.1 Comprehensive Fingerprint Collection

```javascript
class DeviceFingerprintCollector {
  async collectFingerprint() {
    const fingerprint = {
      os: await this._getOS(),
      os_version_hash: await hashString(await this._getOSVersion()),
      device_model_hash: await hashString(await this._getDeviceModel()),
      app_version: await this._getAppVersion(),
      installation_id: await this._getInstallationId(),
      
      // Additional fraud detection fields
      is_tampered: await this._detectTampering(),
      is_emulator: await this._detectEmulator(),
      is_rooted: await this._detectRootJailbreak(),
      battery_level: await getBatteryLevel(),
      screen_resolution: await this._getScreenResolution(),
      timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
      language: navigator.language || navigator.userLanguage
    };
    
    return fingerprint;
  }
  
  async _detectTampering() {
    // Check for signs of app tampering
    const checks = [
      await this._checkCodeSignature(),
      await this._checkDebugger(),
      await this._checkHooks()
    ];
    
    return checks.some(c => c === true);
  }
  
  async _detectEmulator() {
    // Platform-specific emulator detection
    if (this._getOS() === 'Android') {
      return await this._detectAndroidEmulator();
    } else if (this._getOS() === 'iOS') {
      return await this._detectIOSSimulator();
    }
    return false;
  }
  
  async _detectAndroidEmulator() {
    // Check for known emulator properties
    const emulatorSignals = [
      'generic' in navigator.userAgent.toLowerCase(),
      window.navigator.hardwareConcurrency === 2, // Common in emulators
      !('ontouchstart' in window) // No touch support
    ];
    
    return emulatorSignals.filter(s => s).length >= 2;
  }
  
  async _detectIOSSimulator() {
    // Check for simulator-specific properties
    // (Implementation depends on available APIs)
    return false; // Placeholder
  }
  
  async _detectRootJailbreak() {
    // Platform-specific root/jailbreak detection
    if (this._getOS() === 'Android') {
      return await this._detectAndroidRoot();
    } else if (this._getOS() === 'iOS') {
      return await this._detectIOSJailbreak();
    }
    return false;
  }
  
  async _detectAndroidRoot() {
    // Check for common root indicators
    const rootIndicators = [
      '/system/app/Superuser.apk',
      '/sbin/su',
      '/system/bin/su',
      '/system/xbin/su',
      '/data/local/xbin/su',
      '/data/local/bin/su',
      '/system/sd/xbin/su',
      '/system/bin/failsafe/su',
      '/data/local/su'
    ];
    
    // Try to detect if any root files exist
    // (Implementation depends on available APIs)
    return false; // Placeholder
  }
  
  async _detectIOSJailbreak() {
    // Check for common jailbreak indicators
    const jailbreakIndicators = [
      '/Applications/Cydia.app',
      '/Library/MobileSubstrate/MobileSubstrate.dylib',
      '/bin/bash',
      '/usr/sbin/sshd',
      '/etc/apt',
      '/private/var/lib/apt/',
      '/private/var/lib/cydia',
      '/private/var/mobile/Library/SBSettings/Themes',
      '/private/var/stash',
      '/private/var/tmp/cydia.log',
      '/System/Library/LaunchDaemons/com.ikey.bbot.plist',
      '/System/Library/LaunchDaemons/com.saurik.Cydia.Startup.plist'
    ];
    
    // (Implementation depends on available APIs)
    return false; // Placeholder
  }
}
```

---

## 10. Fraud Detection Heuristics

### 10.1 Co-Location Pattern Analysis

```javascript
class FraudDetector {
  async analyzeCoLocationPatterns(userDID) {
    // Get all attestations from this user
    const attestations = await getAllAttestations(userDID);
    
    // Build co-location graph
    const coLocationGraph = new Map();
    
    for (const att of attestations) {
      const key = `${att.claimant_did},${att.attester_did}`;
      if (!coLocationGraph.has(key)) {
        coLocationGraph.set(key, []);
      }
      coLocationGraph.get(key).push(att);
    }
    
    // Detect suspicious patterns
    const suspiciousPatterns = [];
    
    for (const [pair, atts] of coLocationGraph.entries()) {
      // Pattern 1: Too many attestations in short time
      if (atts.length > 50) {
        const timespan = atts[atts.length - 1].created_at - atts[0].created_at;
        const daysSpan = timespan / (86400 * 1000);
        
        if (daysSpan < 30) {
          suspiciousPatterns.push({
            type: 'excessive_attestations',
            pair,
            count: atts.length,
            dayspan: daysSpan,
            rate: atts.length / daysSpan
          });
        }
      }
      
      // Pattern 2: Always perfect overlap
      const perfectOverlaps = atts.filter(att => 
        att.overlap_proof.time_overlap_seconds > 3600 && // > 1 hour
        att.overlap_proof.distance_meters < 10 // < 10m
      );
      
      if (perfectOverlaps.length / atts.length > 0.8) {
        suspiciousPatterns.push({
          type: 'unnaturally_perfect_overlap',
          pair,
          perfect_ratio: perfectOverlaps.length / atts.length
        });
      }
      
      // Pattern 3: Attestations at impossible times
      const impossibleMovements = await this._detectImpossibleMovement(atts);
      if (impossibleMovements.length > 0) {
        suspiciousPatterns.push({
          type: 'impossible_movement',
          pair,
          instances: impossibleMovements
        });
      }
    }
    
    return suspiciousPatterns;
  }
  
  async _detectImpossibleMovement(attestations) {
    // Sort by time
    const sorted = attestations.sort((a, b) => 
      a.created_at - b.created_at
    );
    
    const impossible = [];
    
    for (let i = 0; i < sorted.length - 1; i++) {
      const curr = sorted[i];
      const next = sorted[i + 1];
      
      // Calculate distance and time between attestations
      const distance = haversineDistance(
        curr.overlap_proof.attester_center,
        next.overlap_proof.attester_center
      );
      
      const timeDiff = (next.created_at - curr.created_at) / 1000; // seconds
      
      // Calculate required speed
      const requiredSpeed = distance / timeDiff; // m/s
      
      // Max realistic speed: 250 m/s (900 km/h, roughly airplane speed)
      if (requiredSpeed > 250) {
        impossible.push({
          from: curr.attestation_id,
          to: next.attestation_id,
          distance,
          time: timeDiff,
          required_speed: requiredSpeed
        });
      }
    }
    
    return impossible;
  }
  
  async analyzeFriendshipGraph(userDID) {
    // Get all friendships
    const friendships = await getAllFriendships(userDID);
    
    // Build graph
    const graph = new Map();
    
    for (const friendship of friendships) {
      const a = friendship.participant_a_did;
      const b = friendship.participant_b_did;
      
      if (!graph.has(a)) graph.set(a, new Set());
      if (!graph.has(b)) graph.set(b, new Set());
      
      graph.get(a).add(b);
      graph.get(b).add(a);
    }
    
    // Detect suspicious patterns
    const suspiciousPatterns = [];
    
    // Pattern 1: Too many friends too quickly
    const recentFriendships = friendships.filter(f => 
      Date.now() - f.established_at < 7 * 86400 * 1000 // Last 7 days
    );
    
    if (recentFriendships.length > 20) {
      suspiciousPatterns.push({
        type: 'rapid_friend_acquisition',
        count: recentFriendships.length,
        timespan: 7
      });
    }
    
    // Pattern 2: Star topology (one user friends with many, but they don't know each other)
    if (graph.has(userDID)) {
      const userFriends = Array.from(graph.get(userDID));
      let interconnections = 0;
      
      for (let i = 0; i < userFriends.length; i++) {
        for (let j = i + 1; j < userFriends.length; j++) {
          if (graph.get(userFriends[i])?.has(userFriends[j])) {
            interconnections++;
          }
        }
      }
      
      const maxPossible = (userFriends.length * (userFriends.length - 1)) / 2;
      const interconnectionRatio = maxPossible > 0 ? interconnections / maxPossible : 0;
      
      if (userFriends.length > 10 && interconnectionRatio < 0.1) {
        suspiciousPatterns.push({
          type: 'star_topology',
          friend_count: userFriends.length,
          interconnection_ratio: interconnectionRatio
        });
      }
    }
    
    return suspiciousPatterns;
  }
}
```

---

## 11. User Experience Patterns

### 11.1 Progressive Permission Requests

```javascript
class PermissionManager {
  async requestPermissionsProgressively() {
    // Step 1: Explain value proposition first
    await showOnboarding({
      title: 'Own Your Location Data',
      message: 'Olocus lets you control and monetize your location history.',
      cta: 'Get Started'
    });
    
    // Step 2: Request basic permissions
    const locationPermission = await this._requestLocationPermission();
    if (!locationPermission) {
      await showPermissionDeniedMessage('location');
      return false;
    }
    
    // Step 3: Request background location (iOS requires separate request)
    if (platform === 'iOS') {
      const backgroundPermission = await this._requestBackgroundLocationPermission();
      if (!backgroundPermission) {
        await showPermissionDeniedMessage('background_location');
        return false;
      }
    }
    
    // Step 4: Request motion & fitness (for better motion detection)
    const motionPermission = await this._requestMotionPermission();
    // Non-blocking, continue even if denied
    
    // Step 5: Request notifications (for friend requests)
    const notificationPermission = await this._requestNotificationPermission();
    // Non-blocking
    
    return true;
  }
  
  async _requestLocationPermission() {
    // Show educational screen first
    await showEducationalScreen({
      title: 'Location Access',
      message: 'Olocus needs location access to track your visits and create verifiable location credentials.',
      benefits: [
        'Monetize your location data',
        'Prove you were at important places',
        'Control who sees your location'
      ]
    });
    
    // Request permission
    const result = await navigator.permissions.query({ name: 'geolocation' });
    
    if (result.state === 'prompt') {
      // Will trigger browser permission dialog
      return new Promise((resolve) => {
        navigator.geolocation.getCurrentPosition(
          () => resolve(true),
          () => resolve(false)
        );
      });
    }
    
    return result.state === 'granted';
  }
}
```

### 11.2 Battery Usage Transparency

```javascript
class BatteryUsageUI {
  async showBatteryUsageReport() {
    const usage = await getBatteryUsage();
    
    const report = {
      daily_budget: 9.5,
      current_usage: usage.totalPercent,
      remaining_budget: 9.5 - usage.totalPercent,
      breakdown: [
        { category: 'Location Tracking', percent: usage.locationTracking },
        { category: 'Visit Detection', percent: usage.visitDetection },
        { category: 'Anchoring', percent: usage.anchoring },
        { category: 'Network Sync', percent: usage.networkSync },
        { category: 'Attestations', percent: usage.attestations },
        { category: 'Other', percent: usage.other }
      ],
      optimization_tips: this._getOptimizationTips(usage)
    };
    
    await showBatteryReport(report);
  }
  
  _getOptimizationTips(usage) {
    const tips = [];
    
    if (usage.locationTracking > 5.0) {
      tips.push({
        title: 'Reduce Location Accuracy',
        message: 'Switch to balanced accuracy mode to save battery',
        action: 'adjust_sampling'
      });
    }
    
    if (usage.networkSync > 1.0) {
      tips.push({
        title: 'Sync Only on WiFi',
        message: 'Disable cellular sync to reduce battery usage',
        action: 'wifi_only_sync'
      });
    }
    
    if (usage.totalPercent > 9.5) {
      tips.push({
        title: 'Enable Ultra Low Mode',
        message: 'Temporarily reduce tracking frequency',
        action: 'enable_ultra_low'
      });
    }
    
    return tips;
  }
}
```

---

# PART III: CROSS-PLATFORM IMPLEMENTATION

## 12. Platform-Specific Guidance

### 12.1 iOS Considerations

**Background Location Best Practices**:
- Use `startMonitoringSignificantLocationChanges()` as baseline
- Use `startMonitoringVisits()` for visit detection hints
- Only use `startUpdatingLocation()` when higher accuracy needed
- Request "Always" authorization, not just "When In Use"
- Provide clear rationale in Info.plist

**Battery Optimization**:
- Use `allowsBackgroundLocationUpdates = true` sparingly
- Set `pausesLocationUpdatesAutomatically = false` only when necessary
- Implement region monitoring for known frequent locations
- Use `CLLocationManager.deferredLocationUpdatesAvailable()` when possible

**Key Storage**:
- Use Secure Enclave for private keys
- Mark keys as `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`
- Never export private keys

### 12.2 Android Considerations

**Background Location Best Practices**:
- Use `FusedLocationProviderClient` for efficiency
- Implement `LocationRequest` with batching (`maxWaitTime`)
- Use `JobScheduler` or `WorkManager` for periodic tasks
- Request `ACCESS_BACKGROUND_LOCATION` permission (Android 10+)

**Battery Optimization**:
- Respect Doze mode and App Standby
- Use `setFastestInterval()` to control update rate
- Implement geofencing for known locations
- Use `PRIORITY_BALANCED_POWER_ACCURACY` by default

**Key Storage**:
- Use Android Keystore for private keys
- Mark keys with `KeyProperties.PURPOSE_SIGN`
- Use StrongBox when available (Android 9+)

### 12.3 Web Considerations

**Limitations**:
- No true background tracking
- Less accurate location data
- No motion/activity detection
- Limited battery API

**Recommended Approach**:
- Use `navigator.geolocation.watchPosition()` when tab is active
- Implement Service Worker for background sync
- Store data in IndexedDB
- Sync to server more frequently

---

## 13. Network Synchronization

### 13.1 Efficient Sync Strategy

```javascript
class SyncManager {
  async syncWithServer() {
    // Check network conditions
    const networkType = await getNetworkType();
    const batteryLevel = await getBatteryLevel();
    
    // Only sync on WiFi if battery < 30%
    if (batteryLevel < 0.30 && networkType !== 'wifi') {
      console.log('Deferring sync until WiFi available');
      return;
    }
    
    // Sync anchors (small, high priority)
    await this._syncAnchors();
    
    // Sync batch attestations (medium priority)
    if (networkType === 'wifi' || batteryLevel > 0.50) {
      await this._syncBatchAttestations();
    }
    
    // Sync marketplace listings (low priority, WiFi only)
    if (networkType === 'wifi') {
      await this._syncMarketplaceListings();
    }
  }
  
  async _syncAnchors() {
    // Upload pending anchors
    const pendingAnchors = await db.query(`
      SELECT * FROM daily_anchors WHERE synced = 0
    `);
    
    for (const anchor of pendingAnchors) {
      try {
        await this._uploadAnchor(anchor);
        await db.execute(`
          UPDATE daily_anchors SET synced = 1 WHERE anchor_id = ?
        `, [anchor.anchor_id]);
      } catch (error) {
        console.error(`Failed to sync anchor ${anchor.anchor_id}:`, error);
      }
    }
  }
}
```

---

## 14. Performance Optimization

### 14.1 Benchmarking Targets

| Operation | Target Time | Max Acceptable |
|-----------|-------------|----------------|
| Create LocationBlock | < 50ms | 100ms |
| Encrypt LocationBlock | < 20ms | 50ms |
| Detect Visit (100 blocks) | < 500ms | 1000ms |
| Create DailyAnchor | < 2s | 5s |
| Verify Attestation | < 100ms | 200ms |
| Create Batch (20 attestations) | < 1s | 2s |

### 14.2 Optimization Techniques

**Database Indexing**:
```sql
-- Critical indexes
CREATE INDEX idx_blocks_chain_timestamp ON location_blocks(chain_id, timestamp);
CREATE INDEX idx_visits_chain_time ON visits(chain_id, arrival_time);
CREATE INDEX idx_visits_commitment ON visits(visit_commitment);
```

**Batch Operations**:
```javascript
// Instead of individual inserts
for (const block of blocks) {
  await db.execute('INSERT INTO location_blocks ...');
}

// Use batch insert
await db.executeBatch(`
  INSERT INTO location_blocks (block_hash, ciphertext, ...)
  VALUES (?, ?), (?, ?), ...
`, flattenedValues);
```

**Lazy Loading**:
```javascript
// Don't decrypt all visits at once
async function getVisitSummaries() {
  // Only load metadata
  const summaries = await db.query(`
    SELECT visit_id, visit_commitment, arrival_time, departure_time
    FROM visits
  `);
  return summaries;
}

// Decrypt on demand
async function getVisitDetails(visitId) {
  const encrypted = await db.queryOne(`
    SELECT * FROM visits WHERE visit_id = ?
  `, [visitId]);
  
  return await decryptVisit(encrypted);
}
```

---

## 15. Testing & Validation

### 15.1 Unit Testing

```javascript
describe('LocationBlock Creation', () => {
  it('should create valid LocationBlock with correct hash', async () => {
    const block = await createLocationBlock({
      coordinates: { latitude: 37.7749, longitude: -122.4194 },
      timestamp: Timestamp.now(),
      motionState: MotionState.MOTION_STATIONARY
    });
    
    // Verify hash
    const expectedHash = await computeBlockHash(block);
    expect(block.hash).toEqual(expectedHash);
    
    // Verify signature
    const valid = await verifySignature(
      block.hash,
      block.signature,
      publicKey
    );
    expect(valid).toBe(true);
  });
});

describe('Visit Detection', () => {
  it('should cluster nearby blocks into visit', async () => {
    const blocks = generateTestBlocks({
      center: { lat: 37.7749, lon: -122.4194 },
      radius: 30, // meters
      count: 10,
      duration: 3600 // 1 hour
    });
    
    const visits = await detectVisits(blocks);
    
    expect(visits).toHaveLength(1);
    expect(visits[0].radius_meters).toBeLessThan(50);
    expect(visits[0].duration_seconds).toBeGreaterThan(3000);
  });
});
```

### 15.2 Integration Testing

```javascript
describe('End-to-End Friendship Flow', () => {
  it('should establish friendship and create attestation', async () => {
    // User A creates request
    const requestA = await userA.createFriendRequest(userB.did);
    
    // User B accepts
    const credentialB = await userB.acceptFriendRequest(requestA);
    
    // User A processes response
    const credentialA = await userA.processFriendshipResponse(credentialB);
    
    // Verify both have same commitment
    expect(credentialA.shared_secret_commitment).toEqual(
      credentialB.shared_secret_commitment
    );
    
    // Create overlapping visits
    await userA.createVisit({ lat: 37.7749, lon: -122.4194, duration: 3600 });
    await userB.createVisit({ lat: 37.7748, lon: -122.4195, duration: 3600 });
    
    // User A creates claim
    const claim = await userA.createLocationCredential(visitA.visit_id);
    
    // User A requests attestation from B
    const attRequest = await userA.requestAttestation(claim.id, userB.did);
    
    // User B creates attestation
    const attestation = await userB.createAttestation(attRequest);
    
    // Verify attestation
    const valid = await verifyAttestation(attestation, claim);
    expect(valid).toBe(true);
  });
});
```

---

## 16. Troubleshooting

### 16.1 Common Issues

**Issue: High battery drain**
- Check sampling frequency is adaptive
- Verify battery preservation mode is working
- Check for stuck GPS locks
- Review battery usage logs

**Issue: Visits not detected**
- Verify DBSCAN parameters (epsilon, min_samples)
- Check if enough blocks collected
- Verify location accuracy is sufficient
- Check for gaps in location data

**Issue: Anchoring fails**
- Check TSA server availability
- Verify network connectivity
- Check time synchronization
- Verify NTP accuracy

**Issue: Attestation requests timeout**
- Check network latency
- Verify peer is online
- Check batch window settings
- Verify friend connection exists

### 16.2 Debugging Tools

```javascript
class DebugLogger {
  async dumpState() {
    const state = {
      chain: await this._getChainState(),
      blocks: await this._getBlockStats(),
      visits: await this._getVisitStats(),
      anchors: await this._getAnchorStats(),
      battery: await this._getBatteryStats(),
      network: await this._getNetworkStats()
    };
    
    console.log(JSON.stringify(state, null, 2));
    return state;
  }
  
  async _getChainState() {
    const chain = await db.queryOne('SELECT * FROM chains LIMIT 1');
    return {
      chain_id: chain.chain_id.toString('hex'),
      current_head: chain.current_head.toString('hex'),
      last_anchor: chain.last_anchor_timestamp
    };
  }
  
  async _getBlockStats() {
    const stats = await db.queryOne(`
      SELECT
        COUNT(*) as total,
        MIN(timestamp) as oldest,
        MAX(timestamp) as newest
      FROM location_blocks
    `);
    
    return {
      total_blocks: stats.total,
      oldest: new Date(stats.oldest),
      newest: new Date(stats.newest),
      span_days: (stats.newest - stats.oldest) / 86400000
    };
  }
}
```

---

**End of Olocus Protocol Implementation Guide v1.0**
