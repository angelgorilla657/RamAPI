# Built-in Middleware Reference

RamAPI comes with several built-in middleware functions for common use cases. This guide provides a complete reference with examples for each middleware.

## Table of Contents

1. [Overview](#overview)
2. [Validation Middleware](#validation-middleware)
3. [CORS Middleware](#cors-middleware)
4. [Logger Middleware](#logger-middleware)
5. [Rate Limit Middleware](#rate-limit-middleware)
6. [Authentication Middleware](#authentication-middleware)
7. [Combining Middleware](#combining-middleware)

---

## Overview

All built-in middleware can be imported from `ramapi`:

```typescript
import {
  validate,
  cors,
  logger,
  rateLimit,
  authenticate,
  optionalAuthenticate,
} from 'ramapi';
```

---

## Validation Middleware

Validates request data using Zod schemas.

### Import

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';
```

### Signature

```typescript
function validate(schema: ValidationSchema): Middleware

interface ValidationSchema {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}
```

### Basic Usage

```typescript
import { z } from 'zod';

const createUserSchema = {
  body: z.object({
    name: z.string().min(1),
    email: z.string().email(),
    age: z.number().int().min(18),
  }),
};

app.post('/users',
  validate(createUserSchema),
  async (ctx) => {
    // ctx.body is now typed and validated
    ctx.json({ user: ctx.body });
  }
);
```

### Validate Query Parameters

```typescript
const searchSchema = {
  query: z.object({
    q: z.string().min(1),
    page: z.string().regex(/^\d+$/).transform(Number).default('1'),
    limit: z.string().regex(/^\d+$/).transform(Number).default('10'),
  }),
};

app.get('/search',
  validate(searchSchema),
  async (ctx) => {
    const { q, page, limit } = ctx.query;
    ctx.json({ results: [], page, limit });
  }
);
```

### Validate Route Parameters

```typescript
const getUserSchema = {
  params: z.object({
    id: z.string().uuid(),
  }),
};

app.get('/users/:id',
  validate(getUserSchema),
  async (ctx) => {
    const { id } = ctx.params;
    ctx.json({ user: { id } });
  }
);
```

### Complete Validation

```typescript
const updateUserSchema = {
  params: z.object({
    id: z.string().uuid(),
  }),
  body: z.object({
    name: z.string().min(1).optional(),
    email: z.string().email().optional(),
  }),
  query: z.object({
    notify: z.string().transform((val) => val === 'true').default('false'),
  }),
};

app.patch('/users/:id',
  validate(updateUserSchema),
  async (ctx) => {
    const { id } = ctx.params;
    const updates = ctx.body;
    const { notify } = ctx.query;

    ctx.json({ user: { id, ...updates }, notified: notify });
  }
);
```

**See:** [Validation Guide](../core-concepts/validation.md)

---

## CORS Middleware

Handles Cross-Origin Resource Sharing headers.

### Import

```typescript
import { cors } from 'ramapi';
```

### Signature

```typescript
function cors(config?: CorsConfig): Middleware

interface CorsConfig {
  origin?: string | string[] | ((origin: string) => boolean);
  methods?: HTTPMethod[];
  allowedHeaders?: string[];
  exposedHeaders?: string[];
  credentials?: boolean;
  maxAge?: number;
}
```

### Basic Usage

```typescript
// Allow all origins (development)
app.use(cors());

// Allow specific origins (production)
app.use(cors({
  origin: ['https://example.com'],
  credentials: true,
}));
```

### Complete Configuration

```typescript
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Request-ID'],
  credentials: true,
  maxAge: 86400, // 24 hours
}));
```

### Dynamic Origin

```typescript
app.use(cors({
  origin: (requestOrigin) => {
    const allowedOrigins = ['https://example.com', 'https://app.example.com'];
    return allowedOrigins.includes(requestOrigin);
  },
  credentials: true,
}));
```

**See:** [CORS Middleware Guide](cors.md)

---

## Logger Middleware

Logs incoming requests with response times and status codes.

### Import

```typescript
import { logger } from 'ramapi';
```

### Signature

```typescript
function logger(): Middleware
```

### Basic Usage

```typescript
app.use(logger());
```

### Output

```
[200] GET / - 5ms
[404] GET /not-found - 2ms
[500] POST /error - 15ms
```

### Custom Logger

```typescript
function customLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    try {
      await next();
      const duration = Date.now() - start;
      console.log(`${ctx.method} ${ctx.path} - ${duration}ms`);
    } catch (error) {
      console.error(`${ctx.method} ${ctx.path} - Error:`, error);
      throw error;
    }
  };
}

app.use(customLogger());
```

**See:** [Logger Middleware Guide](logger.md)

---

## Rate Limit Middleware

Protects API from abuse by limiting request frequency.

### Import

```typescript
import { rateLimit } from 'ramapi';
```

### Signature

```typescript
function rateLimit(config?: RateLimitConfig): Middleware

interface RateLimitConfig {
  windowMs?: number;                    // Time window (default: 60000)
  maxRequests?: number;                 // Max requests (default: 100)
  keyGenerator?: (ctx: Context) => string; // Key function
  message?: string;                     // Error message
  skipSuccessfulRequests?: boolean;
  skipFailedRequests?: boolean;
}
```

### Basic Usage

```typescript
// 100 requests per minute per IP
app.use(rateLimit());

// Custom limits
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  maxRequests: 100,
}));
```

### Per-Route Rate Limiting

```typescript
// Strict limit for login
app.post('/auth/login',
  rateLimit({
    windowMs: 15 * 60 * 1000,
    maxRequests: 5,
    message: 'Too many login attempts',
  }),
  loginHandler
);

// Normal limit for API
app.use('/api', rateLimit({
  windowMs: 60000,
  maxRequests: 100,
}));
```

### Custom Key Generator

```typescript
app.use(rateLimit({
  windowMs: 60000,
  maxRequests: 100,
  keyGenerator: (ctx) => {
    // Rate limit by API key instead of IP
    return ctx.headers['x-api-key'] as string || 'anonymous';
  },
}));
```

**See:** [Rate Limiting Guide](rate-limiting.md)

---

## Authentication Middleware

JWT-based authentication middleware.

### Import

```typescript
import { authenticate, optionalAuthenticate, JWTService } from 'ramapi';
```

### Signature

```typescript
function authenticate(jwtService: JWTService): Middleware
function optionalAuthenticate(jwtService: JWTService): Middleware

class JWTService {
  constructor(config: JWTConfig);
  sign(payload: JWTPayload): string;
  verify(token: string): JWTPayload;
  decode(token: string): JWTPayload | null;
}

interface JWTConfig {
  secret: string;
  expiresIn?: number;
  algorithm?: 'HS256' | 'HS384' | 'HS512' | 'RS256' | ...;
  issuer?: string;
  audience?: string;
}
```

### Basic Usage

```typescript
import { JWTService, authenticate } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
});

// Protected route
app.get('/api/profile',
  authenticate(jwtService),
  async (ctx) => {
    ctx.json({ user: ctx.user });
  }
);
```

### Generate Token

```typescript
app.post('/auth/login', async (ctx) => {
  const { email, password } = ctx.body;

  // Verify credentials (pseudo-code)
  const user = await verifyCredentials(email, password);

  if (!user) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Generate JWT token
  const token = jwtService.sign({
    sub: user.id,
    email: user.email,
    role: user.role,
  });

  ctx.json({ token });
});
```

### Optional Authentication

Use when you want to authenticate if a token is present, but not require it:

```typescript
app.get('/api/posts',
  optionalAuthenticate(jwtService),
  async (ctx) => {
    // ctx.user is populated if token was provided
    const posts = ctx.user
      ? await getPrivatePosts(ctx.user.id)
      : await getPublicPosts();

    ctx.json({ posts });
  }
);
```

### Complete Configuration

```typescript
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
  algorithm: 'HS256',
  issuer: 'my-api',
  audience: 'my-app',
});
```

**See:** [Authentication Guide](../guides/authentication.md)

---

## Combining Middleware

### Multiple Middleware on Route

```typescript
app.post('/api/users',
  logger(),
  rateLimit({ maxRequests: 10 }),
  cors({ origin: ['https://example.com'] }),
  validate(createUserSchema),
  authenticate(jwtService),
  async (ctx) => {
    // All middleware applied in order
    ctx.json({ user: ctx.body });
  }
);
```

### Global Middleware

```typescript
const app = createApp();

// Apply to all routes
app.use(logger());
app.use(cors());
app.use(rateLimit());
```

### Route Groups

```typescript
// Public routes
app.group('/api/public', (router) => {
  router.use(cors());
  router.use(rateLimit({ maxRequests: 50 }));

  router.get('/posts', getPublicPosts);
});

// Private routes
app.group('/api/private', (router) => {
  router.use(cors({ origin: ['https://app.example.com'] }));
  router.use(authenticate(jwtService));
  router.use(rateLimit({ maxRequests: 500 }));

  router.get('/profile', getProfile);
  router.put('/profile', validate(updateProfileSchema), updateProfile);
});
```

### Conditional Middleware

```typescript
// Only apply in production
if (process.env.NODE_ENV === 'production') {
  app.use(rateLimit());
}

// Only log in development
if (process.env.NODE_ENV !== 'production') {
  app.use(logger());
}
```

### Middleware Order Matters

```typescript
// CORRECT ORDER
app.use(logger());          // 1. Log first
app.use(cors());            // 2. Handle CORS
app.use(rateLimit());       // 3. Rate limit
app.use(authenticate());    // 4. Authenticate
app.use(validate(schema));  // 5. Validate

// Routes
app.get('/api/data', handler);
```

---

## Quick Reference Table

| Middleware | Purpose | Default Config | Production Ready |
|------------|---------|----------------|------------------|
| `validate()` | Zod schema validation | N/A | Yes |
| `cors()` | CORS headers | Allow all origins | Configure origins |
| `logger()` | Request logging | Colored console output | Use structured logging |
| `rateLimit()` | Rate limiting | 100 req/min per IP | Use Redis |
| `authenticate()` | JWT authentication | N/A | Yes |

---

## Complete Example

```typescript
import {
  createApp,
  validate,
  cors,
  logger,
  rateLimit,
  authenticate,
  JWTService,
} from 'ramapi';
import { z } from 'zod';

const app = createApp();

// Global middleware
app.use(logger());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['*'],
  credentials: true,
}));

// JWT setup
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400,
});

// Public routes
app.post('/auth/login',
  rateLimit({ maxRequests: 5 }),
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(8),
    }),
  }),
  async (ctx) => {
    const token = jwtService.sign({ sub: 'user-id' });
    ctx.json({ token });
  }
);

// Protected routes
app.get('/api/profile',
  authenticate(jwtService),
  rateLimit({ maxRequests: 100 }),
  async (ctx) => {
    ctx.json({ user: ctx.user });
  }
);

app.listen(3000);
```

---

## Next Steps

- [Custom Middleware Guide](custom-middleware.md)
- [Validation Documentation](../core-concepts/validation.md)
- [Authentication Guide](../guides/authentication.md)
- [Security Best Practices](../guides/security.md)

---

**Need help?** Check the [API Reference](../api-reference/middleware.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
