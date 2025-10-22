# Custom Middleware Guide

Learn how to create custom middleware for RamAPI. This guide covers middleware patterns, best practices, and real-world examples.

## Table of Contents

1. [Middleware Basics](#middleware-basics)
2. [Creating Custom Middleware](#creating-custom-middleware)
3. [Middleware Patterns](#middleware-patterns)
4. [Advanced Techniques](#advanced-techniques)
5. [Real-World Examples](#real-world-examples)
6. [Best Practices](#best-practices)
7. [Testing Middleware](#testing-middleware)

---

## Middleware Basics

### What is Middleware?

Middleware functions intercept requests before they reach route handlers. They can:
- Modify the context
- Validate data
- Handle authentication
- Log requests
- Transform responses
- Short-circuit request flow

### Middleware Signature

```typescript
type Middleware = (
  ctx: Context,
  next: () => Promise<void>
) => void | Promise<void>;
```

- **ctx**: Context object with request/response data
- **next**: Function to call the next middleware/handler

### Execution Flow

```typescript
app.use(middleware1); // Executes first
app.use(middleware2); // Executes second
app.use(middleware3); // Executes third

app.get('/route', handler); // Executes last
```

---

## Creating Custom Middleware

### Basic Middleware

```typescript
import type { Middleware } from 'ramapi';

function simpleMiddleware(): Middleware {
  return async (ctx, next) => {
    console.log('Before handler');
    await next(); // Call next middleware/handler
    console.log('After handler');
  };
}

app.use(simpleMiddleware());
```

### Middleware with Configuration

```typescript
interface MyMiddlewareConfig {
  option1: string;
  option2: number;
}

function myMiddleware(config: MyMiddlewareConfig): Middleware {
  return async (ctx, next) => {
    // Use config
    console.log('Config:', config);
    await next();
  };
}

app.use(myMiddleware({
  option1: 'value',
  option2: 42,
}));
```

### Modify Context

```typescript
function addTimestamp(): Middleware {
  return async (ctx, next) => {
    // Add data to context
    ctx.state.timestamp = Date.now();
    await next();
  };
}

app.use(addTimestamp());

app.get('/', async (ctx) => {
  // Access added data
  ctx.json({ timestamp: ctx.state.timestamp });
});
```

### Short-Circuit Execution

```typescript
function authCheck(): Middleware {
  return async (ctx, next) => {
    const token = ctx.headers.authorization;

    if (!token) {
      // Don't call next() - stop here
      ctx.json({ error: 'Unauthorized' }, 401);
      return;
    }

    await next();
  };
}
```

---

## Middleware Patterns

### 1. Request Validation

```typescript
function validateContentType(expected: string): Middleware {
  return async (ctx, next) => {
    const contentType = ctx.headers['content-type'];

    if (!contentType || !contentType.includes(expected)) {
      throw new HTTPError(
        415,
        `Content-Type must be ${expected}`
      );
    }

    await next();
  };
}

app.post('/api/data',
  validateContentType('application/json'),
  handler
);
```

### 2. Request Transformation

```typescript
function parseJsonBody(): Middleware {
  return async (ctx, next) => {
    if (ctx.method === 'POST' || ctx.method === 'PUT') {
      try {
        // ctx.body is already parsed by RamAPI
        // This is just an example
        if (typeof ctx.body === 'string') {
          ctx.body = JSON.parse(ctx.body);
        }
      } catch (error) {
        throw new HTTPError(400, 'Invalid JSON');
      }
    }

    await next();
  };
}
```

### 3. Response Transformation

```typescript
function addResponseHeaders(): Middleware {
  return async (ctx, next) => {
    await next(); // Execute handler first

    // Add headers after handler
    ctx.setHeader('X-Powered-By', 'RamAPI');
    ctx.setHeader('X-Response-Time', Date.now().toString());
  };
}
```

### 4. Error Handling

```typescript
function errorHandler(): Middleware {
  return async (ctx, next) => {
    try {
      await next();
    } catch (error) {
      const err = error as Error;

      // Log error
      console.error('Request error:', err);

      // Send error response
      if (error instanceof HTTPError) {
        ctx.json({
          error: true,
          message: err.message,
        }, error.statusCode);
      } else {
        ctx.json({
          error: true,
          message: 'Internal server error',
        }, 500);
      }
    }
  };
}

// Apply as first middleware
app.use(errorHandler());
```

### 5. Timing/Performance

```typescript
function performanceMonitor(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    await next();

    const duration = Date.now() - start;

    // Warn on slow requests
    if (duration > 1000) {
      console.warn(`Slow request: ${ctx.method} ${ctx.path} - ${duration}ms`);
    }

    // Add to response header
    ctx.setHeader('X-Response-Time', `${duration}ms`);
  };
}
```

---

## Advanced Techniques

### Async Operations

```typescript
function databaseMiddleware(): Middleware {
  return async (ctx, next) => {
    // Perform async operation
    const dbConnection = await connectToDatabase();

    // Add to context state
    ctx.state.db = dbConnection;

    try {
      await next();
    } finally {
      // Cleanup
      await dbConnection.close();
    }
  };
}
```

### Conditional Execution

```typescript
function conditionalMiddleware(condition: boolean): Middleware {
  return async (ctx, next) => {
    if (condition) {
      // Do something
      console.log('Condition met');
    }
    await next();
  };
}

// Only apply in development
app.use(conditionalMiddleware(process.env.NODE_ENV === 'development'));
```

### Composing Middleware

```typescript
function composeMiddleware(...middleware: Middleware[]): Middleware {
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

// Usage
const combined = composeMiddleware(
  logger(),
  cors(),
  rateLimit()
);

app.use(combined);
```

### Context State Management

```typescript
function userContext(): Middleware {
  return async (ctx, next) => {
    // Initialize user state
    ctx.state.user = {
      isAuthenticated: false,
      permissions: [],
    };

    await next();
  };
}

function checkPermission(permission: string): Middleware {
  return async (ctx, next) => {
    const { permissions } = ctx.state.user;

    if (!permissions.includes(permission)) {
      throw new HTTPError(403, 'Forbidden');
    }

    await next();
  };
}

app.use(userContext());
app.get('/admin', checkPermission('admin'), adminHandler);
```

---

## Real-World Examples

### 1. Request ID Middleware

```typescript
import { randomUUID } from 'crypto';

function requestId(): Middleware {
  return async (ctx, next) => {
    // Get or generate request ID
    const requestId = ctx.headers['x-request-id'] as string || randomUUID();

    // Add to context
    ctx.state.requestId = requestId;

    // Add to response headers
    ctx.setHeader('X-Request-ID', requestId);

    await next();
  };
}

app.use(requestId());
```

### 2. API Key Authentication

```typescript
interface ApiKeyConfig {
  header?: string;
  validKeys: string[];
}

function apiKeyAuth(config: ApiKeyConfig): Middleware {
  const header = config.header || 'x-api-key';

  return async (ctx, next) => {
    const apiKey = ctx.headers[header] as string;

    if (!apiKey) {
      throw new HTTPError(401, 'API key required');
    }

    if (!config.validKeys.includes(apiKey)) {
      throw new HTTPError(401, 'Invalid API key');
    }

    // Add API key info to context
    ctx.state.apiKey = apiKey;

    await next();
  };
}

app.use(apiKeyAuth({
  validKeys: process.env.API_KEYS?.split(',') || [],
}));
```

### 3. Request Size Limiter

```typescript
function requestSizeLimit(maxBytes: number): Middleware {
  return async (ctx, next) => {
    const contentLength = ctx.headers['content-length'];

    if (contentLength && parseInt(contentLength) > maxBytes) {
      throw new HTTPError(413, 'Request entity too large');
    }

    await next();
  };
}

app.use(requestSizeLimit(1024 * 1024)); // 1MB limit
```

### 4. Caching Middleware

```typescript
interface CacheConfig {
  ttl: number; // Time to live in seconds
}

function cache(config: CacheConfig): Middleware {
  const cacheStore = new Map<string, { data: any; expires: number }>();

  return async (ctx, next) => {
    // Only cache GET requests
    if (ctx.method !== 'GET') {
      await next();
      return;
    }

    const key = ctx.path;
    const cached = cacheStore.get(key);

    if (cached && cached.expires > Date.now()) {
      // Return cached response
      ctx.json(cached.data);
      return;
    }

    // Capture response
    const originalJson = ctx.json;
    let responseData: any;

    ctx.json = function(data: any, status?: number) {
      responseData = data;
      return originalJson.call(ctx, data, status);
    };

    await next();

    // Cache response
    if (responseData) {
      cacheStore.set(key, {
        data: responseData,
        expires: Date.now() + (config.ttl * 1000),
      });
    }
  };
}

app.get('/api/data',
  cache({ ttl: 60 }), // Cache for 60 seconds
  handler
);
```

### 5. Compression Middleware

```typescript
import zlib from 'zlib';
import { promisify } from 'util';

const gzip = promisify(zlib.gzip);

function compression(): Middleware {
  return async (ctx, next) => {
    const acceptEncoding = ctx.headers['accept-encoding'] as string || '';

    if (!acceptEncoding.includes('gzip')) {
      await next();
      return;
    }

    // Intercept response
    const originalJson = ctx.json;

    ctx.json = async function(data: any, status?: number) {
      const jsonString = JSON.stringify(data);

      // Only compress if response is large enough
      if (jsonString.length > 1024) {
        const compressed = await gzip(jsonString);
        ctx.setHeader('Content-Encoding', 'gzip');
        ctx.setHeader('Content-Type', 'application/json');
        ctx.status(status || 200);
        ctx.res.end(compressed);
      } else {
        originalJson.call(ctx, data, status);
      }
    };

    await next();
  };
}
```

### 6. Security Headers

```typescript
function securityHeaders(): Middleware {
  return async (ctx, next) => {
    // Security headers
    ctx.setHeader('X-Content-Type-Options', 'nosniff');
    ctx.setHeader('X-Frame-Options', 'DENY');
    ctx.setHeader('X-XSS-Protection', '1; mode=block');
    ctx.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    ctx.setHeader('Content-Security-Policy', "default-src 'self'");

    await next();
  };
}

app.use(securityHeaders());
```

---

## Best Practices

### 1. Always Call next()

```typescript
// BAD - Forgot to call next()
function badMiddleware(): Middleware {
  return async (ctx, next) => {
    console.log('Before');
    // Missing: await next();
    console.log('After');
  };
}

// GOOD
function goodMiddleware(): Middleware {
  return async (ctx, next) => {
    console.log('Before');
    await next();
    console.log('After');
  };
}
```

### 2. Handle Errors Properly

```typescript
function errorSafeMiddleware(): Middleware {
  return async (ctx, next) => {
    try {
      // Your middleware logic
      await next();
    } catch (error) {
      // Handle or re-throw
      console.error('Middleware error:', error);
      throw error;
    }
  };
}
```

### 3. Use Factory Pattern

```typescript
// GOOD - Factory with config
function myMiddleware(config: MyConfig): Middleware {
  return async (ctx, next) => {
    // Use config
    await next();
  };
}

// BAD - Direct middleware without config option
function myMiddleware(): Middleware {
  return async (ctx, next) => {
    await next();
  };
}
```

### 4. Keep Middleware Focused

```typescript
// GOOD - Single responsibility
function authenticateUser(): Middleware { /* ... */ }
function checkPermissions(): Middleware { /* ... */ }
function validateInput(): Middleware { /* ... */ }

// BAD - Does too much
function doEverything(): Middleware { /* ... */ }
```

### 5. Document Middleware

```typescript
/**
 * Rate limiting middleware
 *
 * @param config - Rate limit configuration
 * @param config.windowMs - Time window in milliseconds
 * @param config.maxRequests - Maximum requests per window
 *
 * @example
 * app.use(rateLimit({ windowMs: 60000, maxRequests: 100 }));
 */
function rateLimit(config: RateLimitConfig): Middleware {
  return async (ctx, next) => {
    // Implementation
    await next();
  };
}
```

### 6. Use TypeScript Types

```typescript
import type { Middleware, Context } from 'ramapi';

interface MyMiddlewareConfig {
  option: string;
}

function myMiddleware(config: MyMiddlewareConfig): Middleware {
  return async (ctx: Context, next: () => Promise<void>) => {
    await next();
  };
}
```

---

## Testing Middleware

### Unit Testing

```typescript
import { describe, it, expect, vi } from 'vitest';

describe('myMiddleware', () => {
  it('should call next()', async () => {
    const middleware = myMiddleware({ option: 'value' });

    const ctx = {
      // Mock context
      headers: {},
      state: {},
    } as any;

    const next = vi.fn();

    await middleware(ctx, next);

    expect(next).toHaveBeenCalled();
  });

  it('should add data to context', async () => {
    const middleware = addTimestamp();

    const ctx = {
      state: {},
    } as any;

    const next = vi.fn();

    await middleware(ctx, next);

    expect(ctx.state.timestamp).toBeDefined();
  });
});
```

### Integration Testing

```typescript
import { createApp } from 'ramapi';
import request from 'supertest';

describe('middleware integration', () => {
  it('should apply middleware to route', async () => {
    const app = createApp();

    app.use(myMiddleware({ option: 'value' }));

    app.get('/test', async (ctx) => {
      ctx.json({ success: true });
    });

    const response = await request(app)
      .get('/test')
      .expect(200);

    expect(response.body).toEqual({ success: true });
  });
});
```

---

## Complete Example

```typescript
import { createApp, type Middleware } from 'ramapi';

// Custom middleware
function requestLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    console.log(`→ ${ctx.method} ${ctx.path}`);

    await next();

    const duration = Date.now() - start;
    console.log(`← ${ctx.method} ${ctx.path} - ${duration}ms`);
  };
}

function addRequestId(): Middleware {
  return async (ctx, next) => {
    ctx.state.requestId = crypto.randomUUID();
    ctx.setHeader('X-Request-ID', ctx.state.requestId);
    await next();
  };
}

function timing(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    await next();
    const duration = Date.now() - start;
    ctx.setHeader('X-Response-Time', `${duration}ms`);
  };
}

// Apply middleware
const app = createApp();

app.use(requestLogger());
app.use(addRequestId());
app.use(timing());

app.get('/', async (ctx) => {
  ctx.json({
    message: 'Hello',
    requestId: ctx.state.requestId,
  });
});

app.listen(3000);
```

---

## Next Steps

- [Built-in Middleware Reference](built-in-middleware.md)
- [CORS Middleware](cors.md)
- [Rate Limiting Middleware](rate-limiting.md)
- [Middleware API Reference](../api-reference/middleware.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
