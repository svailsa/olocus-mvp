# OwnTracks-Inspired Patterns

This directory documents patterns from OwnTracks that we're adopting (reimplemented, not copied) for the Olocus Protocol.

## Documents

### Priority 3
- **[owntracks_analysis.md](owntracks_analysis.md)** - Patterns we're adopting vs. avoiding
- **[battery_optimization.md](battery_optimization.md)** - Motion-adaptive sampling patterns
- **[background_resilience.md](background_resilience.md)** - Task scheduling patterns
- **[permission_ux.md](permission_ux.md)** - Onboarding best practices
- **[implementation_mapping.md](implementation_mapping.md)** - OwnTracks concepts → Olocus implementation

## Key Patterns to Adopt

### 1. Motion-Adaptive Sampling
- Use motion activity recognition to adjust sampling frequency
- Stationary: 5-minute intervals
- Walking: 1-minute intervals
- Vehicle: 2-minute intervals
- Running: 45-second intervals

### 2. Battery Preservation Modes
- Normal (>30% battery): Full accuracy
- Low (20-30% battery): Disable GPS, double intervals
- Critical (<20% battery): Cell tower only, quadruple intervals
- Ultra-low (<10% battery): Significant changes only

### 3. Background Task Resilience
- iOS: BGTaskScheduler for daily anchors
- Android: WorkManager for periodic sync
- Web: Service Worker for offline operation

### 4. Permission UX Flow
- Progressive disclosure
- Clear value proposition before request
- Settings deep-link for denied permissions
- Educational overlays

### 5. Significant Location Changes
- Use as fallback when battery critical
- Supplement with geofencing for visits
- Combine with motion detection

## Patterns to Avoid

### From OwnTracks (Don't Copy)
- Variable names and code structure
- Direct MQTT integration (we use different protocol)
- Manual waypoint system (we use automatic visit detection)
- Friend tracking UI (privacy-first approach)

### General Anti-Patterns
- Continuous high-accuracy GPS
- Synchronous network operations
- Unencrypted local storage
- Battery-draining background services

## Implementation Notes

All patterns must be:
1. **Reimplemented** from scratch in our codebase
2. **Adapted** to fit Olocus Protocol requirements
3. **Optimized** for our 8-10% battery budget
4. **Tested** against our performance benchmarks

## References
- [OwnTracks Documentation](https://owntracks.org/booklet/)
- [Implementation Guide](../implementation/)
- [Battery Testing](../testing/battery_testing.md)