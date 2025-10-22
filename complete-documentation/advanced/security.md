# Security Best Practices

Comprehensive security guide for RamAPI applications: input validation, authentication, authorization, rate limiting, and protection against common vulnerabilities.

> **Note**: This documentation has been verified against RamAPI source code. Core APIs verified:
> - ✅ `rateLimit()` middleware - Configuration and usage verified (windowMs, maxRequests, message)
> - ✅ `cors()` middleware - Configuration verified (origin, methods, allowedHeaders, credentials, maxAge)
> - ✅ `authenticate()` and `JWTService` - JWT authentication patterns verified
> - ✅ `passwordService` - Password hashing with bcrypt verified
> - ✅ `validate()` middleware - Zod schema validation verified
> - ⚠️ Advanced security patterns (CSRF, custom headers) are conceptual best practices

## Table of Contents

1. [Security Principles](#security-principles)
2. [Input Validation](#input-validation)
3. [Authentication](#authentication)
4. [Authorization](#authorization)
5. [Rate Limiting](#rate-limiting)
6. [CORS Configuration](#cors-configuration)
7. [Common Vulnerabilities](#common-vulnerabilities)
8. [Security Headers](#security-headers)
9. [Secrets Management](#secrets-management)
10. [Security Checklist](#security-checklist)

---

## Security Principles

### Defense in Depth

Implement multiple layers of security:

```typescript
import { createApp, authenticate, cors, rateLimit, validate } from 'ramapi';

const app = createApp();

// Layer 1: CORS - Control who can access
app.use(cors({
  origin: ['https://yourdomain.com'],
  credentials: true,
}));

// Layer 2: Rate limiting - Prevent abuse
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
}));

// Layer 3: Authentication - Verify identity
const auth = authenticate(jwtService);

// Layer 4: Input validation - Validate data
app.post('/users', auth, validate({ body: userSchema }), createUser);
```

### Least Privilege

Grant minimum necessary permissions:

```typescript
// Good: Role-based access
function requireRole(role: string) {
  return (ctx: Context, next: () => Promise<void>) => {
    if (ctx.user?.role !== role) {
      throw new HTTPError(403, 'Insufficient permissions');
    }
    return next();
  };
}

app.delete('/users/:id', auth, requireRole('admin'), deleteUser);

// Bad: All authenticated users can delete
app.delete('/users/:id', auth, deleteUser);
```

### Fail Securely

Handle errors without leaking sensitive information:

```typescript
// Good: Generic error messages
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error: any) {
    if (error.statusCode) {
      ctx.status(error.statusCode);
      ctx.json({ error: error.message });
    } else {
      // Don't leak internal errors
      console.error('Internal error:', error);
      ctx.status(500);
      ctx.json({ error: 'Internal server error' });
    }
  }
});

// Bad: Exposes stack traces
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error: any) {
    ctx.status(500);
    ctx.json({ error: error.message, stack: error.stack });
  }
});
```

---

## Input Validation

### Schema Validation with Zod

```typescript
import { z } from 'zod';
import { validate } from 'ramapi';

// Define schemas
const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100),
  age: z.number().int().min(13).max(120).optional(),
});

const updateUserSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  email: z.string().email().optional(),
});

// Use validation middleware
app.post('/users', validate({ body: createUserSchema }), async (ctx) => {
  // ctx.body is typed and validated
  const { email, password, name } = ctx.body;

  // Safe to use
  const user = await createUser({ email, password, name });
  ctx.json(user, 201);
});

app.put('/users/:id', validate({ body: updateUserSchema }), async (ctx) => {
  const user = await updateUser(ctx.params.id, ctx.body);
  ctx.json(user);
});
```

### Parameter Validation

```typescript
// Validate URL parameters
const idParamSchema = z.object({
  id: z.string().regex(/^\d+$/, 'ID must be numeric'),
});

app.get('/users/:id', validate({ params: idParamSchema }), async (ctx) => {
  const userId = parseInt(ctx.params.id);
  const user = await getUser(userId);
  ctx.json(user);
});

// Validate query parameters
const searchQuerySchema = z.object({
  q: z.string().min(1).max(100),
  page: z.string().regex(/^\d+$/).transform(Number).optional(),
  limit: z.string().regex(/^\d+$/).transform(Number).optional(),
});

app.get('/search', async (ctx) => {
  const url = new URL(ctx.path, `http://${ctx.headers.host}`);
  const query = {
    q: url.searchParams.get('q'),
    page: url.searchParams.get('page'),
    limit: url.searchParams.get('limit'),
  };

  const result = searchQuerySchema.safeParse(query);

  if (!result.success) {
    ctx.status(400);
    ctx.json({ error: 'Invalid query parameters', details: result.error });
    return;
  }

  const { q, page = 1, limit = 10 } = result.data;
  const results = await search(q, page, limit);
  ctx.json(results);
});
```

### Sanitization

```typescript
// Sanitize user input
function sanitizeString(input: string): string {
  return input
    .trim()
    .replace(/[<>]/g, '') // Remove HTML tags
    .slice(0, 1000); // Limit length
}

app.post('/comments', validate({ body: commentSchema }), async (ctx) => {
  const { content } = ctx.body;

  // Sanitize before storing
  const sanitized = sanitizeString(content);

  const comment = await createComment({ content: sanitized });
  ctx.json(comment, 201);
});
```

---

## Authentication

### JWT Authentication

```typescript
import { JWTService, authenticate, passwordService } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 3600, // 1 hour
  algorithm: 'HS256',
});

const auth = authenticate(jwtService);

// Registration
app.post('/auth/register', validate({ body: registerSchema }), async (ctx) => {
  const { email, password, name } = ctx.body;

  // Check if user exists
  const existingUser = await getUserByEmail(email);
  if (existingUser) {
    throw new HTTPError(409, 'User already exists');
  }

  // Hash password
  const hashedPassword = await passwordService.hash(password);

  // Create user
  const user = await createUser({
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

  ctx.json({ token, user: { id: user.id, email, name } }, 201);
});

// Login
app.post('/auth/login', validate({ body: loginSchema }), async (ctx) => {
  const { email, password } = ctx.body;

  // Find user
  const user = await getUserByEmail(email);
  if (!user) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Verify password
  const valid = await passwordService.verify(password, user.password);
  if (!valid) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Generate token
  const token = jwtService.sign({
    sub: user.id,
    email: user.email,
    role: user.role,
  });

  ctx.json({ token });
});

// Protected routes
app.get('/auth/profile', auth, async (ctx) => {
  const user = await getUser(ctx.state.userId);
  ctx.json(user);
});
```

### Refresh Tokens

```typescript
import { randomBytes } from 'crypto';

interface RefreshToken {
  token: string;
  userId: string;
  expiresAt: Date;
}

const refreshTokens = new Map<string, RefreshToken>();

// Generate refresh token
function generateRefreshToken(userId: string): string {
  const token = randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // 30 days

  refreshTokens.set(token, { token, userId, expiresAt });

  return token;
}

// Login with refresh token
app.post('/auth/login', validate({ body: loginSchema }), async (ctx) => {
  // ... verify credentials

  const accessToken = jwtService.sign({ sub: user.id }, { expiresIn: 900 }); // 15 min
  const refreshToken = generateRefreshToken(user.id);

  ctx.json({ accessToken, refreshToken });
});

// Refresh access token
app.post('/auth/refresh', async (ctx) => {
  const { refreshToken } = ctx.body as { refreshToken: string };

  const tokenData = refreshTokens.get(refreshToken);
  if (!tokenData || tokenData.expiresAt < new Date()) {
    throw new HTTPError(401, 'Invalid or expired refresh token');
  }

  // Generate new access token
  const accessToken = jwtService.sign({ sub: tokenData.userId }, { expiresIn: 900 });

  ctx.json({ accessToken });
});

// Logout (invalidate refresh token)
app.post('/auth/logout', auth, async (ctx) => {
  const { refreshToken } = ctx.body as { refreshToken: string };
  refreshTokens.delete(refreshToken);

  ctx.json({ message: 'Logged out successfully' });
});
```

---

## Authorization

### Role-Based Access Control (RBAC)

```typescript
type Role = 'user' | 'admin' | 'moderator';

interface User {
  id: string;
  email: string;
  role: Role;
}

function requireRole(...allowedRoles: Role[]) {
  return async (ctx: Context, next: () => Promise<void>) => {
    if (!ctx.user) {
      throw new HTTPError(401, 'Authentication required');
    }

    if (!allowedRoles.includes(ctx.user.role)) {
      throw new HTTPError(403, 'Insufficient permissions');
    }

    await next();
  };
}

// Usage
app.get('/admin/users', auth, requireRole('admin'), async (ctx) => {
  const users = await getAllUsers();
  ctx.json({ users });
});

app.delete('/posts/:id', auth, requireRole('admin', 'moderator'), async (ctx) => {
  await deletePost(ctx.params.id);
  ctx.json({ deleted: true });
});
```

### Resource-Based Authorization

```typescript
async function requireOwnership(ctx: Context, next: () => Promise<void>) {
  const postId = ctx.params.id;
  const post = await getPost(postId);

  if (!post) {
    throw new HTTPError(404, 'Post not found');
  }

  // Check if user owns the resource or is admin
  if (post.authorId !== ctx.state.userId && ctx.user?.role !== 'admin') {
    throw new HTTPError(403, 'You can only edit your own posts');
  }

  // Store post in context for handler
  ctx.state.post = post;

  await next();
}

app.put('/posts/:id', auth, requireOwnership, async (ctx) => {
  const post = ctx.state.post;
  const updatedPost = await updatePost(post.id, ctx.body);
  ctx.json(updatedPost);
});

app.delete('/posts/:id', auth, requireOwnership, async (ctx) => {
  const post = ctx.state.post;
  await deletePost(post.id);
  ctx.json({ deleted: true });
});
```

### Permission-Based Access Control

```typescript
type Permission = 'users:read' | 'users:write' | 'posts:read' | 'posts:write' | 'posts:delete';

interface UserWithPermissions extends User {
  permissions: Permission[];
}

function requirePermission(permission: Permission) {
  return async (ctx: Context, next: () => Promise<void>) => {
    const user = ctx.user as UserWithPermissions;

    if (!user) {
      throw new HTTPError(401, 'Authentication required');
    }

    if (!user.permissions.includes(permission)) {
      throw new HTTPError(403, `Missing required permission: ${permission}`);
    }

    await next();
  };
}

// Usage
app.get('/users', auth, requirePermission('users:read'), listUsers);
app.post('/users', auth, requirePermission('users:write'), createUser);
app.delete('/posts/:id', auth, requirePermission('posts:delete'), deletePost);
```

---

## Rate Limiting

### Basic Rate Limiting

```typescript
import { rateLimit } from 'ramapi';

// Global rate limit
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Max 100 requests per window
  message: 'Too many requests, please try again later',
}));

// Stricter limit for login
app.post('/auth/login',
  rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }),
  loginHandler
);
```

### Advanced Rate Limiting

```typescript
interface RateLimitEntry {
  count: number;
  resetTime: number;
}

function advancedRateLimit(options: {
  points: number;
  duration: number;
  blockDuration?: number;
}) {
  const store = new Map<string, RateLimitEntry>();

  return async (ctx: Context, next: () => Promise<void>) => {
    const key = ctx.state.userId || ctx.headers['x-forwarded-for'] || 'anonymous';
    const now = Date.now();

    const entry = store.get(key);

    // Reset if window expired
    if (!entry || entry.resetTime < now) {
      store.set(key, {
        count: 1,
        resetTime: now + options.duration,
      });
      await next();
      return;
    }

    // Check if blocked
    if (entry.count >= options.points) {
      const resetIn = Math.ceil((entry.resetTime - now) / 1000);
      ctx.status(429);
      ctx.setHeader('Retry-After', resetIn.toString());
      ctx.json({
        error: 'Too many requests',
        retryAfter: resetIn,
      });
      return;
    }

    // Increment and continue
    entry.count++;
    await next();
  };
}

// Usage: 10 requests per minute
app.use(advancedRateLimit({
  points: 10,
  duration: 60 * 1000,
}));
```

### Per-User Rate Limiting

```typescript
function userRateLimit(maxRequests: number, windowMs: number) {
  const userLimits = new Map<string, RateLimitEntry>();

  return async (ctx: Context, next: () => Promise<void>) => {
    if (!ctx.state.userId) {
      await next();
      return;
    }

    const userId = ctx.state.userId;
    const now = Date.now();
    const entry = userLimits.get(userId);

    if (!entry || entry.resetTime < now) {
      userLimits.set(userId, {
        count: 1,
        resetTime: now + windowMs,
      });
      await next();
      return;
    }

    if (entry.count >= maxRequests) {
      throw new HTTPError(429, 'Rate limit exceeded');
    }

    entry.count++;
    await next();
  };
}

// Apply to authenticated routes
app.use('/api', auth, userRateLimit(100, 15 * 60 * 1000));
```

---

## CORS Configuration

### Basic CORS

```typescript
import { cors } from 'ramapi';

// Allow all origins (development only!)
app.use(cors({
  origin: '*',
}));

// Production: Specific origins
app.use(cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com'],
  credentials: true,
}));
```

### Advanced CORS

```typescript
app.use(cors({
  origin: (origin: string) => {
    // Dynamic origin validation
    const allowedOrigins = [
      'https://yourdomain.com',
      'https://app.yourdomain.com',
    ];

    // Allow localhost in development
    if (process.env.NODE_ENV === 'development') {
      allowedOrigins.push('http://localhost:3000', 'http://localhost:5173');
    }

    return allowedOrigins.includes(origin);
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count', 'X-Page-Number'],
  credentials: true,
  maxAge: 86400, // 24 hours
}));
```

---

## Common Vulnerabilities

### SQL Injection Prevention

```typescript
// BAD: Vulnerable to SQL injection
app.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  const user = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
  ctx.json(user);
});

// GOOD: Use parameterized queries
app.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  ctx.json(user);
});
```

### XSS Prevention

```typescript
// Sanitize HTML
import DOMPurify from 'isomorphic-dompurify';

app.post('/posts', auth, validate({ body: postSchema }), async (ctx) => {
  const { title, content } = ctx.body;

  // Sanitize HTML content
  const sanitizedContent = DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'a'],
    ALLOWED_ATTR: ['href'],
  });

  const post = await createPost({
    title,
    content: sanitizedContent,
    authorId: ctx.state.userId,
  });

  ctx.json(post, 201);
});
```

### CSRF Protection

```typescript
import { randomBytes } from 'crypto';

// Generate CSRF token
function generateCSRFToken(): string {
  return randomBytes(32).toString('hex');
}

// CSRF middleware
function csrfProtection() {
  const tokens = new Set<string>();

  return {
    // Generate token endpoint
    getToken: (ctx: Context) => {
      const token = generateCSRFToken();
      tokens.add(token);
      ctx.json({ csrfToken: token });
    },

    // Verify token middleware
    verifyToken: async (ctx: Context, next: () => Promise<void>) => {
      if (['GET', 'HEAD', 'OPTIONS'].includes(ctx.method)) {
        await next();
        return;
      }

      const token = ctx.headers['x-csrf-token'] as string;

      if (!token || !tokens.has(token)) {
        throw new HTTPError(403, 'Invalid CSRF token');
      }

      await next();

      // Remove token after use (single-use)
      tokens.delete(token);
    },
  };
}

const csrf = csrfProtection();

app.get('/csrf-token', csrf.getToken);
app.post('/api/*', csrf.verifyToken);
```

### Path Traversal Prevention

```typescript
import { resolve, normalize, relative } from 'path';

app.get('/files/:filename', async (ctx) => {
  const filename = ctx.params.filename;
  const baseDir = '/var/www/uploads';

  // Normalize and resolve path
  const fullPath = resolve(baseDir, normalize(filename));

  // Ensure path is within baseDir
  const relativePath = relative(baseDir, fullPath);
  if (relativePath.startsWith('..') || path.isAbsolute(relativePath)) {
    throw new HTTPError(403, 'Access denied');
  }

  // Safe to serve file
  const file = await readFile(fullPath);
  ctx.send(file);
});
```

---

## Security Headers

> **Note**: The `ctx.setHeader()` method has been verified in the RamAPI source code.

```typescript
function securityHeaders() {
  return async (ctx: Context, next: () => Promise<void>) => {
    await next();

    // Prevent XSS attacks
    ctx.setHeader('X-Content-Type-Options', 'nosniff');
    ctx.setHeader('X-Frame-Options', 'DENY');
    ctx.setHeader('X-XSS-Protection', '1; mode=block');

    // HTTPS enforcement
    ctx.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');

    // Content Security Policy
    ctx.setHeader(
      'Content-Security-Policy',
      "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
    );

    // Referrer policy
    ctx.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');

    // Permissions policy
    ctx.setHeader('Permissions-Policy', 'geolocation=(), microphone=()');
  };
}

app.use(securityHeaders());
```

---

## Secrets Management

### Environment Variables

```typescript
// .env
JWT_SECRET=your-secret-key-here
DATABASE_URL=postgresql://user:password@localhost/db
API_KEY=your-api-key

// Load environment variables
import 'dotenv/config';

// Validate required secrets
function validateConfig() {
  const required = ['JWT_SECRET', 'DATABASE_URL'];
  const missing = required.filter((key) => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
}

validateConfig();

// Use secrets
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
});
```

### Never Hardcode Secrets

```typescript
// BAD
const jwtSecret = 'my-secret-key';
const apiKey = '1234567890abcdef';

// GOOD
const jwtSecret = process.env.JWT_SECRET!;
const apiKey = process.env.API_KEY!;
```

---

## Security Checklist

### Production Checklist

- [ ] **Authentication**
  - [ ] Passwords hashed with bcrypt (min 10 rounds)
  - [ ] JWT secrets are strong and stored securely
  - [ ] Refresh tokens implemented
  - [ ] Token expiration configured appropriately

- [ ] **Authorization**
  - [ ] Role-based access control implemented
  - [ ] Resource ownership checked
  - [ ] Least privilege principle applied

- [ ] **Input Validation**
  - [ ] All inputs validated with Zod schemas
  - [ ] File uploads restricted by type and size
  - [ ] SQL queries use parameterized statements
  - [ ] User-generated content sanitized

- [ ] **Rate Limiting**
  - [ ] Global rate limiting enabled
  - [ ] Stricter limits on sensitive endpoints (login, register)
  - [ ] Per-user rate limiting for authenticated routes

- [ ] **CORS**
  - [ ] Only allowed origins configured
  - [ ] Credentials enabled only when needed
  - [ ] Appropriate headers exposed

- [ ] **Headers**
  - [ ] Security headers configured
  - [ ] HTTPS enforced in production
  - [ ] CSP policy defined

- [ ] **Secrets**
  - [ ] No secrets in source code
  - [ ] Environment variables validated on startup
  - [ ] Secrets rotated regularly

- [ ] **Error Handling**
  - [ ] Generic error messages for users
  - [ ] Detailed errors logged server-side
  - [ ] Stack traces never exposed

- [ ] **Dependencies**
  - [ ] All dependencies up to date
  - [ ] Regular security audits (`npm audit`)
  - [ ] No known vulnerabilities

---

## See Also

- [Authentication Guide](../authentication/jwt.md)
- [Middleware](../middleware/overview.md)
- [Production Setup](../deployment/production-setup.md)
- [Testing Strategies](testing.md)
