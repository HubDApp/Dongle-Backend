# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned Features

- [ ] Complete API implementation for all endpoints
- [ ] PostgreSQL database schema and migrations
- [ ] Soroban event indexing service
- [ ] JWT authentication middleware
- [ ] Admin dashboard backend
- [ ] WebSocket support for real-time updates
- [ ] Redis caching layer
- [ ] Comprehensive test suite (>80% coverage)
- [ ] Sentry error tracking integration
- [ ] Production deployment infrastructure

### In Progress

- [ ] Project service implementation
- [ ] Review aggregation service
- [ ] Verification request workflow
- [ ] User profile management
- [ ] Database schema design

## [0.1-alpha] - 2024-11-06

### Added

- Initial project scaffold and documentation
- Architecture design document
- API specification and endpoint reference
- Database schema design
- Environment configuration template (.env.example)
- Contributing guidelines template
- Build status tracking (BUILD_STATUS.md)

### Notes

- This is an **alpha release** — not production-ready
- Core architecture designed but implementation not yet complete
- Database layer not yet implemented
- Soroban indexing service planned
- No deployments to mainnet recommended

---

## How to Contribute

When making changes, please:

1. Update this CHANGELOG.md following the Keep a Changelog format
2. Use [Conventional Commits](https://www.conventionalcommits.org/) for commit messages:
   - `feat:` for new features
   - `fix:` for bug fixes
   - `docs:` for documentation changes
   - `refactor:` for code refactoring
   - `test:` for test additions/changes
   - `perf:` for performance improvements
   - `chore:` for build/dependency updates
   - `security:` for security fixes

3. Link to related GitHub issues/PRs in the commit message

### Example Entry Format

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- Brief description of new features

### Changed
- Brief description of changes to existing features

### Fixed
- Brief description of bug fixes

### Deprecated
- Brief description of soon-to-be removed features

### Removed
- Brief description of removed features

### Security
- Brief description of security fixes
```

## Release Process

1. Update version in `package.json`
2. Update CHANGELOG.md with new version section
3. Create git tag: `git tag vX.Y.Z`
4. Push changes and tag: `git push origin main --tags`
5. Create GitHub Release from tag with CHANGELOG.md entry

---

**See** [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md) for detailed contribution guidelines.
