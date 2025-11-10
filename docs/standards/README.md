# Development Standards

This directory contains coding standards and development guidelines for the Olocus Protocol.

## Documents

### Priority 1 (Required before development)
- **[coding_standards.md](coding_standards.md)** - Language-specific style guides (Swift, Kotlin, JS)
- **[git_workflow.md](git_workflow.md)** - Branch strategy, commit message conventions
- **[security_checklist.md](security_checklist.md)** - Pre-commit security requirements

### Priority 2
- **[testing_requirements.md](testing_requirements.md)** - Coverage requirements, test naming

## Quick Reference

### Git Commit Format
```
type(scope): subject

body

footer
```

Types: feat, fix, docs, style, refactor, test, chore

### Code Review Checklist
- [ ] Follows coding standards
- [ ] Includes tests
- [ ] Updates documentation
- [ ] Passes security checklist
- [ ] Battery impact assessed

### Language Standards
- **Swift**: Swift API Design Guidelines
- **Kotlin**: Kotlin Coding Conventions
- **JavaScript**: ESLint with Airbnb config
- **Protocol Buffers**: Google Style Guide

## Related Documentation
- [Testing Guide](../testing/)
- [Security Architecture](../architecture/security_architecture.md)
- [API Design](../api/)