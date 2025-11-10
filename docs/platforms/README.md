# Platform Implementation Guides

Platform-specific implementation guides for the Olocus Protocol.

## Platforms

### [iOS](ios/)
- Swift implementation
- Core Location and Core Motion integration
- Background modes and BGTaskScheduler
- Keychain security

### [Android](android/)
- Kotlin implementation
- Fused Location Provider
- Foreground services and WorkManager
- Android Keystore

### [Web](web/)
- JavaScript/TypeScript implementation
- Progressive Web App (PWA)
- Geolocation API
- IndexedDB storage

## Common Requirements

All platforms must:
1. Implement the full Olocus Protocol specification
2. Pass all test vectors
3. Maintain 8-10% daily battery usage
4. Support background operation
5. Provide secure key storage
6. Handle offline operation

## OwnTracks-Inspired Patterns

All platforms should adopt these patterns (reimplemented):
- Motion-adaptive sampling
- Battery preservation modes
- Background task resilience
- Progressive permission requests
- Significant location changes as fallback

## Related Documentation
- [Protocol Specification](../specification/)
- [Implementation Guide](../implementation/)
- [Test Vectors](../testing/test_vectors/)