# CORS Middleware

Cross-Origin Resource Sharing (CORS) middleware enables your API to handle cross-origin requests from browsers. This guide covers CORS configuration, security considerations, and common patterns.

## Table of Contents

1. [Basic Usage](#basic-usage)
2. [Configuration Options](#configuration-options)
3. [Origin Validation](#origin-validation)
4. [Credentials and Authentication](#credentials-and-authentication)
5. [Common Patterns](#common-patterns)
6. [Security Best Practices](#security-best-practices)
7. [Troubleshooting](#troubleshooting)

---

## Basic Usage

### Allow All Origins (Development)

```typescript
import { createApp, cors } from 'ramapi';

const app = createApp();

// Allow all origins (use only in development!)
app.use(cors());

app.get('/', async (ctx) => {
  ctx.json({ message: 'CORS enabled' });
});

app.listen(3000);
```

### Allow Specific Origins (Production)

```typescript
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  credentials: true,
}));
```

---

## Configuration Options

### CorsConfig Interface

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

### All Options Example

```typescript
app.use(cors({
  // Allowed origins
  origin: ['https://example.com', 'https://app.example.com'],

  // Allowed HTTP methods
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],

  // Allowed request headers
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],

  // Exposed response headers (accessible to client)
  exposedHeaders: ['X-Request-ID', 'X-RateLimit-Remaining'],

  // Allow credentials (cookies, authorization headers)
  credentials: true,

  // Preflight cache duration (in seconds)
  maxAge: 86400, // 24 hours
}));
```

---

## Origin Validation

### String Origin

Allow a single origin:

```typescript
app.use(cors({
  origin: 'https://example.com'
}));
```

### Array of Origins

Allow multiple specific origins:

```typescript
app.use(cors({
  origin: [
    'https://example.com',
    'https://app.example.com',
    'https://admin.example.com',
  ]
}));
```

### Wildcard (All Origins)

Allow all origins (default):

```typescript
app.use(cors({
  origin: '*'
}));

// Or simply:
app.use(cors());
```

### Dynamic Origin Validation

Use a function to validate origins dynamically:

```typescript
app.use(cors({
  origin: (requestOrigin) => {
    // Allow localhost in development
    if (process.env.NODE_ENV === 'development') {
      return requestOrigin.includes('localhost') ||
             requestOrigin.includes('127.0.0.1');
    }

    // Production whitelist
    const allowedOrigins = [
      'https://example.com',
      'https://app.example.com',
    ];

    return allowedOrigins.includes(requestOrigin);
  },
  credentials: true,
}));
```

### Environment-Based Origins

```typescript
const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || ['*'];

app.use(cors({
  origin: allowedOrigins,
  credentials: true,
}));
```

---

## Credentials and Authentication

### Enable Credentials

Required for cookies and authentication headers:

```typescript
app.use(cors({
  origin: ['https://example.com'],
  credentials: true,
}));
```

**Important:** When `credentials: true`, you **cannot** use `origin: '*'`. You must specify exact origins.

### Handling Authentication

```typescript
import { authenticate, cors } from 'ramapi';

// Apply CORS before authentication
app.use(cors({
  origin: ['https://app.example.com'],
  credentials: true,
  allowedHeaders: ['Content-Type', 'Authorization'],
}));

// Protected route with authentication
app.get('/api/user', authenticate(), async (ctx) => {
  ctx.json({ user: ctx.user });
});
```

---

## Common Patterns

### Global CORS

Apply CORS to all routes:

```typescript
const app = createApp();

app.use(cors({
  origin: ['https://example.com'],
  credentials: true,
}));
```

### Per-Route CORS

Apply different CORS settings to specific routes:

```typescript
// Public API - allow all origins
app.get('/api/public', cors(), async (ctx) => {
  ctx.json({ message: 'Public data' });
});

// Private API - restrict origins
app.get('/api/private',
  cors({
    origin: ['https://app.example.com'],
    credentials: true,
  }),
  authenticate(),
  async (ctx) => {
    ctx.json({ data: 'Private data' });
  }
);
```

### Route Groups with CORS

```typescript
// Public routes - allow all
app.group('/api/public', (router) => {
  router.use(cors());

  router.get('/posts', async (ctx) => {
    ctx.json({ posts: [] });
  });
});

// Private routes - restrict origins
app.group('/api/private', (router) => {
  router.use(cors({
    origin: ['https://app.example.com'],
    credentials: true,
  }));

  router.use(authenticate());

  router.get('/profile', async (ctx) => {
    ctx.json({ user: ctx.user });
  });
});
```

### Server-Level CORS

Configure CORS in server config:

```typescript
const app = createApp({
  cors: {
    origin: ['https://example.com'],
    credentials: true,
  }
});
```

---

## Security Best Practices

### 1. Never Use Wildcards with Credentials

```typescript
// BAD - Security risk!
app.use(cors({
  origin: '*',
  credentials: true, // This won't work and is insecure
}));

// GOOD - Specify exact origins
app.use(cors({
  origin: ['https://example.com'],
  credentials: true,
}));
```

### 2. Validate Origins Strictly

```typescript
// BAD - Too permissive
app.use(cors({
  origin: (requestOrigin) => requestOrigin.includes('example.com')
}));

// GOOD - Exact match
app.use(cors({
  origin: (requestOrigin) => {
    const allowedOrigins = [
      'https://example.com',
      'https://app.example.com',
    ];
    return allowedOrigins.includes(requestOrigin);
  }
}));
```

### 3. Limit Exposed Headers

Only expose headers that are necessary:

```typescript
app.use(cors({
  origin: ['https://example.com'],
  exposedHeaders: ['X-Request-ID'], // Only expose specific headers
}));
```

### 4. Restrict HTTP Methods

Only allow methods your API actually uses:

```typescript
app.use(cors({
  origin: ['https://example.com'],
  methods: ['GET', 'POST'], // Only allow GET and POST
}));
```

### 5. Set Reasonable maxAge

Cache preflight requests to reduce overhead:

```typescript
app.use(cors({
  origin: ['https://example.com'],
  maxAge: 86400, // Cache for 24 hours
}));
```

### 6. Environment-Specific Configuration

```typescript
const corsConfig = {
  development: {
    origin: '*',
    credentials: false,
  },
  production: {
    origin: process.env.ALLOWED_ORIGINS?.split(',') || [],
    credentials: true,
    maxAge: 86400,
  },
};

const env = process.env.NODE_ENV || 'development';
app.use(cors(corsConfig[env]));
```

---

## Troubleshooting

### Problem: "No 'Access-Control-Allow-Origin' header"

**Cause:** CORS middleware not applied or origin not allowed.

**Solution:**

```typescript
// Ensure CORS middleware is applied
app.use(cors({
  origin: ['https://your-frontend.com']
}));
```

### Problem: "Credentials mode requires exact origin"

**Cause:** Using `origin: '*'` with `credentials: true`.

**Solution:**

```typescript
// Specify exact origins
app.use(cors({
  origin: ['https://example.com'],
  credentials: true,
}));
```

### Problem: "Request header not allowed"

**Cause:** Custom header not in `allowedHeaders`.

**Solution:**

```typescript
app.use(cors({
  origin: ['https://example.com'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Custom-Header'],
}));
```

### Problem: "Response header not accessible"

**Cause:** Custom response header not in `exposedHeaders`.

**Solution:**

```typescript
app.use(cors({
  origin: ['https://example.com'],
  exposedHeaders: ['X-Request-ID', 'X-Custom-Header'],
}));
```

### Problem: Preflight request fails

**Cause:** OPTIONS method not handled.

**Solution:** CORS middleware automatically handles OPTIONS requests. Ensure it's applied globally:

```typescript
app.use(cors()); // Must be before routes
```

### Debugging CORS

```typescript
app.use(cors({
  origin: (requestOrigin) => {
    console.log('Request from origin:', requestOrigin);
    const allowed = allowedOrigins.includes(requestOrigin);
    console.log('Allowed:', allowed);
    return allowed;
  }
}));
```

---

## Complete Examples

### Development Setup

```typescript
import { createApp, cors, logger } from 'ramapi';

const app = createApp();

app.use(logger());
app.use(cors()); // Allow all origins in development

app.get('/api/data', async (ctx) => {
  ctx.json({ data: 'Available to all origins' });
});

app.listen(3000);
```

### Production Setup

```typescript
import { createApp, cors, logger, authenticate } from 'ramapi';

const app = createApp();

app.use(logger());

// Strict CORS for production
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['https://example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Request-ID'],
  credentials: true,
  maxAge: 86400,
}));

// Public endpoint
app.get('/api/public', async (ctx) => {
  ctx.json({ message: 'Public data' });
});

// Protected endpoint
app.get('/api/private', authenticate(), async (ctx) => {
  ctx.json({ user: ctx.user });
});

app.listen(3000);
```

### Multiple CORS Policies

```typescript
// Separate CORS configs for different route groups
const publicCors = cors({
  origin: '*',
});

const privateCors = cors({
  origin: ['https://app.example.com'],
  credentials: true,
});

// Public API
app.group('/api/public', (router) => {
  router.use(publicCors);
  router.get('/posts', async (ctx) => {
    ctx.json({ posts: [] });
  });
});

// Private API
app.group('/api/private', (router) => {
  router.use(privateCors);
  router.use(authenticate());
  router.get('/profile', async (ctx) => {
    ctx.json({ user: ctx.user });
  });
});
```

---

## Next Steps

- [Logger Middleware](logger.md)
- [Rate Limiting Middleware](rate-limiting.md)
- [Custom Middleware Guide](custom-middleware.md)
- [Authentication Guide](../guides/authentication.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
