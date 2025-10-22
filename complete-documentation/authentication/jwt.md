# JWT Authentication

JSON Web Tokens (JWT) provide stateless authentication for your API. This guide covers the JWTService class, token generation, verification, and authentication middleware.

## Table of Contents

1. [Overview](#overview)
2. [JWTService Class](#jwtservice-class)
3. [Token Generation](#token-generation)
4. [Token Verification](#token-verification)
5. [Authentication Middleware](#authentication-middleware)
6. [Configuration](#configuration)
7. [Security Best Practices](#security-best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Overview

RamAPI uses the `jsonwebtoken` library for JWT operations, providing a simple and secure API for authentication.

### Installation

JWT support is built-in, but you need to install the peer dependency:

```bash
npm install jsonwebtoken
npm install -D @types/jsonwebtoken
```

### Quick Start

```typescript
import { createApp, JWTService, authenticate } from 'ramapi';

// Create JWT service
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});

const app = createApp();

// Protected route
app.get('/api/profile',
  authenticate(jwtService),
  async (ctx) => {
    ctx.json({ user: ctx.user });
  }
);

app.listen(3000);
```

---

## JWTService Class

### Constructor

```typescript
class JWTService {
  constructor(config: JWTConfig);
}

interface JWTConfig {
  secret: string;              // Required: Secret key for signing
  expiresIn?: number;          // Optional: Expiration time in seconds
  algorithm?: jwt.Algorithm;   // Optional: Signing algorithm (default: 'HS256')
  issuer?: string;             // Optional: Token issuer
  audience?: string;           // Optional: Token audience
}
```

### Create Instance

```typescript
import { JWTService } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours in seconds
});
```

### Methods

#### sign(payload: JWTPayload): string

Signs a JWT token with the provided payload.

```typescript
interface JWTPayload {
  sub: string;  // Subject (user ID) - required
  [key: string]: unknown; // Additional claims
}
```

#### verify(token: string): JWTPayload

Verifies and decodes a JWT token. Throws HTTPError on invalid/expired tokens.

#### decode(token: string): JWTPayload | null

Decodes a token without verification. Use with caution.

---

## Token Generation

### Basic Token

```typescript
const token = jwtService.sign({
  sub: 'user-123', // User ID (required)
});

console.log(token);
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Token with Custom Claims

```typescript
const token = jwtService.sign({
  sub: 'user-123',
  email: 'user@example.com',
  role: 'admin',
  permissions: ['read', 'write', 'delete'],
});
```

### Login Endpoint Example

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

const loginSchema = {
  body: z.object({
    email: z.string().email(),
    password: z.string().min(8),
  }),
};

app.post('/auth/login',
  validate(loginSchema),
  async (ctx) => {
    const { email, password } = ctx.body;

    // Verify credentials (pseudo-code)
    const user = await db.users.findByEmail(email);
    if (!user || !(await passwordService.verify(password, user.passwordHash))) {
      throw new HTTPError(401, 'Invalid credentials');
    }

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
      role: user.role,
    });

    ctx.json({ token });
  }
);
```

---

## Token Verification

### Manual Verification

```typescript
app.get('/api/verify', async (ctx) => {
  const authHeader = ctx.headers.authorization as string;

  if (!authHeader) {
    throw new HTTPError(401, 'No token provided');
  }

  const token = authHeader.split(' ')[1]; // Extract from "Bearer <token>"

  try {
    const payload = jwtService.verify(token);
    ctx.json({
      valid: true,
      payload,
    });
  } catch (error) {
    ctx.json({
      valid: false,
      error: (error as Error).message,
    }, 401);
  }
});
```

### Decode Without Verification

Use this to inspect token contents without validating the signature:

```typescript
const token = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';
const payload = jwtService.decode(token);

console.log(payload);
// { sub: 'user-123', email: 'user@example.com', ... }
```

**Warning:** Never use decoded tokens for authentication without verification!

---

## Authentication Middleware

### authenticate()

Requires a valid JWT token. Throws 401 if missing or invalid.

```typescript
import { authenticate } from 'ramapi';

app.get('/api/profile',
  authenticate(jwtService),
  async (ctx) => {
    // ctx.user is populated with token payload
    console.log(ctx.user);
    // { sub: 'user-123', email: 'user@example.com', ... }

    // ctx.state.userId contains the user ID
    console.log(ctx.state.userId); // 'user-123'

    ctx.json({ user: ctx.user });
  }
);
```

### optionalAuthenticate()

Authenticates if token is present, but doesn't fail if missing.

```typescript
import { optionalAuthenticate } from 'ramapi';

app.get('/api/posts',
  optionalAuthenticate(jwtService),
  async (ctx) => {
    if (ctx.user) {
      // User is authenticated - show private posts
      const posts = await db.posts.findByUserId(ctx.user.sub);
      ctx.json({ posts, authenticated: true });
    } else {
      // No authentication - show public posts only
      const posts = await db.posts.findPublic();
      ctx.json({ posts, authenticated: false });
    }
  }
);
```

### Authorization Header Format

The middleware expects tokens in the Authorization header:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Error Handling

The middleware throws HTTPError with appropriate status codes:

- **401**: Authorization header missing
- **401**: Invalid authorization header format
- **401**: Token expired
- **401**: Invalid token signature
- **401**: Other authentication failures

---

## Configuration

### Basic Configuration

```typescript
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});
```

### Full Configuration

```typescript
const jwtService = new JWTService({
  // Required: Secret key for signing tokens
  secret: process.env.JWT_SECRET!,

  // Optional: Token expiration in seconds
  expiresIn: 86400, // 24 hours

  // Optional: Signing algorithm
  algorithm: 'HS256', // Default, also: HS384, HS512, RS256, etc.

  // Optional: Token issuer (your API name)
  issuer: 'my-api',

  // Optional: Token audience (your app name)
  audience: 'my-app',
});
```

### Algorithm Options

| Algorithm | Type | Security | Speed |
|-----------|------|----------|-------|
| HS256 | HMAC | Good | Fast |
| HS384 | HMAC | Better | Fast |
| HS512 | HMAC | Best | Fast |
| RS256 | RSA | Best | Slower |
| RS384 | RSA | Best | Slower |
| RS512 | RSA | Best | Slower |

**Recommendation:** Use HS256 for most applications, RS256 for microservices.

### Environment Variables

```bash
# .env
JWT_SECRET=your-super-secret-key-at-least-32-characters-long
JWT_EXPIRES_IN=86400
JWT_ISSUER=my-api
JWT_AUDIENCE=my-app
```

```typescript
import { config as loadEnv } from 'dotenv';
loadEnv();

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: parseInt(process.env.JWT_EXPIRES_IN || '86400'),
  issuer: process.env.JWT_ISSUER,
  audience: process.env.JWT_AUDIENCE,
});
```

### Token Expiration

Common expiration times:

```typescript
// 15 minutes
expiresIn: 900

// 1 hour
expiresIn: 3600

// 24 hours
expiresIn: 86400

// 7 days
expiresIn: 604800

// 30 days
expiresIn: 2592000
```

---

## Security Best Practices

### 1. Strong Secret Keys

```typescript
// BAD - Too short
secret: 'secret123'

// GOOD - Long, random, secure
secret: crypto.randomBytes(64).toString('hex')
```

Generate a secure secret:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### 2. Short Expiration Times

```typescript
// BAD - Token valid for 1 year
expiresIn: 31536000

// GOOD - Token valid for 24 hours
expiresIn: 86400

// BETTER - Short-lived with refresh tokens
expiresIn: 900 // 15 minutes
```

### 3. Use HTTPS

JWT tokens should only be transmitted over HTTPS to prevent interception.

### 4. Store Secrets Securely

```typescript
// BAD - Hardcoded secret
const jwtService = new JWTService({
  secret: 'my-secret-key',
});

// GOOD - From environment variable
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
});
```

### 5. Validate Token Claims

```typescript
app.get('/admin',
  authenticate(jwtService),
  async (ctx) => {
    // Verify role claim
    if (ctx.user.role !== 'admin') {
      throw new HTTPError(403, 'Forbidden: Admin access required');
    }

    ctx.json({ data: 'Admin data' });
  }
);
```

### 6. Don't Store Sensitive Data in Tokens

```typescript
// BAD - Storing sensitive data
const token = jwtService.sign({
  sub: user.id,
  password: user.password, // NEVER!
  ssn: user.ssn, // NEVER!
});

// GOOD - Only non-sensitive identifiers
const token = jwtService.sign({
  sub: user.id,
  email: user.email,
  role: user.role,
});
```

### 7. Implement Token Revocation

JWTs are stateless, so implement a blacklist for revoked tokens:

```typescript
const revokedTokens = new Set<string>();

function revokeToken(token: string) {
  revokedTokens.add(token);
}

function customAuthenticate(jwtService: JWTService): Middleware {
  return async (ctx, next) => {
    const authHeader = ctx.headers.authorization as string;
    if (!authHeader) {
      throw new HTTPError(401, 'Authorization header missing');
    }

    const token = authHeader.split(' ')[1];

    // Check if token is revoked
    if (revokedTokens.has(token)) {
      throw new HTTPError(401, 'Token has been revoked');
    }

    const payload = jwtService.verify(token);
    ctx.user = payload;
    await next();
  };
}
```

### 8. Use Refresh Tokens

Implement refresh tokens for long-lived sessions:

```typescript
// Short-lived access token
const accessToken = jwtService.sign({
  sub: user.id,
  type: 'access',
}, { expiresIn: 900 }); // 15 minutes

// Long-lived refresh token (stored securely)
const refreshToken = jwtService.sign({
  sub: user.id,
  type: 'refresh',
}, { expiresIn: 604800 }); // 7 days
```

---

## Troubleshooting

### Problem: "Authorization header missing"

**Cause:** No Authorization header in request.

**Solution:** Include Authorization header with Bearer token:

```javascript
fetch('/api/profile', {
  headers: {
    'Authorization': `Bearer ${token}`,
  },
});
```

### Problem: "Invalid authorization header format"

**Cause:** Authorization header not in "Bearer <token>" format.

**Solution:** Ensure correct format:

```typescript
// WRONG
Authorization: eyJhbGci...

// CORRECT
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Problem: "Token expired"

**Cause:** Token's expiration time has passed.

**Solution:** Generate a new token or implement refresh token flow.

### Problem: "Invalid token"

**Cause:** Token signature doesn't match or token is malformed.

**Solution:** Ensure the same secret is used for signing and verification.

### Problem: Token works locally but not in production

**Cause:** Different JWT_SECRET in environments.

**Solution:** Ensure JWT_SECRET is set correctly in production environment.

---

## Complete Example

```typescript
import { createApp, JWTService, authenticate, optionalAuthenticate } from 'ramapi';
import { passwordService } from 'ramapi';

// Setup JWT
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
  issuer: 'my-api',
  audience: 'my-app',
});

const app = createApp();

// Login endpoint
app.post('/auth/login', async (ctx) => {
  const { email, password } = ctx.body;

  // Verify credentials
  const user = await db.users.findByEmail(email);
  if (!user || !(await passwordService.verify(password, user.passwordHash))) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Generate token
  const token = jwtService.sign({
    sub: user.id,
    email: user.email,
    role: user.role,
  });

  ctx.json({ token, user: { id: user.id, email: user.email } });
});

// Public endpoint (optional auth)
app.get('/api/posts',
  optionalAuthenticate(jwtService),
  async (ctx) => {
    const posts = ctx.user
      ? await db.posts.findAll()
      : await db.posts.findPublic();

    ctx.json({ posts });
  }
);

// Protected endpoint
app.get('/api/profile',
  authenticate(jwtService),
  async (ctx) => {
    const user = await db.users.findById(ctx.user.sub);
    ctx.json({ user });
  }
);

// Admin-only endpoint
app.get('/api/admin',
  authenticate(jwtService),
  async (ctx) => {
    if (ctx.user.role !== 'admin') {
      throw new HTTPError(403, 'Admin access required');
    }

    ctx.json({ data: 'Admin data' });
  }
);

app.listen(3000);
```

---

## Next Steps

- [Password Hashing](password-hashing.md)
- [Authentication Patterns](auth-patterns.md)
- [Security Best Practices](../guides/security.md)
- [Middleware Guide](../middleware/custom-middleware.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
