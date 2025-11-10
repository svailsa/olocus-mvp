# Olocus Protocol Documentation

## Documentation Structure

### Core Documentation
- **[overview.md](overview.md)** - Business and technical overview
- **[specification/](specification/)** - Protocol specifications and release notes
- **[implementation/](implementation/)** - Implementation guides and best practices

### Technical Documentation
- **[architecture/](architecture/)** - System architecture and design documents
- **[api/](api/)** - API specifications and protocols
- **[standards/](standards/)** - Coding standards and development guidelines
- **[testing/](testing/)** - Test plans, vectors, and methodologies

### Platform Guides
- **[platforms/ios/](platforms/ios/)** - iOS implementation guide
- **[platforms/android/](platforms/android/)** - Android implementation guide
- **[platforms/web/](platforms/web/)** - Web/PWA implementation guide

### Patterns & Practices
- **[patterns/](patterns/)** - OwnTracks-inspired patterns and best practices
- **[ux/](ux/)** - User experience specifications

### Operational Documentation
- **[operations/](operations/)** - Deployment and operational procedures
- **[migration/](migration/)** - Version migration and compatibility
- **[legal/](legal/)** - Legal templates and compliance

## Documentation Status

See [TASKS.md](/TASKS.md) for current documentation task status and priorities.

## Quick Links

### Priority 1 - Must Read Before Development
1. [System Architecture](architecture/system_architecture.md)
2. [Database Schema](architecture/database_schema.md)
3. [OpenAPI Specification](api/openapi.yaml)
4. [Coding Standards](standards/coding_standards.md)
5. [Test Vectors](testing/test_vectors/)

### For Developers
- [iOS Setup Guide](platforms/ios/setup_guide.md)
- [Android Setup Guide](platforms/android/setup_guide.md)
- [Web Setup Guide](platforms/web/pwa_requirements.md)
- [Git Workflow](standards/git_workflow.md)

### For Security Review
- [Security Architecture](architecture/security_architecture.md)
- [Security Checklist](standards/security_checklist.md)
- [Privacy Compliance](operations/privacy_compliance.md)

## Contributing

Please ensure all documentation follows these principles:
1. **Clarity over completeness** - Clear, actionable documentation
2. **Examples over theory** - Include code snippets and scenarios
3. **Testable specifications** - Everything must be verifiable
4. **Platform-agnostic core** - Protocol works for any implementation

## Documentation Versioning

Current Version: **v1.0.0**

All documentation is versioned alongside the protocol specification. Breaking changes require a major version bump.