# Middleware

Middleware functions are the backbone of RamAPI applications. They intercept requests, perform operations, and control the flow of execution. This guide covers everything you need to know about using and creating middleware.

## Table of Contents

1. [What is Middleware?](#what-is-middleware)
2. [Middleware Signature](#middleware-signature)
3. [Using Middleware](#using-middleware)
4. [Execution Order](#execution-order)
5. [Built-in Middleware](#built-in-middleware)
6. [Creating Custom Middleware](#creating-custom-middleware)
7. [Middleware Patterns](#middleware-patterns)
8. [Best Practices](#best-practices)

---

## What is Middleware?

Middleware functions are executed in sequence before your route handler. They can:

- **Modify the context** - Add properties to `ctx.state`, set headers, etc.
- **Validate requests** - Check authentication, validate input
- **Transform data** - Parse, sanitize, or transform request/response data
- **Handle errors** - Catch and handle errors
- **Log requests** - Record request information
- **Control flow** - Decide whether to proceed to the next middleware/handler

### Visual Flow

```
Request
  ↓
Middleware 1 (logger)
  ↓
Middleware 2 (cors)
  ↓
Middleware 3 (auth)
  ↓
Route Handler
  ↓
Response
```

---

## Middleware Signature

Middleware functions follow a specific signature:

```typescript
type Middleware = (
  ctx: Context,
  next: () => Promise<void>
) => void | Promise<void>;
```

### Parameters

- **`ctx`** - The context object (same as handlers)
- **`next`** - Function to call the next middleware/handler

### Basic Middleware

```typescript
const myMiddleware: Middleware = async (ctx, next) => {
  // Code before next() runs BEFORE the handler
  console.log('Before handler');

  await next(); // Call next middleware/handler

  // Code after next() runs AFTER the handler
  console.log('After handler');
};
```

---

## Using Middleware

Middleware can be applied globally, to route groups, or to individual routes.

### Global Middleware

Applied to all routes:

```typescript
import { createApp, logger, cors } from 'ramapi';

const app = createApp();

// Apply to all routes
app.use(logger());
app.use(cors());

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello!' });
});

// Both middlewares run for this route
```

### Group Middleware

Applied to all routes in a group:

```typescript
import { authenticate } from 'ramapi';

const jwt = new JWTService({ secret: 'secret' });

app.group('/api', (api) => {
  // Applies to all routes in this group
  api.use(authenticate(jwt));

  api.get('/profile', handler);  // Auth required
  api.get('/settings', handler); // Auth required
});
```

### Route Middleware

Applied to specific routes:

```typescript
import { validate, rateLimit } from 'ramapi';

const schema = z.object({
  email: z.string().email(),
});

// Middleware only for this route
app.post('/subscribe',
  rateLimit({ maxRequests: 5, windowMs: 60000 }),
  validate({ body: schema }),
  async (ctx) => {
    ctx.json({ message: 'Subscribed!' });
  }
);
```

### Multiple Middleware

Chain multiple middleware together:

```typescript
app.post('/admin/users',
  authenticate(jwt),        // First
  checkAdmin,              // Second
  validate({ body: schema }), // Third
  createUserHandler        // Finally
);
```

---

## Execution Order

Middleware executes in a specific order:

### Order of Execution

```typescript
// 1. Global middleware (in registration order)
app.use(middleware1);
app.use(middleware2);

// 2. Group middleware
app.group('/api', (api) => {
  api.use(middleware3);

  // 3. Route middleware
  api.get('/data',
    middleware4,
    middleware5,
    handler
  );
});

// Execution order for GET /api/data:
// middleware1 → middleware2 → middleware3 → middleware4 → middleware5 → handler
```

### Before and After Handler

```typescript
const timingMiddleware: Middleware = async (ctx, next) => {
  const start = Date.now();
  console.log('Request started');

  await next(); // Handler executes here

  const duration = Date.now() - start;
  console.log(`Request completed in ${duration}ms`);
};

app.use(timingMiddleware);

app.get('/data', async (ctx) => {
  console.log('Handler executing');
  ctx.json({ data: 'value' });
});

// Output:
// Request started
// Handler executing
// Request completed in 5ms
```

### Stopping Execution

Don't call `next()` to stop the chain:

```typescript
const authMiddleware: Middleware = async (ctx, next) => {
  const token = ctx.headers['authorization'];

  if (!token) {
    ctx.status(401).json({ error: 'Unauthorized' });
    return; // Don't call next() - stops here
  }

  await next(); // Only called if token exists
};
```

---

## Built-in Middleware

RamAPI provides several built-in middleware.

### Logger

Logs requests with status codes and duration:

```typescript
import { logger } from 'ramapi';

app.use(logger());

// Output:
// [200] GET /users - 15ms
// [404] GET /unknown - 2ms
// [500] POST /error - 120ms
```

### CORS

Enable Cross-Origin Resource Sharing:

```typescript
import { cors } from 'ramapi';

// Simple - allow all origins
app.use(cors());

// Configured
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
}));
```

### Rate Limiting

Protect against abuse:

```typescript
import { rateLimit } from 'ramapi';

// Global rate limit
app.use(rateLimit({
  maxRequests: 100,
  windowMs: 60000, // 1 minute
}));

// Per-route rate limit
app.post('/api/expensive',
  rateLimit({ maxRequests: 5, windowMs: 60000 }),
  handler
);
```

### Validation

Validate requests with Zod:

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  age: z.number().min(18),
});

app.post('/users',
  validate({ body: schema }),
  async (ctx) => {
    // ctx.body is validated and typed
    const user = ctx.body as z.infer<typeof schema>;
    ctx.json({ user }, 201);
  }
);
```

### Authentication

JWT-based authentication:

```typescript
import { JWTService, authenticate } from 'ramapi';

const jwt = new JWTService({ secret: 'secret' });

// Protected route
app.get('/profile',
  authenticate(jwt),
  async (ctx) => {
    // ctx.user contains decoded JWT
    ctx.json({ user: ctx.user });
  }
);

// Optional authentication
import { optionalAuthenticate } from 'ramapi';

app.get('/public',
  optionalAuthenticate(jwt),
  async (ctx) => {
    // ctx.user is set if token provided, undefined otherwise
    if (ctx.user) {
      ctx.json({ message: 'Hello, user!', user: ctx.user });
    } else {
      ctx.json({ message: 'Hello, guest!' });
    }
  }
);
```

---

## Creating Custom Middleware

Build your own middleware for specific needs.

### Basic Custom Middleware

```typescript
import { Middleware } from 'ramapi';

const requestId: Middleware = async (ctx, next) => {
  // Generate unique ID for this request
  const id = crypto.randomUUID();

  // Add to context state
  ctx.state.requestId = id;

  // Add to response headers
  ctx.setHeader('X-Request-ID', id);

  await next();
};

app.use(requestId);
```

### Middleware with Configuration

```typescript
interface TimingConfig {
  threshold?: number;
  logSlow?: boolean;
}

function timing(config: TimingConfig = {}): Middleware {
  const { threshold = 1000, logSlow = true } = config;

  return async (ctx, next) => {
    const start = Date.now();

    await next();

    const duration = Date.now() - start;

    if (logSlow && duration > threshold) {
      console.warn(`Slow request: ${ctx.method} ${ctx.path} - ${duration}ms`);
    }

    ctx.setHeader('X-Response-Time', `${duration}ms`);
  };
}

app.use(timing({ threshold: 500, logSlow: true }));
```

### Error Handling Middleware

```typescript
const errorHandler: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    console.error('Error caught:', error);

    if (error instanceof HTTPError) {
      ctx.json({
        error: true,
        message: error.message,
        details: error.details
      }, error.statusCode);
    } else {
      ctx.json({
        error: true,
        message: 'Internal server error'
      }, 500);
    }
  }
};

app.use(errorHandler);
```

### Conditional Middleware

```typescript
function conditionalMiddleware(condition: boolean, middleware: Middleware): Middleware {
  return async (ctx, next) => {
    if (condition) {
      await middleware(ctx, next);
    } else {
      await next();
    }
  };
}

// Use it
const isDevelopment = process.env.NODE_ENV === 'development';

app.use(conditionalMiddleware(isDevelopment, debugMiddleware));
```

### Async Data Loading

```typescript
const loadUser: Middleware = async (ctx, next) => {
  const userId = ctx.state.userId;

  if (userId) {
    const user = await database.findUser(userId);
    ctx.state.user = user;
  }

  await next();
};

app.use(authenticate(jwt));
app.use(loadUser); // Runs after auth

app.get('/profile', async (ctx) => {
  const user = ctx.state.user;
  ctx.json({ user });
});
```

---

## Middleware Patterns

Common patterns for building middleware.

### Request Transformation

```typescript
const jsonParser: Middleware = async (ctx, next) => {
  if (ctx.method === 'POST' && !ctx.body) {
    // Body already parsed by RamAPI, but you can transform it
    if (typeof ctx.body === 'string') {
      try {
        ctx.body = JSON.parse(ctx.body);
      } catch (error) {
        throw new HTTPError(400, 'Invalid JSON');
      }
    }
  }

  await next();
};
```

### Response Transformation

```typescript
const responseWrapper: Middleware = async (ctx, next) => {
  // Store original json method
  const originalJson = ctx.json;

  // Override json method
  ctx.json = (data: unknown, status = 200) => {
    const wrapped = {
      success: true,
      data,
      timestamp: new Date().toISOString()
    };

    originalJson.call(ctx, wrapped, status);
  };

  await next();
};

app.use(responseWrapper);

app.get('/data', async (ctx) => {
  ctx.json({ value: 'test' });
});

// Response:
// {
//   "success": true,
//   "data": { "value": "test" },
//   "timestamp": "2024-01-15T10:30:00.000Z"
// }
```

### Caching Middleware

```typescript
const cache = new Map<string, { data: any; expires: number }>();

function cacheMiddleware(ttl: number = 60000): Middleware {
  return async (ctx, next) => {
    const key = `${ctx.method}:${ctx.path}`;

    // Check cache
    const cached = cache.get(key);
    if (cached && cached.expires > Date.now()) {
      ctx.json(cached.data);
      return; // Don't call next()
    }

    // Store original json
    const originalJson = ctx.json;

    // Override to cache response
    ctx.json = (data: unknown, status = 200) => {
      if (status === 200) {
        cache.set(key, {
          data,
          expires: Date.now() + ttl
        });
      }

      originalJson.call(ctx, data, status);
    };

    await next();
  };
}

app.get('/expensive',
  cacheMiddleware(300000), // Cache for 5 minutes
  async (ctx) => {
    const data = await expensiveOperation();
    ctx.json(data);
  }
);
```

### Security Headers

```typescript
const securityHeaders: Middleware = async (ctx, next) => {
  ctx.setHeader('X-Content-Type-Options', 'nosniff');
  ctx.setHeader('X-Frame-Options', 'DENY');
  ctx.setHeader('X-XSS-Protection', '1; mode=block');
  ctx.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');

  await next();
};

app.use(securityHeaders);
```

### Request Sanitization

```typescript
const sanitizeInput: Middleware = async (ctx, next) => {
  if (ctx.body && typeof ctx.body === 'object') {
    // Remove any properties starting with $
    const sanitized = Object.keys(ctx.body).reduce((acc, key) => {
      if (!key.startsWith('$')) {
        acc[key] = ctx.body[key];
      }
      return acc;
    }, {} as any);

    ctx.body = sanitized;
  }

  await next();
};
```

---

## Best Practices

### 1. Always Call next()

Unless you're ending the request:

```typescript
// Good - calls next()
const middleware: Middleware = async (ctx, next) => {
  console.log('Before');
  await next();
  console.log('After');
};

// Good - ends request (no next())
const authMiddleware: Middleware = async (ctx, next) => {
  if (!isAuthorized) {
    ctx.status(401).json({ error: 'Unauthorized' });
    return; // Don't call next()
  }
  await next();
};

// Bad - doesn't call next() and doesn't end request
const badMiddleware: Middleware = async (ctx, next) => {
  console.log('Before');
  // Forgot to call next() - request hangs!
};
```

### 2. Order Matters

Place middleware in logical order:

```typescript
// Good order
app.use(logger());           // 1. Log first
app.use(errorHandler());     // 2. Catch errors
app.use(cors());             // 3. CORS
app.use(authenticate(jwt));  // 4. Auth
app.use(rateLimit());        // 5. Rate limit authenticated users

// Bad order
app.use(authenticate(jwt));  // Auth before CORS fails
app.use(cors());
```

### 3. Use Async/Await

Always use `async/await` with `next()`:

```typescript
// Good
const middleware: Middleware = async (ctx, next) => {
  await next();
};

// Bad - may cause issues
const middleware: Middleware = (ctx, next) => {
  next(); // Should be awaited!
};
```

### 4. Keep Middleware Focused

Each middleware should do one thing:

```typescript
// Good - single purpose
const requestId: Middleware = async (ctx, next) => {
  ctx.state.requestId = crypto.randomUUID();
  await next();
};

// Bad - too many responsibilities
const doEverything: Middleware = async (ctx, next) => {
  ctx.state.requestId = crypto.randomUUID();
  ctx.state.user = await getUser();
  ctx.state.settings = await getSettings();
  // ... too much
  await next();
};
```

### 5. Handle Errors Properly

```typescript
const safeMiddleware: Middleware = async (ctx, next) => {
  try {
    // Your logic
    const data = await fetchData();
    ctx.state.data = data;

    await next();
  } catch (error) {
    // Let error propagate or handle it
    throw error;
  }
};
```

### 6. Use State for Communication

```typescript
// Middleware sets state
const authMiddleware: Middleware = async (ctx, next) => {
  const user = await authenticateUser(ctx);
  ctx.state.userId = user.id;
  ctx.state.userRole = user.role;
  await next();
};

// Handler uses state
app.get('/data', async (ctx) => {
  const userId = ctx.state.userId;
  const data = await fetchDataForUser(userId);
  ctx.json(data);
});
```

### 7. Document Your Middleware

```typescript
/**
 * Rate limiting middleware
 *
 * Limits requests per IP address using a sliding window algorithm.
 * Stores rate limit data in memory (use Redis for production).
 *
 * @param config - Configuration options
 * @param config.maxRequests - Maximum requests allowed in window
 * @param config.windowMs - Time window in milliseconds
 *
 * @example
 * app.use(rateLimit({ maxRequests: 100, windowMs: 60000 }));
 */
export function rateLimit(config: RateLimitConfig): Middleware {
  // Implementation
}
```

---

## Common Use Cases

### API Key Authentication

```typescript
const apiKeyAuth: Middleware = async (ctx, next) => {
  const apiKey = ctx.headers['x-api-key'];

  if (!apiKey) {
    throw new HTTPError(401, 'API key required');
  }

  const valid = await validateApiKey(apiKey);

  if (!valid) {
    throw new HTTPError(403, 'Invalid API key');
  }

  ctx.state.apiKey = apiKey;
  await next();
};
```

### Request Body Size Limit

```typescript
function bodyLimit(maxBytes: number = 1024 * 1024): Middleware {
  return async (ctx, next) => {
    const contentLength = parseInt(ctx.headers['content-length'] || '0', 10);

    if (contentLength > maxBytes) {
      throw new HTTPError(413, 'Request entity too large');
    }

    await next();
  };
}

app.use(bodyLimit(5 * 1024 * 1024)); // 5MB limit
```

### Conditional Logging

```typescript
const conditionalLogger: Middleware = async (ctx, next) => {
  const shouldLog = !ctx.path.startsWith('/health');

  const start = shouldLog ? Date.now() : 0;

  await next();

  if (shouldLog) {
    const duration = Date.now() - start;
    console.log(`[${ctx.res.statusCode}] ${ctx.method} ${ctx.path} - ${duration}ms`);
  }
};
```

---

## Next Steps

- [Learn about Validation](validation.md)
- [Explore Error Handling](error-handling.md)
- [See Built-in Middleware Reference](../middleware/built-in-middleware.md)
- [Create Custom Middleware](../middleware/custom-middleware.md)

---

**Need help?** Check the [Middleware API Reference](../api-reference/middleware-api.md) for complete documentation.
