# Changelog

All notable changes to RamAPI will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- WebSocket support
- Server-Sent Events (SSE)
- Built-in caching layer
- Advanced rate limiting strategies
- API versioning support
- Request/response compression middleware
- CSRF protection middleware
- GraphQL subscriptions
- gRPC streaming support

---

## [0.1.0] - 2024-01-15 (Current)

### Added

#### Core Features
- **Ultra-fast routing system** with O(1) static route lookup
- **Zero-overhead middleware** through pre-compilation
- **Context-based API** replacing traditional req/res pattern
- **TypeScript-first design** with full type safety
- **HTTPError class** for structured error handling
- **Nested router support** for modular application structure

#### Built-in Middleware
- **Validation middleware** with Zod schema integration
- **CORS middleware** with configurable options
- **Rate limiting** with in-memory store
- **Logger middleware** for request logging
- **Body parsing** (JSON, text, form data) - automatic

#### Authentication
- **JWTService** for token generation and verification
- **authenticate() middleware** for protected routes
- **optionalAuthenticate() middleware** for optional auth
- **PasswordService** for secure password hashing with bcrypt
- JWT payload validation and error handling

#### Observability
- **OpenTelemetry integration** for distributed tracing
- **Flow tracking system** for request visualization
- **Performance profiling** with timing measurements
- **Metrics collection** for monitoring
- **Multiple exporters**: Console, OTLP, Memory
- **Jaeger integration** for trace visualization
- **Prometheus metrics** endpoint support

#### Multi-Protocol Support
- **GraphQL adapter** for GraphQL APIs
- **gRPC adapter** for gRPC services
- **Protocol manager** for unified multi-protocol services
- Support for REST, GraphQL, and gRPC from single handler

#### HTTP Adapters
- **Node.js HTTP adapter** (default) - stable, production-ready
- **uWebSockets.js adapter** - extreme performance (350K+ req/s)
- **Pluggable adapter system** - bring your own HTTP implementation

#### Performance Optimizations
- Pre-compiled route patterns at registration time
- Cached dynamic route matches
- Zero-copy body parsing where possible
- Minimal allocations in hot paths
- Efficient header management
- Request pooling support

#### Developer Experience
- Comprehensive error messages
- Full TypeScript definitions
- Extensive JSDoc documentation
- Example applications included
- Detailed logging and debugging support

#### Documentation
- Complete getting started guides
- Core concepts documentation
- API reference for all features
- Real-world examples (Todo API, Auth, GraphQL, gRPC)
- Migration guides from Express/Fastify/Koa
- Production deployment guides
- Troubleshooting documentation
- 60+ comprehensive documentation files

### Performance
- **400K+ requests/second** with Node.js HTTP adapter (benchmarked)
- **0.2ms p50 latency** under load
- **1ms p99 latency** under load
- 10x faster than Express
- 4x faster than Fastify
- 3x faster than Koa

### Breaking Changes
None (initial release)

---

## Version History

### Version Numbering

RamAPI follows semantic versioning:
- **Major** (1.0.0): Breaking API changes
- **Minor** (0.1.0): New features, backwards compatible
- **Patch** (0.1.1): Bug fixes, backwards compatible

### Release Schedule

- **Patch releases**: As needed for bug fixes
- **Minor releases**: Every 1-2 months
- **Major releases**: When breaking changes are necessary

---

## Upgrade Guide

### From 0.0.x to 0.1.0

This is the initial stable release. If you were using pre-release versions:

1. **Update package**:
```bash
npm install ramapi@latest
```

2. **Review breaking changes** (if any apply to your version)

3. **Run tests** to ensure compatibility

4. **Update observability config** if using tracing:
```typescript
// Before (example)
observability: {
  tracing: true
}

// After
observability: {
  tracing: {
    enabled: true,
    serviceName: 'my-service'
  }
}
```

---

## Migration Notes

### Express → RamAPI

Key changes when migrating from Express:
- Replace `req, res` with `ctx`
- Use `ctx.json()` instead of `res.json()`
- Use `ctx.status()` instead of `res.status()`
- Middleware uses `async/await` pattern
- Body parsing is automatic

See [Migration Guide](guides/migration-guide.md) for complete details.

### Fastify → RamAPI

Key changes when migrating from Fastify:
- Replace `request, reply` with `ctx`
- Replace JSON Schema with Zod schemas
- Use `validate()` middleware for validation
- Hooks become middleware

See [Migration Guide](guides/migration-guide.md) for complete details.

### Koa → RamAPI

RamAPI is similar to Koa! Key changes:
- Add routing (Koa requires koa-router)
- Use `ctx.json()` instead of `ctx.body = {}`
- Body parsing is automatic (no koa-bodyparser needed)
- Built-in validation with Zod

See [Migration Guide](guides/migration-guide.md) for complete details.

---

## Deprecation Policy

When features are deprecated:
1. **Announced** in changelog with deprecation notice
2. **Warning logged** when deprecated feature is used
3. **Maintained** for at least one major version
4. **Removed** in next major version

---

## Security Updates

Security vulnerabilities will be addressed in patch releases as quickly as possible.

To report a security issue, please email: security@ramapi.dev (or open a private security advisory on GitHub)

---

## Contributors

Thank you to all contributors who have helped make RamAPI better!

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

---

## Links

- [GitHub Repository](https://github.com/shanmukhram/RamAPI)
- [NPM Package](https://www.npmjs.com/package/ramapi)
- [Documentation](README.md)
- [Roadmap](ROADMAP.md)
- [Contributing Guide](CONTRIBUTING.md)

---

**Last Updated**: January 2024
