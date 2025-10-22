# Contributing to RamAPI

Thank you for your interest in contributing to RamAPI! This guide will help you get started.

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Getting Started](#getting-started)
3. [Development Setup](#development-setup)
4. [Making Changes](#making-changes)
5. [Testing](#testing)
6. [Documentation](#documentation)
7. [Submitting Pull Requests](#submitting-pull-requests)
8. [Code Standards](#code-standards)

---

## Code of Conduct

### Our Pledge

We are committed to providing a welcoming and inclusive environment for all contributors.

### Expected Behavior

- Be respectful and constructive
- Welcome newcomers
- Focus on what is best for the community
- Show empathy towards others

### Unacceptable Behavior

- Harassment or discriminatory language
- Personal attacks
- Trolling or insulting comments
- Publishing others' private information

---

## Getting Started

### Prerequisites

- Node.js 18+ installed
- Git installed
- Basic TypeScript knowledge
- Familiarity with Node.js APIs

### Find an Issue

1. Browse [open issues](https://github.com/your-org/ramapi/issues)
2. Look for issues labeled `good first issue` or `help wanted`
3. Comment on the issue to let others know you're working on it

### Ask Questions

- Open a [discussion](https://github.com/your-org/ramapi/discussions) for questions
- Join our community chat (if available)
- Check existing documentation first

---

## Development Setup

### 1. Fork and Clone

```bash
# Fork the repository on GitHub, then:
git clone https://github.com/YOUR_USERNAME/ramapi.git
cd ramapi
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Build the Project

```bash
npm run build
```

### 4. Run Tests

```bash
npm test
```

### 5. Run Examples

```bash
npm run example:basic
```

---

## Project Structure

```
ramapi/
├── src/                    # Source code
│   ├── core/              # Core framework
│   ├── middleware/        # Built-in middleware
│   ├── auth/              # Authentication
│   ├── observability/     # Tracing, profiling
│   ├── protocols/         # GraphQL, gRPC
│   └── adapters/          # HTTP adapters
├── examples/              # Example applications
├── tests/                 # Test files
├── docs/                  # Documentation
└── benchmarks/            # Performance benchmarks
```

---

## Making Changes

### 1. Create a Branch

```bash
git checkout -b feature/my-feature
# or
git checkout -b fix/my-bugfix
```

Branch naming:
- `feature/` for new features
- `fix/` for bug fixes
- `docs/` for documentation
- `perf/` for performance improvements
- `refactor/` for code refactoring

### 2. Make Your Changes

- Write clean, readable code
- Follow existing code style
- Add comments for complex logic
- Keep changes focused and atomic

### 3. Write Tests

All code changes should include tests:

```typescript
import { describe, it, expect } from 'vitest';
import { createApp } from '../src/index.js';

describe('My Feature', () => {
  it('should work correctly', () => {
    const app = createApp();
    // Test implementation
    expect(true).toBe(true);
  });
});
```

### 4. Update Documentation

- Add/update JSDoc comments
- Update relevant markdown docs
- Add examples if applicable

---

## Testing

### Run All Tests

```bash
npm test
```

### Run Specific Test File

```bash
npm test -- router.test.ts
```

### Run Tests in Watch Mode

```bash
npm test -- --watch
```

### Run Tests with Coverage

```bash
npm run test:coverage
```

### Test Guidelines

- Write tests for all new features
- Fix any failing tests
- Aim for >80% code coverage
- Test edge cases and error conditions
- Use descriptive test names

**Example:**

```typescript
describe('Router', () => {
  describe('get()', () => {
    it('should register GET route', () => {
      // Test implementation
    });

    it('should handle route parameters', () => {
      // Test implementation
    });

    it('should throw error for invalid path', () => {
      // Test implementation
    });
  });
});
```

---

## Documentation

### Inline Documentation

Use JSDoc comments:

```typescript
/**
 * Create a new RamAPI application
 *
 * @param config - Server configuration
 * @returns Server instance
 *
 * @example
 * ```typescript
 * const app = createApp({
 *   port: 3000,
 *   observability: { tracing: { enabled: true } }
 * });
 * ```
 */
export function createApp(config?: ServerConfig): Server {
  // Implementation
}
```

### Markdown Documentation

- Use clear headings
- Include code examples
- Add usage notes
- Link to related docs

### Documentation Structure

```
docs/
├── getting-started/
├── core-concepts/
├── middleware/
├── guides/
└── api-reference/
```

---

## Submitting Pull Requests

### Before Submitting

- [ ] Tests pass locally
- [ ] Code follows style guide
- [ ] Documentation updated
- [ ] Commit messages are clear
- [ ] Branch is up to date with main

### PR Process

1. **Push your branch**:
```bash
git push origin feature/my-feature
```

2. **Open PR on GitHub**

3. **Fill out PR template**:
   - Description of changes
   - Related issues
   - Testing performed
   - Breaking changes (if any)

4. **Respond to feedback**:
   - Address review comments
   - Make requested changes
   - Push updates to your branch

5. **Wait for approval**:
   - At least one maintainer approval required
   - CI tests must pass
   - No merge conflicts

### PR Title Format

```
type(scope): description

Examples:
feat(router): add support for regex patterns
fix(auth): resolve token expiration issue
docs(guides): add migration guide
perf(core): optimize route matching algorithm
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Code style (formatting)
- `refactor`: Code refactoring
- `perf`: Performance improvement
- `test`: Test updates
- `chore`: Build/tooling changes

---

## Code Standards

### TypeScript Style

```typescript
// ✅ Good
export interface UserConfig {
  name: string;
  email: string;
}

export async function getUser(id: string): Promise<User> {
  const user = await db.findById(id);
  return user;
}

// ❌ Bad
export interface user_config {
  Name: string;
  Email: string;
}

export function GetUser(id) {
  return db.findById(id);
}
```

### Naming Conventions

- **Classes**: `PascalCase` (e.g., `JWTService`, `Router`)
- **Functions**: `camelCase` (e.g., `createApp`, `authenticate`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `MAX_RETRIES`)
- **Interfaces**: `PascalCase` (e.g., `ServerConfig`, `Context`)
- **Types**: `PascalCase` (e.g., `HTTPMethod`, `Handler`)

### Code Formatting

We use Prettier and ESLint:

```bash
# Format code
npm run format

# Lint code
npm run lint

# Fix lint issues
npm run lint:fix
```

### Performance Considerations

- Avoid unnecessary allocations
- Use pre-compilation where possible
- Profile performance-critical code
- Document performance characteristics

**Example:**

```typescript
// ✅ Good - pre-compile regex
const pattern = /^\/users\/\d+$/;
export function matchRoute(path: string): boolean {
  return pattern.test(path);
}

// ❌ Bad - compiles regex every call
export function matchRoute(path: string): boolean {
  return /^\/users\/\d+$/.test(path);
}
```

---

## Benchmarking

### Run Benchmarks

```bash
npm run benchmark
```

### Adding Benchmarks

```typescript
import { benchmark } from './utils/benchmark.js';

benchmark('route matching', () => {
  app.findRoute('GET', '/users/123');
});
```

### Performance Requirements

- No performance regressions
- Document performance impact of changes
- Include benchmarks for performance-related PRs

---

## Release Process

(For maintainers)

### Version Bumping

```bash
# Patch (0.1.0 -> 0.1.1)
npm version patch

# Minor (0.1.0 -> 0.2.0)
npm version minor

# Major (0.1.0 -> 1.0.0)
npm version major
```

### Publishing

```bash
npm run build
npm publish
```

---

## Questions?

- **Documentation**: Check [complete documentation](/)
- **Discussions**: Ask questions on GitHub Discussions
- **Issues**: Report bugs via GitHub Issues
- **Email**: Contact maintainers (if provided)

---

## Recognition

Contributors will be:
- Listed in CONTRIBUTORS.md
- Mentioned in release notes
- Recognized in documentation

---

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

Thank you for contributing to RamAPI! 
