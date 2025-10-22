# Auth API Reference

Complete API reference for RamAPI's authentication utilities.

## Table of Contents

1. [JWTService](#jwtservice)
2. [JWTConfig](#jwtconfig)
3. [JWTPayload](#jwtpayload)
4. [authenticate()](#authenticate)
5. [optionalAuthenticate()](#optionalauthenticate)
6. [PasswordService](#passwordservice)
7. [Examples](#examples)

---

## JWTService

Service class for JWT token operations.

### Constructor

```typescript
new JWTService(config: JWTConfig)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `config` | `JWTConfig` | Yes | JWT configuration |

**Example:**

```typescript
import { JWTService } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours in seconds
  algorithm: 'HS256',
});
```

---

### sign()

Sign a JWT token with payload.

```typescript
sign(payload: JWTPayload): string
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payload` | `JWTPayload` | Yes | Token payload (must include `sub`) |

**Returns:** `string` - Signed JWT token

**Example:**

```typescript
const token = jwtService.sign({
  sub: 'user-123',
  email: 'user@example.com',
  role: 'admin',
});

console.log(token);
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyIsImVtYWlsIjoidXNl...
```

---

### verify()

Verify and decode a JWT token.

```typescript
verify(token: string): JWTPayload
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `token` | `string` | Yes | JWT token to verify |

**Returns:** `JWTPayload` - Decoded and verified payload

**Throws:**
- `HTTPError(401, 'Token expired')` - If token has expired
- `HTTPError(401, 'Invalid token')` - If token is malformed or signature is invalid

**Example:**

```typescript
try {
  const payload = jwtService.verify(token);
  console.log('User ID:', payload.sub);
  console.log('Email:', payload.email);
} catch (error) {
  if (error.statusCode === 401) {
    console.error('Authentication failed:', error.message);
  }
}
```

---

### decode()

Decode token without verification (unsafe - use with caution).

```typescript
decode(token: string): JWTPayload | null
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `token` | `string` | Yes | JWT token to decode |

**Returns:** `JWTPayload | null` - Decoded payload (not verified) or null if invalid

**Warning:** This method does NOT verify the token signature. Only use for debugging or when verification is not required.

**Example:**

```typescript
const payload = jwtService.decode(token);

if (payload) {
  console.log('Token contains:', payload);
  // Don't trust this data - it's not verified!
}
```

---

## JWTConfig

Configuration interface for JWT service.

```typescript
interface JWTConfig {
  secret: string;
  expiresIn?: number;
  algorithm?: jwt.Algorithm;
  issuer?: string;
  audience?: string;
}
```

### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `secret` | `string` | **Yes** | - | Secret key for signing/verifying tokens |
| `expiresIn` | `number` | No | `undefined` | Token expiration time in seconds |
| `algorithm` | `jwt.Algorithm` | No | `'HS256'` | Signing algorithm |
| `issuer` | `string` | No | `undefined` | Token issuer (iss claim) |
| `audience` | `string` | No | `undefined` | Token audience (aud claim) |

### Algorithms

Supported algorithms (from jsonwebtoken):

**Symmetric:**
- `HS256` (HMAC SHA-256) - Default
- `HS384` (HMAC SHA-384)
- `HS512` (HMAC SHA-512)

**Asymmetric:**
- `RS256` (RSA SHA-256)
- `RS384` (RSA SHA-384)
- `RS512` (RSA SHA-512)
- `ES256` (ECDSA SHA-256)
- `ES384` (ECDSA SHA-384)
- `ES512` (ECDSA SHA-512)

### Examples

**Basic configuration:**

```typescript
const config: JWTConfig = {
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
};
```

**With issuer and audience:**

```typescript
const config: JWTConfig = {
  secret: process.env.JWT_SECRET!,
  expiresIn: 3600, // 1 hour
  issuer: 'my-app',
  audience: 'api.example.com',
};
```

**With RSA (asymmetric):**

```typescript
import fs from 'fs';

const config: JWTConfig = {
  secret: fs.readFileSync('private.key', 'utf8'),
  algorithm: 'RS256',
  expiresIn: 86400,
};
```

---

## JWTPayload

JWT payload structure.

```typescript
interface JWTPayload {
  sub: string; // Subject (user ID) - REQUIRED
  [key: string]: unknown; // Additional custom claims
}
```

### Standard Claims

| Claim | Type | Required | Description |
|-------|------|----------|-------------|
| `sub` | `string` | **Yes** | Subject (typically user ID) |
| `iat` | `number` | Auto | Issued at (timestamp) |
| `exp` | `number` | Auto | Expiration (timestamp) |
| `iss` | `string` | Optional | Issuer |
| `aud` | `string` | Optional | Audience |

### Custom Claims

Add any custom data to the payload:

```typescript
const payload: JWTPayload = {
  sub: 'user-123',
  email: 'user@example.com',
  role: 'admin',
  permissions: ['read', 'write', 'delete'],
  organizationId: 'org-456',
};
```

---

## authenticate()

Middleware factory for required JWT authentication.

```typescript
function authenticate(jwtService: JWTService): Middleware
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `jwtService` | `JWTService` | Yes | JWT service instance |

**Returns:** `Middleware` - Authentication middleware

**Behavior:**

1. Extracts token from `Authorization: Bearer <token>` header
2. Verifies token using JWT service
3. Sets `ctx.user` to decoded payload
4. Sets `ctx.state.userId` to `payload.sub`
5. Throws `HTTPError(401)` if:
   - Authorization header is missing
   - Header format is invalid (not "Bearer <token>")
   - Token verification fails

**Example:**

```typescript
import { JWTService, authenticate } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
});

const auth = authenticate(jwtService);

// Protected route
app.get('/profile', auth, async (ctx) => {
  console.log('Authenticated user:', ctx.user);
  // { sub: 'user-123', email: 'user@example.com', ... }

  const userId = ctx.state.userId; // 'user-123'
  const user = await db.getUser(userId);

  ctx.json({ profile: user });
});

// Multiple protected routes
app.get('/orders', auth, getOrders);
app.post('/orders', auth, createOrder);
app.get('/settings', auth, getSettings);
```

**Error Responses:**

```json
// Missing header
{
  "error": true,
  "message": "Authorization header missing"
}

// Invalid format
{
  "error": true,
  "message": "Invalid authorization header format. Expected: Bearer <token>"
}

// Invalid/expired token
{
  "error": true,
  "message": "Token expired"
}
```

---

## optionalAuthenticate()

Middleware factory for optional JWT authentication.

```typescript
function optionalAuthenticate(jwtService: JWTService): Middleware
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `jwtService` | `JWTService` | Yes | JWT service instance |

**Returns:** `Middleware` - Optional authentication middleware

**Behavior:**

1. Checks for `Authorization: Bearer <token>` header
2. If present and valid:
   - Sets `ctx.user` to decoded payload
   - Sets `ctx.state.userId` to `payload.sub`
3. If missing or invalid:
   - Continues without error
   - `ctx.user` remains `undefined`

**Example:**

```typescript
import { JWTService, optionalAuthenticate } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
});

const optionalAuth = optionalAuthenticate(jwtService);

// Public endpoint with optional auth
app.get('/posts', optionalAuth, async (ctx) => {
  if (ctx.user) {
    // Authenticated - show user-specific content
    const posts = await db.getPostsForUser(ctx.user.sub);
    ctx.json({ posts, authenticated: true });
  } else {
    // Anonymous - show public content only
    const posts = await db.getPublicPosts();
    ctx.json({ posts, authenticated: false });
  }
});
```

---

## PasswordService

Service class for password hashing and verification using bcrypt.

### Constructor

```typescript
new PasswordService(saltRounds?: number)
```

**Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `saltRounds` | `number` | No | `10` | Number of bcrypt salt rounds |

**Salt Rounds Guide:**

| Rounds | Time | Security |
|--------|------|----------|
| 10 | ~100ms | Good (default) |
| 12 | ~400ms | Better |
| 14 | ~1.6s | Best |

**Example:**

```typescript
import { PasswordService } from 'ramapi';

const passwordService = new PasswordService(12); // Higher security
```

---

### hash()

Hash a password using bcrypt.

```typescript
async hash(password: string): Promise<string>
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `password` | `string` | Yes | Plain text password |

**Returns:** `Promise<string>` - Hashed password

**Example:**

```typescript
const hashedPassword = await passwordService.hash('myPassword123');
console.log(hashedPassword);
// $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
```

---

### verify()

Verify a password against a hash.

```typescript
async verify(password: string, hash: string): Promise<boolean>
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `password` | `string` | Yes | Plain text password |
| `hash` | `string` | Yes | Hashed password to compare against |

**Returns:** `Promise<boolean>` - `true` if password matches, `false` otherwise

**Example:**

```typescript
const isValid = await passwordService.verify('myPassword123', hashedPassword);

if (isValid) {
  console.log('Password correct!');
} else {
  console.log('Password incorrect!');
}
```

---

### needsRehash()

Check if a hash needs to be rehashed (e.g., salt rounds changed).

```typescript
needsRehash(hash: string): boolean
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `hash` | `string` | Yes | Hashed password |

**Returns:** `boolean` - `true` if hash should be regenerated

**Example:**

```typescript
const hash = '$2a$10$...'; // Hash with 10 rounds

const service = new PasswordService(12); // Now using 12 rounds

if (service.needsRehash(hash)) {
  // Rehash with new salt rounds
  const newHash = await service.hash(password);
  await db.updateUserPassword(userId, newHash);
}
```

---

### passwordService (default instance)

Pre-configured global instance with default settings (10 salt rounds).

```typescript
import { passwordService } from 'ramapi';

// Hash
const hash = await passwordService.hash(password);

// Verify
const valid = await passwordService.verify(password, hash);
```

---

## Examples

### Complete Auth Flow

```typescript
import { createApp, JWTService, passwordService, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

// JWT Service
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});

// Register
app.post('/auth/register',
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(8),
      name: z.string().min(2),
    }),
  }),
  async (ctx) => {
    const { email, password, name } = ctx.body;

    // Check if user exists
    const exists = await db.getUserByEmail(email);
    if (exists) {
      ctx.json({ error: 'Email already registered' }, 409);
      return;
    }

    // Hash password
    const hashedPassword = await passwordService.hash(password);

    // Create user
    const user = await db.createUser({
      email,
      password: hashedPassword,
      name,
    });

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
    });

    ctx.json({ user: { id: user.id, email, name }, token }, 201);
  }
);

// Login
app.post('/auth/login',
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string(),
    }),
  }),
  async (ctx) => {
    const { email, password } = ctx.body;

    // Get user
    const user = await db.getUserByEmail(email);
    if (!user) {
      ctx.json({ error: 'Invalid credentials' }, 401);
      return;
    }

    // Verify password
    const valid = await passwordService.verify(password, user.password);
    if (!valid) {
      ctx.json({ error: 'Invalid credentials' }, 401);
      return;
    }

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
    });

    ctx.json({ user: { id: user.id, email: user.email, name: user.name }, token });
  }
);

// Protected routes
const auth = authenticate(jwtService);

app.get('/profile', auth, async (ctx) => {
  const user = await db.getUser(ctx.user.sub);
  ctx.json({ user });
});
```

---

## See Also

- [Authentication Guide](../authentication/jwt.md) - Complete authentication guide
- [Middleware API](middleware-api.md) - Middleware reference
- [Context API](context.md) - Context reference

---

**Need help?** Check the [Authentication Documentation](../authentication/) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
