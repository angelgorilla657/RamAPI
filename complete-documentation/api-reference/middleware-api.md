# Middleware API Reference

Complete API reference for RamAPI's built-in middleware and middleware patterns.

## Table of Contents

1. [Middleware Function Signature](#middleware-function-signature)
2. [cors()](#cors)
3. [logger()](#logger)
4. [rateLimit()](#ratelimit)
5. [Custom Middleware](#custom-middleware)
6. [Middleware Patterns](#middleware-patterns)

---

## Middleware Function Signature

### Type Definition

```typescript
type Middleware = (
  ctx: Context,
  next: () => Promise<void>
) => void | Promise<void>;
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ctx` | `Context` | Request/response context |
| `next` | `() => Promise<void>` | Function to call next middleware |

### Example

```typescript
const myMiddleware: Middleware = async (ctx, next) => {
  // Before request handling
  console.log('Before:', ctx.path);

  await next(); // Pass control to next middleware/handler

  // After request handling
  console.log('After:', ctx.path);
};

app.use(myMiddleware);
```

---

## cors()

CORS (Cross-Origin Resource Sharing) middleware.

### Signature

```typescript
function cors(config?: CorsConfig): Middleware
```

### Configuration

```typescript
interface CorsConfig {
  origin?: string | string[] | ((origin: string) => boolean);
  methods?: HTTPMethod[];
  allowedHeaders?: string[];
  exposedHeaders?: string[];
  credentials?: boolean;
  maxAge?: number;
}
```

### Options

#### origin

- **Type:** `string | string[] | ((origin: string) => boolean)`
- **Default:** `'*'`
- **Description:** Allowed origin(s)

```typescript
// Single origin
app.use(cors({ origin: 'https://example.com' }));

// Multiple origins
app.use(cors({ origin: ['https://example.com', 'https://app.example.com'] }));

// Dynamic origin
app.use(cors({
  origin: (origin) => origin.endsWith('.example.com'),
}));
```

#### methods

- **Type:** `HTTPMethod[]`
- **Default:** `['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS']`
- **Description:** Allowed HTTP methods

```typescript
app.use(cors({
  methods: ['GET', 'POST'],
}));
```

#### allowedHeaders

- **Type:** `string[]`
- **Default:** `['Content-Type', 'Authorization']`
- **Description:** Allowed request headers

```typescript
app.use(cors({
  allowedHeaders: ['Content-Type', 'Authorization', 'X-API-Key'],
}));
```

#### exposedHeaders

- **Type:** `string[]`
- **Default:** `[]`
- **Description:** Headers exposed to the client

```typescript
app.use(cors({
  exposedHeaders: ['X-Total-Count', 'X-Page-Number'],
}));
```

#### credentials

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Allow credentials (cookies, auth headers)

```typescript
app.use(cors({
  credentials: true,
  origin: 'https://example.com', // Must specify origin when credentials = true
}));
```

#### maxAge

- **Type:** `number`
- **Default:** `86400` (24 hours)
- **Description:** Preflight cache duration in seconds

```typescript
app.use(cors({
  maxAge: 3600, // 1 hour
}));
```

### Examples

**Basic CORS:**

```typescript
import { createApp, cors } from 'ramapi';

const app = createApp();

app.use(cors()); // Allow all origins

app.get('/api/data', async (ctx) => {
  ctx.json({ data: 'public' });
});
```

**Restricted CORS:**

```typescript
app.use(cors({
  origin: 'https://example.com',
  methods: ['GET', 'POST'],
  credentials: true,
}));
```

**Dynamic CORS:**

```typescript
app.use(cors({
  origin: (origin) => {
    // Allow all subdomains
    return origin.endsWith('.example.com');
  },
  credentials: true,
}));
```

---

## logger()

Request logging middleware with colored output.

### Signature

```typescript
function logger(): Middleware
```

### Features

- Logs HTTP method, path, status code, and duration
- Color-coded status codes:
  - Green (2xx): Success
  - Yellow (4xx): Client errors
  - Red (5xx): Server errors
- Logs errors with stack traces

### Example

```typescript
import { createApp, logger } from 'ramapi';

const app = createApp();

app.use(logger());

app.get('/users', async (ctx) => {
  ctx.json({ users: [] });
});

await app.listen(3000);

// Output:
// [200] GET /users - 12ms
```

### Output Format

```
[<status>] <method> <path> - <duration>ms
```

**Examples:**

```
[200] GET /users - 5ms
[201] POST /users - 23ms
[404] GET /notfound - 2ms
[500] POST /error - 15ms
[ERROR] GET /crash - 10ms Error: Something went wrong
```

---

## rateLimit()

Rate limiting middleware to prevent abuse.

### Signature

```typescript
function rateLimit(config?: RateLimitConfig): Middleware
```

### Configuration

```typescript
interface RateLimitConfig {
  windowMs?: number;
  maxRequests?: number;
  keyGenerator?: (ctx: Context) => string;
  message?: string;
  skipSuccessfulRequests?: boolean;
  skipFailedRequests?: boolean;
}
```

### Options

#### windowMs

- **Type:** `number`
- **Default:** `60000` (1 minute)
- **Description:** Time window in milliseconds

```typescript
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
}));
```

#### maxRequests

- **Type:** `number`
- **Default:** `100`
- **Description:** Maximum requests per window

```typescript
app.use(rateLimit({
  maxRequests: 1000, // 1000 requests per window
}));
```

#### keyGenerator

- **Type:** `(ctx: Context) => string`
- **Default:** IP address
- **Description:** Function to generate unique key per client

```typescript
app.use(rateLimit({
  keyGenerator: (ctx) => {
    // Rate limit by user ID
    return ctx.user?.id || 'anonymous';
  },
}));

app.use(rateLimit({
  keyGenerator: (ctx) => {
    // Rate limit by API key
    return ctx.headers['x-api-key'] || 'unknown';
  },
}));
```

#### message

- **Type:** `string`
- **Default:** `'Too many requests, please try again later'`
- **Description:** Custom error message

```typescript
app.use(rateLimit({
  message: 'Slow down! Too many requests.',
}));
```

#### skipSuccessfulRequests

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Don't count successful requests (2xx)

```typescript
app.use(rateLimit({
  skipSuccessfulRequests: true, // Only count failed requests
}));
```

#### skipFailedRequests

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Don't count failed requests (4xx, 5xx)

```typescript
app.use(rateLimit({
  skipFailedRequests: true, // Only count successful requests
}));
```

### Response Headers

Rate limit middleware sets these headers:

- `X-RateLimit-Limit`: Maximum requests per window
- `X-RateLimit-Remaining`: Remaining requests in current window
- `X-RateLimit-Reset`: Time when the window resets (ISO 8601)
- `Retry-After`: Seconds until client can retry (when limited)

### Examples

**Basic rate limiting:**

```typescript
import { createApp, rateLimit } from 'ramapi';

const app = createApp();

// 100 requests per minute per IP
app.use(rateLimit());

app.get('/api/data', async (ctx) => {
  ctx.json({ data: 'limited' });
});
```

**Custom limits:**

```typescript
// Strict rate limit
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  maxRequests: 100,
}));

// Generous rate limit
app.use(rateLimit({
  windowMs: 60 * 1000, // 1 minute
  maxRequests: 1000,
}));
```

**Per-route rate limiting:**

```typescript
// Global: generous limit
app.use(rateLimit({ maxRequests: 1000 }));

// API route: strict limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  maxRequests: 100,
});

app.get('/api/sensitive', apiLimiter, async (ctx) => {
  ctx.json({ data: 'protected' });
});
```

**Rate limit by user:**

```typescript
const userLimiter = rateLimit({
  keyGenerator: (ctx) => {
    // Authenticated users get higher limits
    if (ctx.user) {
      return `user:${ctx.user.id}`;
    }
    // Anonymous users share a limit
    return 'anonymous';
  },
  maxRequests: 1000, // Per user
});

app.get('/api/data',
  authenticate, // Optional
  userLimiter,
  async (ctx) => {
    ctx.json({ data: 'value' });
  }
);
```

---

## Custom Middleware

### Creating Custom Middleware

```typescript
import { Middleware, Context } from 'ramapi';

// Simple middleware
export function requestId(): Middleware {
  return async (ctx, next) => {
    ctx.state.requestId = Math.random().toString(36).substring(7);
    await next();
  };
}

// Middleware with configuration
export interface CacheConfig {
  ttl: number;
  key?: (ctx: Context) => string;
}

export function cache(config: CacheConfig): Middleware {
  const cache = new Map<string, { data: any; expires: number }>();

  return async (ctx, next) => {
    const key = config.key ? config.key(ctx) : ctx.url;
    const cached = cache.get(key);

    if (cached && Date.now() < cached.expires) {
      ctx.json(cached.data);
      return;
    }

    await next();

    // Cache response (simplified)
    cache.set(key, {
      data: { /* response data */ },
      expires: Date.now() + config.ttl,
    });
  };
}
```

### Using Custom Middleware

```typescript
app.use(requestId());
app.use(cache({ ttl: 60000 }));
```

---

## Middleware Patterns

### Before/After Pattern

```typescript
const timing: Middleware = async (ctx, next) => {
  const start = Date.now();

  await next(); // Execute route handler

  const duration = Date.now() - start;
  ctx.setHeader('X-Response-Time', `${duration}ms`);
};

app.use(timing);
```

### Early Return Pattern

```typescript
const authRequired: Middleware = async (ctx, next) => {
  if (!ctx.headers.authorization) {
    ctx.json({ error: 'Unauthorized' }, 401);
    return; // Don't call next()
  }

  await next();
};

app.use(authRequired);
```

### State Sharing Pattern

```typescript
const addUser: Middleware = async (ctx, next) => {
  const token = ctx.headers.authorization;
  const user = await verifyToken(token);

  ctx.state.user = user; // Share with other middleware/handlers

  await next();
};

app.use(addUser);

app.get('/profile', async (ctx) => {
  const user = ctx.state.user; // Access shared state
  ctx.json({ profile: user });
});
```

### Error Handling Pattern

```typescript
const errorHandler: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    console.error('Error:', error);

    ctx.json({
      error: error.message,
    }, 500);
  }
};

app.use(errorHandler);
```

### Conditional Middleware Pattern

```typescript
const conditionalLogger: Middleware = async (ctx, next) => {
  if (process.env.NODE_ENV === 'development') {
    console.log(`${ctx.method} ${ctx.path}`);
  }

  await next();
};

app.use(conditionalLogger);
```

### Composition Pattern

```typescript
function compose(...middleware: Middleware[]): Middleware {
  return async (ctx, next) => {
    let index = 0;

    const dispatch = async (): Promise<void> => {
      if (index < middleware.length) {
        await middleware[index++](ctx, dispatch);
      } else {
        await next();
      }
    };

    await dispatch();
  };
}

// Compose multiple middleware
const authStack = compose(
  authenticate,
  requireRole('admin'),
  logger()
);

app.use('/admin', authStack);
```

---

## Complete Example

```typescript
import { createApp, cors, logger, rateLimit, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

// Global middleware
app.use(logger());
app.use(cors({
  origin: 'https://example.com',
  credentials: true,
}));
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  maxRequests: 100,
}));

// Custom middleware
const requestId = async (ctx, next) => {
  ctx.state.requestId = Math.random().toString(36).substring(7);
  ctx.setHeader('X-Request-ID', ctx.state.requestId);
  await next();
};

app.use(requestId);

// Route-specific middleware
const adminOnly = async (ctx, next) => {
  if (!ctx.user?.isAdmin) {
    ctx.json({ error: 'Forbidden' }, 403);
    return;
  }
  await next();
};

// Routes with middleware
app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});

app.post('/api/users',
  validate({
    body: z.object({
      name: z.string(),
      email: z.string().email(),
    }),
  }),
  async (ctx) => {
    ctx.json({ user: ctx.body }, 201);
  }
);

app.delete('/api/users/:id',
  authenticate,
  adminOnly,
  async (ctx) => {
    ctx.status(204);
  }
);

await app.listen(3000);
```

---

## See Also

- [CORS Middleware](../middleware/cors.md) - CORS middleware guide
- [Logger Middleware](../middleware/logger.md) - Logger middleware guide
- [Rate Limiting](../middleware/rate-limiting.md) - Rate limiting guide
- [Custom Middleware](../middleware/custom-middleware.md) - Creating custom middleware

---

**Need help?** Check the [Middleware Guide](../core-concepts/middleware.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
