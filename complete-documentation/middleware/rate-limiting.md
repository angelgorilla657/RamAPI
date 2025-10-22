# Rate Limiting Middleware

Rate limiting protects your API from abuse by limiting the number of requests a client can make within a time window. This guide covers configuration, strategies, and best practices.

## Table of Contents

1. [Basic Usage](#basic-usage)
2. [Configuration Options](#configuration-options)
3. [Rate Limit Strategies](#rate-limit-strategies)
4. [Per-Route Rate Limits](#per-route-rate-limits)
5. [Custom Key Generators](#custom-key-generators)
6. [Redis-Based Rate Limiting](#redis-based-rate-limiting)
7. [Rate Limit Headers](#rate-limit-headers)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Basic Usage

### Simple Rate Limiting

```typescript
import { createApp, rateLimit } from 'ramapi';

const app = createApp();

// Apply rate limiting globally
app.use(rateLimit({
  windowMs: 60000, // 1 minute
  maxRequests: 100, // 100 requests per minute
}));

app.get('/', async (ctx) => {
  ctx.json({ message: 'Rate limited endpoint' });
});

app.listen(3000);
```

### Default Configuration

If no config is provided, the defaults are:
- **windowMs**: 60000 (1 minute)
- **maxRequests**: 100 (100 requests per window)
- **keyGenerator**: IP address-based

```typescript
// Uses default: 100 requests per minute per IP
app.use(rateLimit());
```

---

## Configuration Options

### RateLimitConfig Interface

```typescript
interface RateLimitConfig {
  windowMs?: number;                    // Time window in milliseconds
  maxRequests?: number;                 // Max requests per window
  keyGenerator?: (ctx: Context) => string; // Function to identify clients
  message?: string;                     // Custom error message
  skipSuccessfulRequests?: boolean;     // Don't count successful requests
  skipFailedRequests?: boolean;         // Don't count failed requests
}
```

### All Options Example

```typescript
app.use(rateLimit({
  // Time window (in milliseconds)
  windowMs: 15 * 60 * 1000, // 15 minutes

  // Maximum requests per window
  maxRequests: 100,

  // Custom key generator (default: IP address)
  keyGenerator: (ctx) => {
    // Rate limit by API key instead of IP
    return ctx.headers['x-api-key'] as string || 'anonymous';
  },

  // Custom error message
  message: 'Too many requests from this IP, please try again later',

  // Don't count successful requests (only count errors)
  skipSuccessfulRequests: false,

  // Don't count failed requests
  skipFailedRequests: false,
}));
```

---

## Rate Limit Strategies

### By IP Address (Default)

```typescript
app.use(rateLimit({
  windowMs: 60000,
  maxRequests: 100,
}));
```

### By API Key

```typescript
app.use(rateLimit({
  windowMs: 60000,
  maxRequests: 1000, // Higher limit for authenticated users
  keyGenerator: (ctx) => {
    const apiKey = ctx.headers['x-api-key'] as string;
    return apiKey || 'anonymous';
  },
}));
```

### By User ID

```typescript
import { authenticate, rateLimit } from 'ramapi';

app.use(authenticate());

app.use(rateLimit({
  windowMs: 60000,
  maxRequests: 200,
  keyGenerator: (ctx) => {
    // Rate limit by authenticated user
    return ctx.user?.id || ctx.req.socket.remoteAddress || 'unknown';
  },
}));
```

### Tiered Rate Limits

```typescript
function tierBasedRateLimit(): Middleware {
  const limits = {
    free: { windowMs: 60000, maxRequests: 10 },
    pro: { windowMs: 60000, maxRequests: 100 },
    enterprise: { windowMs: 60000, maxRequests: 1000 },
  };

  return async (ctx, next) => {
    const tier = ctx.user?.tier || 'free';
    const config = limits[tier];

    const limiter = rateLimit(config);
    await limiter(ctx, next);
  };
}

app.use(authenticate());
app.use(tierBasedRateLimit());
```

---

## Per-Route Rate Limits

### Different Limits for Different Routes

```typescript
// Strict limit for auth endpoints
app.post('/api/auth/login',
  rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    maxRequests: 5, // Only 5 login attempts
    message: 'Too many login attempts, please try again later',
  }),
  async (ctx) => {
    // Login logic
    ctx.json({ token: 'jwt-token' });
  }
);

// Normal limit for public API
app.get('/api/public',
  rateLimit({
    windowMs: 60000,
    maxRequests: 100,
  }),
  async (ctx) => {
    ctx.json({ data: 'public data' });
  }
);

// Higher limit for authenticated endpoints
app.get('/api/private',
  authenticate(),
  rateLimit({
    windowMs: 60000,
    maxRequests: 1000,
    keyGenerator: (ctx) => ctx.user?.id || 'anonymous',
  }),
  async (ctx) => {
    ctx.json({ data: 'private data' });
  }
);
```

### Route Groups with Rate Limits

```typescript
// Public routes - strict limits
app.group('/api/public', (router) => {
  router.use(rateLimit({
    windowMs: 60000,
    maxRequests: 50,
  }));

  router.get('/posts', async (ctx) => {
    ctx.json({ posts: [] });
  });
});

// Private routes - higher limits
app.group('/api/private', (router) => {
  router.use(authenticate());
  router.use(rateLimit({
    windowMs: 60000,
    maxRequests: 500,
    keyGenerator: (ctx) => ctx.user?.id,
  }));

  router.get('/profile', async (ctx) => {
    ctx.json({ user: ctx.user });
  });
});
```

---

## Custom Key Generators

### IP Address with X-Forwarded-For

```typescript
app.use(rateLimit({
  keyGenerator: (ctx) => {
    return ctx.headers['x-forwarded-for'] as string ||
           ctx.headers['x-real-ip'] as string ||
           ctx.req.socket.remoteAddress ||
           'unknown';
  },
}));
```

### Combination of IP and User Agent

```typescript
app.use(rateLimit({
  keyGenerator: (ctx) => {
    const ip = ctx.req.socket.remoteAddress || 'unknown';
    const userAgent = ctx.headers['user-agent'] || 'unknown';
    return `${ip}:${userAgent}`;
  },
}));
```

### Path-Based Keys

```typescript
app.use(rateLimit({
  keyGenerator: (ctx) => {
    // Different limits for different paths
    const ip = ctx.req.socket.remoteAddress || 'unknown';
    return `${ip}:${ctx.path}`;
  },
}));
```

---

## Redis-Based Rate Limiting

The built-in rate limiter uses in-memory storage. For production with multiple servers, use Redis:

### Install Redis Client

```bash
npm install ioredis
```

### Redis Rate Limiter

```typescript
import Redis from 'ioredis';
import type { Middleware } from 'ramapi';
import { HTTPError } from 'ramapi';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
});

interface RedisRateLimitConfig {
  windowMs: number;
  maxRequests: number;
  keyGenerator?: (ctx: any) => string;
  message?: string;
}

function redisRateLimit(config: RedisRateLimitConfig): Middleware {
  const {
    windowMs,
    maxRequests,
    keyGenerator = (ctx) => ctx.req.socket.remoteAddress || 'unknown',
    message = 'Too many requests, please try again later',
  } = config;

  return async (ctx, next) => {
    const key = `rate-limit:${keyGenerator(ctx)}`;
    const now = Date.now();
    const windowStart = now - windowMs;

    // Remove old entries and count current requests
    await redis.zremrangebyscore(key, 0, windowStart);
    const count = await redis.zcard(key);

    if (count >= maxRequests) {
      // Get reset time
      const oldestEntry = await redis.zrange(key, 0, 0, 'WITHSCORES');
      const resetTime = parseInt(oldestEntry[1]) + windowMs;

      ctx.setHeader('X-RateLimit-Limit', maxRequests.toString());
      ctx.setHeader('X-RateLimit-Remaining', '0');
      ctx.setHeader('X-RateLimit-Reset', new Date(resetTime).toISOString());
      ctx.setHeader('Retry-After', Math.ceil((resetTime - now) / 1000).toString());

      throw new HTTPError(429, message);
    }

    // Add current request
    await redis.zadd(key, now, `${now}`);
    await redis.expire(key, Math.ceil(windowMs / 1000));

    // Set rate limit headers
    ctx.setHeader('X-RateLimit-Limit', maxRequests.toString());
    ctx.setHeader('X-RateLimit-Remaining', (maxRequests - count - 1).toString());
    ctx.setHeader('X-RateLimit-Reset', new Date(now + windowMs).toISOString());

    await next();
  };
}

// Usage
app.use(redisRateLimit({
  windowMs: 60000,
  maxRequests: 100,
}));
```

---

## Rate Limit Headers

The rate limit middleware automatically adds headers to responses:

### Standard Headers

- **X-RateLimit-Limit**: Maximum requests allowed in window
- **X-RateLimit-Remaining**: Remaining requests in current window
- **X-RateLimit-Reset**: ISO timestamp when the window resets
- **Retry-After**: (when limited) Seconds until next request is allowed

### Example Response

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 2025-01-15T10:35:00.000Z
```

### When Rate Limited

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 2025-01-15T10:35:00.000Z
Retry-After: 45
Content-Type: application/json

{
  "error": true,
  "message": "Too many requests, please try again later"
}
```

---

## Best Practices

### 1. Use Different Limits for Different Endpoints

```typescript
// Strict limits for sensitive operations
app.post('/api/auth/login', rateLimit({ maxRequests: 5 }), loginHandler);
app.post('/api/auth/register', rateLimit({ maxRequests: 3 }), registerHandler);

// Normal limits for general API
app.get('/api/data', rateLimit({ maxRequests: 100 }), dataHandler);
```

### 2. Higher Limits for Authenticated Users

```typescript
app.use(async (ctx, next) => {
  const isAuthenticated = ctx.headers.authorization;

  if (isAuthenticated) {
    // Higher limit for authenticated users
    await rateLimit({ maxRequests: 1000 })(ctx, next);
  } else {
    // Lower limit for anonymous users
    await rateLimit({ maxRequests: 100 })(ctx, next);
  }
});
```

### 3. Use Redis in Production

The in-memory limiter doesn't work across multiple servers:

```typescript
// Development: in-memory
const limiter = process.env.NODE_ENV === 'production'
  ? redisRateLimit({ windowMs: 60000, maxRequests: 100 })
  : rateLimit({ windowMs: 60000, maxRequests: 100 });

app.use(limiter);
```

### 4. Skip Internal Requests

```typescript
app.use(rateLimit({
  windowMs: 60000,
  maxRequests: 100,
  keyGenerator: (ctx) => {
    const ip = ctx.req.socket.remoteAddress;

    // Don't rate limit internal requests
    if (ip === '127.0.0.1' || ip === '::1') {
      return 'internal';
    }

    return ip || 'unknown';
  },
}));
```

### 5. Provide Clear Error Messages

```typescript
app.post('/api/auth/login',
  rateLimit({
    maxRequests: 5,
    windowMs: 15 * 60 * 1000,
    message: 'Too many login attempts. Please try again in 15 minutes.',
  }),
  loginHandler
);
```

### 6. Monitor Rate Limit Violations

```typescript
function monitoredRateLimit(config: RateLimitConfig): Middleware {
  const limiter = rateLimit(config);

  return async (ctx, next) => {
    try {
      await limiter(ctx, next);
    } catch (error) {
      if (error instanceof HTTPError && error.statusCode === 429) {
        // Log rate limit violation
        console.warn('Rate limit exceeded:', {
          ip: ctx.req.socket.remoteAddress,
          path: ctx.path,
          method: ctx.method,
        });
      }
      throw error;
    }
  };
}
```

---

## Troubleshooting

### Problem: Rate limits not working across servers

**Cause:** Using in-memory storage with multiple server instances.

**Solution:** Use Redis-based rate limiting for multi-server deployments.

### Problem: Rate limits too strict

**Cause:** windowMs or maxRequests too low.

**Solution:** Adjust configuration:

```typescript
app.use(rateLimit({
  windowMs: 60000, // Increase window
  maxRequests: 200, // Increase max requests
}));
```

### Problem: Users behind same IP getting rate limited together

**Cause:** Using IP-based rate limiting with users behind NAT/proxy.

**Solution:** Use user-based or API key-based rate limiting:

```typescript
app.use(rateLimit({
  keyGenerator: (ctx) => {
    // Use API key if available, fall back to IP
    return ctx.headers['x-api-key'] as string ||
           ctx.req.socket.remoteAddress ||
           'unknown';
  },
}));
```

### Problem: Rate limit resets unexpectedly

**Cause:** Server restart clears in-memory storage.

**Solution:** Use persistent storage (Redis).

---

## Complete Examples

### Simple API Protection

```typescript
import { createApp, rateLimit, logger } from 'ramapi';

const app = createApp();

app.use(logger());
app.use(rateLimit({
  windowMs: 60000, // 1 minute
  maxRequests: 100, // 100 requests per minute
}));

app.get('/api/data', async (ctx) => {
  ctx.json({ data: 'Protected by rate limiting' });
});

app.listen(3000);
```

### Production Setup with Redis

```typescript
import { createApp, logger } from 'ramapi';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const app = createApp();

app.use(logger());
app.use(redisRateLimit({
  windowMs: 60000,
  maxRequests: 100,
}));

app.listen(3000);
```

### Multi-Tier Rate Limiting

```typescript
// Strict limits for auth
app.post('/api/auth/login',
  rateLimit({
    windowMs: 15 * 60 * 1000,
    maxRequests: 5,
    message: 'Too many login attempts',
  }),
  loginHandler
);

// Normal limits for public API
app.use('/api/public', rateLimit({
  windowMs: 60000,
  maxRequests: 50,
}));

// Higher limits for authenticated API
app.use('/api/private',
  authenticate(),
  rateLimit({
    windowMs: 60000,
    maxRequests: 500,
    keyGenerator: (ctx) => ctx.user?.id,
  })
);
```

---

## Next Steps

- [CORS Middleware](cors.md)
- [Logger Middleware](logger.md)
- [Custom Middleware Guide](custom-middleware.md)
- [Security Best Practices](../guides/security.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
