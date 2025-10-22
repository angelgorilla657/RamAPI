# Authentication API Reference

Complete API reference for RamAPI's JWT authentication and password hashing utilities.

## Table of Contents

1. [JWTService](#jwtservice)
2. [authenticate()](#authenticate)
3. [optionalAuthenticate()](#optionalauthenticate)
4. [PasswordService](#passwordservice)
5. [Complete Examples](#complete-examples)

---

## JWTService

Service class for signing and verifying JWT tokens.

### Constructor

```typescript
class JWTService {
  constructor(config: JWTConfig)
}
```

### JWTConfig

```typescript
interface JWTConfig {
  secret: string;
  expiresIn?: number;
  algorithm?: jwt.Algorithm;
  issuer?: string;
  audience?: string;
}
```

#### Options

| Option | Type | Required | Default | Description |
|--------|------|----------|---------|-------------|
| `secret` | `string` | Yes | - | Secret key for signing tokens |
| `expiresIn` | `number` | No | `undefined` | Expiration time in seconds |
| `algorithm` | `jwt.Algorithm` | No | `'HS256'` | JWT signing algorithm |
| `issuer` | `string` | No | `undefined` | Token issuer |
| `audience` | `string` | No | `undefined` | Token audience |

### Methods

#### sign()

Sign a JWT token.

```typescript
sign(payload: JWTPayload): string
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `payload` | `JWTPayload` | Token payload |

**Returns:** `string` - Signed JWT token

**Example:**

```typescript
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});

const token = jwtService.sign({
  sub: '123', // User ID
  email: 'user@example.com',
  role: 'admin',
});

console.log(token);
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

#### verify()

Verify and decode a JWT token.

```typescript
verify(token: string): JWTPayload
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `token` | `string` | JWT token to verify |

**Returns:** `JWTPayload` - Decoded token payload

**Throws:**
- `HTTPError(401)` if token is expired
- `HTTPError(401)` if token is invalid

**Example:**

```typescript
try {
  const payload = jwtService.verify(token);
  console.log(payload);
  // { sub: '123', email: 'user@example.com', role: 'admin', iat: ..., exp: ... }
} catch (error) {
  console.error('Token verification failed:', error.message);
}
```

---

#### decode()

Decode token without verification (use with caution).

```typescript
decode(token: string): JWTPayload | null
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `token` | `string` | JWT token to decode |

**Returns:** `JWTPayload | null` - Decoded payload or null

**Example:**

```typescript
const payload = jwtService.decode(token);
console.log(payload?.sub); // User ID (not verified!)
```

---

### JWTPayload

JWT payload structure.

```typescript
interface JWTPayload {
  sub: string; // Subject (user ID)
  [key: string]: unknown;
}
```

**Common fields:**

- `sub`: Subject (user ID) - **required**
- `iat`: Issued at (timestamp)
- `exp`: Expiration (timestamp)
- `iss`: Issuer
- `aud`: Audience
- Custom fields: Add any additional data

**Example:**

```typescript
const payload: JWTPayload = {
  sub: '123',
  email: 'user@example.com',
  role: 'admin',
  permissions: ['read', 'write'],
};
```

---

## authenticate()

Middleware factory for JWT authentication.

### Signature

```typescript
function authenticate(jwtService: JWTService): Middleware
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `jwtService` | `JWTService` | JWT service instance |

### Returns

`Middleware` - Authentication middleware

### Behavior

1. Extracts token from `Authorization` header
2. Verifies token using JWT service
3. Sets `ctx.user` with decoded payload
4. Throws `HTTPError(401)` if token is missing or invalid

### Example

```typescript
import { JWTService, authenticate } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400,
});

const auth = authenticate(jwtService);

// Protected route
app.get('/profile', auth, async (ctx) => {
  console.log(ctx.user);
  // { sub: '123', email: 'user@example.com', ... }

  ctx.json({ profile: ctx.user });
});
```

---

## optionalAuthenticate()

Middleware factory for optional JWT authentication.

### Signature

```typescript
function optionalAuthenticate(jwtService: JWTService): Middleware
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `jwtService` | `JWTService` | JWT service instance |

### Returns

`Middleware` - Optional authentication middleware

### Behavior

1. Checks for `Authorization` header
2. If present, verifies token and sets `ctx.user`
3. If missing or invalid, continues without setting `ctx.user`
4. Never throws authentication errors

### Example

```typescript
import { JWTService, optionalAuthenticate } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
});

const optionalAuth = optionalAuthenticate(jwtService);

// Public route with optional auth
app.get('/posts', optionalAuth, async (ctx) => {
  if (ctx.user) {
    // Authenticated user - show private posts
    const posts = await getPostsForUser(ctx.user.sub);
    ctx.json({ posts });
  } else {
    // Anonymous user - show public posts only
    const posts = await getPublicPosts();
    ctx.json({ posts });
  }
});
```

---

## PasswordService

Service for hashing and verifying passwords using bcrypt.

### hash()

Hash a password.

```typescript
async hash(password: string): Promise<string>
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `password` | `string` | Plain text password |

**Returns:** `Promise<string>` - Hashed password

**Example:**

```typescript
import { passwordService } from 'ramapi';

const hashedPassword = await passwordService.hash('myPassword123');
console.log(hashedPassword);
// $2b$10$...
```

---

### verify()

Verify a password against a hash.

```typescript
async verify(password: string, hash: string): Promise<boolean>
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `password` | `string` | Plain text password |
| `hash` | `string` | Hashed password |

**Returns:** `Promise<boolean>` - True if password matches

**Example:**

```typescript
const isValid = await passwordService.verify('myPassword123', hashedPassword);

if (isValid) {
  console.log('Password correct');
} else {
  console.log('Password incorrect');
}
```

---

### passwordService

Pre-configured global instance of PasswordService.

```typescript
import { passwordService } from 'ramapi';

// Hash password
const hash = await passwordService.hash(password);

// Verify password
const valid = await passwordService.verify(password, hash);
```

---

## Complete Examples

### User Registration

```typescript
import { createApp, validate, passwordService, JWTService } from 'ramapi';
import { z } from 'zod';

const app = createApp();

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});

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
    const existing = await db.getUserByEmail(email);
    if (existing) {
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
      role: user.role,
    });

    ctx.json({
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
      token,
    }, 201);
  }
);
```

---

### User Login

```typescript
import { authenticate } from 'ramapi';

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
      role: user.role,
    });

    ctx.json({
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
      token,
    });
  }
);
```

---

### Protected Routes

```typescript
const auth = authenticate(jwtService);

// Get current user profile
app.get('/profile', auth, async (ctx) => {
  const userId = ctx.user.sub;
  const user = await db.getUserById(userId);

  ctx.json({ user });
});

// Update profile
app.put('/profile',
  auth,
  validate({
    body: z.object({
      name: z.string().min(2).optional(),
      bio: z.string().max(500).optional(),
    }),
  }),
  async (ctx) => {
    const userId = ctx.user.sub;
    const updates = ctx.body;

    const user = await db.updateUser(userId, updates);

    ctx.json({ user });
  }
);
```

---

### Role-Based Access Control

```typescript
// Middleware factory for role checking
function requireRole(role: string): Middleware {
  return async (ctx, next) => {
    if (!ctx.user) {
      ctx.json({ error: 'Unauthorized' }, 401);
      return;
    }

    if (ctx.user.role !== role) {
      ctx.json({ error: 'Forbidden' }, 403);
      return;
    }

    await next();
  };
}

const auth = authenticate(jwtService);
const adminOnly = requireRole('admin');

// Admin-only route
app.delete('/users/:id',
  auth,
  adminOnly,
  async (ctx) => {
    await db.deleteUser(ctx.params.id);
    ctx.status(204);
  }
);
```

---

### Token Refresh

```typescript
app.post('/auth/refresh',
  auth,
  async (ctx) => {
    // Generate new token
    const newToken = jwtService.sign({
      sub: ctx.user.sub,
      email: ctx.user.email,
      role: ctx.user.role,
    });

    ctx.json({ token: newToken });
  }
);
```

---

### Custom JWT Claims

```typescript
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400,
  issuer: 'my-app',
  audience: 'api.example.com',
});

// Sign with custom claims
const token = jwtService.sign({
  sub: user.id,
  email: user.email,
  role: user.role,
  permissions: ['read', 'write'],
  sessionId: generateSessionId(),
});

// Verify includes issuer/audience checks
const payload = jwtService.verify(token);
```

---

## Security Best Practices

### 1. Use Strong Secrets

```typescript
// Bad
const jwtService = new JWTService({ secret: 'secret' });

// Good
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!, // Long, random string
});
```

### 2. Set Expiration

```typescript
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});
```

### 3. Use HTTPS

Always use HTTPS in production to protect tokens in transit.

### 4. Store Tokens Securely

- Don't store in localStorage (XSS vulnerable)
- Use httpOnly cookies or secure storage
- Clear tokens on logout

### 5. Validate All Inputs

```typescript
app.post('/auth/login',
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(1), // Don't reveal length requirements to attackers
    }),
  }),
  async (ctx) => {
    // ...
  }
);
```

---

## See Also

- [Authentication Guide](../authentication/jwt.md) - JWT authentication guide
- [Password Hashing Guide](../authentication/password-hashing.md) - Password security guide
- [Middleware API](middleware-api.md) - Built-in middleware

---

**Need help?** Check the [Authentication Guide](../getting-started/authentication.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
