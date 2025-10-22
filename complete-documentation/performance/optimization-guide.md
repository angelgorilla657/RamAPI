# Optimization Guide

Practical techniques and strategies to optimize your RamAPI application for maximum performance.

## Table of Contents

1. [Quick Wins](#quick-wins)
2. [Profiling & Monitoring](#profiling--monitoring)
3. [Middleware Optimization](#middleware-optimization)
4. [Database Optimization](#database-optimization)
5. [Caching Strategies](#caching-strategies)
6. [Memory Optimization](#memory-optimization)
7. [Real-World Examples](#real-world-examples)

---

## Quick Wins

### 1. Enable uWebSockets

**Impact:** 2-3x performance improvement
**Effort:** 5 minutes
**Risk:** Low

```typescript
// Before: 124K req/s
const app = createApp({
  adapter: { type: 'node-http' },
});

// After: 350K req/s
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// Or let RamAPI choose automatically
const app = createApp(); // Tries uWebSockets first
```

**Installation:**
```bash
npm install uWebSockets.js
```

---

### 2. Minimize Middleware

**Impact:** 15-20% improvement per middleware removed
**Effort:** 10 minutes
**Risk:** Low

```typescript
// Bad: Many middleware (slower)
app.use(helmet());
app.use(compress());
app.use(bodyParser());
app.use(cookieParser());
app.use(sessionMiddleware());
app.use(corsMiddleware());
app.use(rateLimiter());
app.use(requestLogger());
app.use(responseLogger());
app.use(errorLogger());
// Result: 10 middleware = 50-80% overhead

// Good: Only essential middleware (faster)
app.use(authenticate);
app.use(logger());
// Result: 2 middleware = 15-20% overhead
```

**Rule of thumb:** Each middleware adds ~5-8% latency

---

### 3. Use Validation Efficiently

**Impact:** 10-15% improvement
**Effort:** 5 minutes
**Risk:** None

```typescript
// Bad: Validate everything (slow)
validate({
  body: z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    age: z.number().int().min(0).max(150),
    address: z.object({
      street: z.string(),
      city: z.string(),
      zipCode: z.string(),
      country: z.string(),
    }),
    preferences: z.object({
      newsletter: z.boolean(),
      notifications: z.boolean(),
      theme: z.enum(['light', 'dark']),
    }),
    metadata: z.record(z.string()),
  }),
  query: z.object({
    filter: z.string().optional(),
    sort: z.string().optional(),
    page: z.number().optional(),
  }),
  params: z.object({
    id: z.string().regex(/^\d+$/),
  }),
})

// Good: Validate only what's needed (fast)
validate({
  body: z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    age: z.number().int().positive(),
  }),
  params: z.object({
    id: z.string().regex(/^\d+$/),
  }),
})
```

**Tip:** Validate at the edge, trust internal services

---

### 4. Cache Expensive Operations

**Impact:** 10-100x improvement (depending on operation)
**Effort:** 15 minutes
**Risk:** Medium (cache invalidation)

```typescript
// Bad: Query on every request (slow)
app.get('/api/config', async (ctx) => {
  const config = await db.query('SELECT * FROM config');
  ctx.json(config);
});
// Result: 1000 req/s (database bottleneck)

// Good: Cache static data (fast)
let configCache: any = null;
let cacheTime = 0;
const CACHE_TTL = 60000; // 1 minute

app.get('/api/config', async (ctx) => {
  const now = Date.now();
  if (!configCache || now - cacheTime > CACHE_TTL) {
    configCache = await db.query('SELECT * FROM config');
    cacheTime = now;
  }
  ctx.json(configCache);
});
// Result: 50,000+ req/s (memory cache)
```

---

### 5. Enable Production Mode

**Impact:** 5-10% improvement
**Effort:** 1 minute
**Risk:** None

```bash
# Development mode (slow)
NODE_ENV=development npm start

# Production mode (fast)
NODE_ENV=production npm start
```

**What changes in production:**
- No stack traces in errors
- Disabled source maps
- Optimized logging
- Disabled debug output

---

## Profiling & Monitoring

### Built-in Profiling

**Enable profiling to find bottlenecks:**

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 100, // Flag requests >100ms
      autoDetectBottlenecks: true,
      sampleRate: 0.1, // Profile 10% of requests
    },
  },
});
```

**View profiling data:**

```typescript
import { getProfiles } from 'ramapi';

app.get('/debug/slow', async (ctx) => {
  const profiles = await getProfiles({ slowOnly: true });
  ctx.json({ profiles });
});
```

**Example output:**

```json
{
  "profiles": [
    {
      "operationName": "GET /api/users/:id",
      "duration": 245.6,
      "breakdown": {
        "authenticate": 12.3,
        "validate": 8.7,
        "database": 220.1,
        "handler": 4.5
      },
      "bottleneck": "database"
    }
  ]
}
```

---

### Performance Metrics

**Track key metrics:**

```typescript
import { getMetrics } from 'ramapi';

const app = createApp({
  observability: {
    metrics: { enabled: true },
  },
});

app.get('/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.json({
    requestsPerSecond: metrics.requestsPerSecond,
    p50Latency: metrics.p50Latency,
    p95Latency: metrics.p95Latency,
    p99Latency: metrics.p99Latency,
    errorRate: metrics.errorRate,
  });
});
```

**Set up alerts:**

```typescript
setInterval(() => {
  const metrics = getMetrics();

  // Alert on high latency
  if (metrics.p95Latency > 100) {
    console.error('⚠️  High latency detected:', metrics.p95Latency);
  }

  // Alert on high error rate
  if (metrics.errorRate > 0.01) {
    console.error('⚠️  High error rate:', metrics.errorRate);
  }
}, 60000); // Check every minute
```

---

### Node.js Profiling

**CPU profiling:**

```bash
# Generate CPU profile
node --prof dist/index.js

# Process profile
node --prof-process isolate-0x*.log > profile.txt
```

**Memory profiling:**

```bash
# Generate heap snapshot
node --inspect dist/index.js

# Open Chrome DevTools
# chrome://inspect
# Take heap snapshot
```

**Flame graphs:**

```bash
# Install clinic.js
npm install -g clinic

# Run flame graph
clinic flame -- node dist/index.js

# Open results
open .clinic/*.html
```

---

## Middleware Optimization

### Conditional Middleware

**Apply middleware only where needed:**

```typescript
// Bad: Apply to all routes (slow)
app.use(authenticate);
app.get('/public', handler);      // Doesn't need auth!
app.get('/api/users', handler);   // Needs auth

// Good: Apply selectively (fast)
app.get('/public', handler);      // No auth overhead
app.get('/api/users', authenticate, handler); // Auth only where needed
```

---

### Fast Path for Simple Routes

**Skip middleware for high-traffic routes:**

```typescript
// Health check doesn't need middleware
app.get('/health', async (ctx) => {
  ctx.json({ status: 'ok' });
});
// Result: 350K req/s (no middleware)

// API routes use middleware
app.get('/api/users', authenticate, logger(), async (ctx) => {
  ctx.json({ users: [] });
});
// Result: 285K req/s (with middleware)
```

---

### Async Middleware Optimization

**Avoid unnecessary awaits:**

```typescript
// Bad: Awaiting everything (slow)
app.use(async (ctx, next) => {
  const start = await Promise.resolve(Date.now());
  await next();
  const duration = await Promise.resolve(Date.now() - start);
  console.log('Duration:', duration);
});

// Good: Only await what's needed (fast)
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const duration = Date.now() - start;
  console.log('Duration:', duration);
});
```

---

## Database Optimization

### Connection Pooling

**Use connection pools for better performance:**

```typescript
import { Pool } from 'pg';

// Bad: New connection per request (slow)
app.get('/api/users', async (ctx) => {
  const client = new Client({ connectionString: process.env.DATABASE_URL });
  await client.connect();
  const result = await client.query('SELECT * FROM users');
  await client.end();
  ctx.json({ users: result.rows });
});
// Result: 500 req/s (connection overhead)

// Good: Connection pool (fast)
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

app.get('/api/users', async (ctx) => {
  const result = await pool.query('SELECT * FROM users');
  ctx.json({ users: result.rows });
});
// Result: 5,000+ req/s (reused connections)
```

---

### Query Optimization

**Optimize database queries:**

```typescript
// Bad: N+1 query problem (very slow)
app.get('/api/users', async (ctx) => {
  const users = await db.query('SELECT * FROM users');

  for (const user of users) {
    user.posts = await db.query('SELECT * FROM posts WHERE user_id = ?', [user.id]);
  }

  ctx.json({ users });
});
// Result: 10 users = 11 queries = 100ms latency

// Good: Single query with JOIN (fast)
app.get('/api/users', async (ctx) => {
  const users = await db.query(`
    SELECT u.*, p.id as post_id, p.title, p.content
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
  `);

  // Group results
  const grouped = groupByUser(users);

  ctx.json({ users: grouped });
});
// Result: 1 query = 10ms latency
```

---

### Indexes

**Add indexes for frequently queried columns:**

```sql
-- Bad: Full table scan (slow)
SELECT * FROM users WHERE email = 'user@example.com';
-- Result: 500ms for 1M users

-- Good: Index lookup (fast)
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'user@example.com';
-- Result: 1ms for 1M users
```

---

### Prepared Statements

**Use prepared statements for repeated queries:**

```typescript
// Bad: Query parsing on every request (slow)
app.get('/api/users/:id', async (ctx) => {
  const result = await db.query(
    'SELECT * FROM users WHERE id = ' + ctx.params.id // SQL injection risk!
  );
  ctx.json({ user: result.rows[0] });
});

// Good: Prepared statement (fast + safe)
const getUserStmt = db.prepare('SELECT * FROM users WHERE id = ?');

app.get('/api/users/:id', async (ctx) => {
  const user = getUserStmt.get(ctx.params.id);
  ctx.json({ user });
});
```

---

## Caching Strategies

### In-Memory Cache

**Simple in-memory cache:**

```typescript
class SimpleCache<T> {
  private cache = new Map<string, { value: T; expires: number }>();

  set(key: string, value: T, ttl: number): void {
    this.cache.set(key, {
      value,
      expires: Date.now() + ttl,
    });
  }

  get(key: string): T | undefined {
    const item = this.cache.get(key);
    if (!item) return undefined;

    if (Date.now() > item.expires) {
      this.cache.delete(key);
      return undefined;
    }

    return item.value;
  }

  clear(): void {
    this.cache.clear();
  }
}

const cache = new SimpleCache<any>();

app.get('/api/expensive', async (ctx) => {
  const cacheKey = 'expensive-operation';
  const cached = cache.get(cacheKey);

  if (cached) {
    ctx.json(cached);
    return;
  }

  const result = await expensiveOperation();
  cache.set(cacheKey, result, 60000); // 1 minute TTL

  ctx.json(result);
});
```

---

### Redis Cache

**Distributed cache with Redis:**

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

app.get('/api/users/:id', async (ctx) => {
  const cacheKey = `user:${ctx.params.id}`;

  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    ctx.json(JSON.parse(cached));
    return;
  }

  // Query database
  const user = await db.query('SELECT * FROM users WHERE id = ?', [ctx.params.id]);

  // Store in cache
  await redis.setex(cacheKey, 300, JSON.stringify(user)); // 5 minutes

  ctx.json(user);
});
```

---

### Cache Invalidation

**Invalidate cache on updates:**

```typescript
// Update user
app.put('/api/users/:id', async (ctx) => {
  const user = await db.query(
    'UPDATE users SET name = ? WHERE id = ?',
    [ctx.body.name, ctx.params.id]
  );

  // Invalidate cache
  await redis.del(`user:${ctx.params.id}`);

  ctx.json({ user });
});
```

---

### HTTP Caching

**Use HTTP cache headers:**

```typescript
// Cache static responses
app.get('/api/config', async (ctx) => {
  const config = await getConfig();

  // Cache for 5 minutes
  ctx.setHeader('Cache-Control', 'public, max-age=300');
  ctx.setHeader('ETag', hashConfig(config));

  ctx.json(config);
});

// Conditional requests
app.get('/api/users', async (ctx) => {
  const etag = await getUsersETag();

  if (ctx.headers['if-none-match'] === etag) {
    ctx.status(304); // Not Modified
    return;
  }

  const users = await getUsers();
  ctx.setHeader('ETag', etag);
  ctx.json({ users });
});
```

---

## Memory Optimization

### Avoid Memory Leaks

**Common memory leak patterns:**

```typescript
// Bad: Global array keeps growing (leak)
const requestLog: any[] = [];

app.use(async (ctx, next) => {
  requestLog.push({ url: ctx.url, timestamp: Date.now() });
  await next();
});
// Result: Memory grows forever

// Good: Limited size queue (no leak)
class CircularBuffer<T> {
  private buffer: T[] = [];
  constructor(private maxSize: number) {}

  push(item: T): void {
    this.buffer.push(item);
    if (this.buffer.length > this.maxSize) {
      this.buffer.shift();
    }
  }
}

const requestLog = new CircularBuffer<any>(1000); // Keep last 1000

app.use(async (ctx, next) => {
  requestLog.push({ url: ctx.url, timestamp: Date.now() });
  await next();
});
```

---

### Stream Large Responses

**Stream instead of buffering:**

```typescript
// Bad: Load entire file in memory (high memory)
app.get('/download/:file', async (ctx) => {
  const content = await fs.readFile(`/files/${ctx.params.file}`);
  ctx.send(content);
});
// Result: 100 concurrent downloads × 10MB = 1GB memory

// Good: Stream file (low memory)
import { createReadStream } from 'fs';

app.get('/download/:file', async (ctx) => {
  const stream = createReadStream(`/files/${ctx.params.file}`);
  ctx.setHeader('Content-Type', 'application/octet-stream');
  ctx.res.pipe(stream);
});
// Result: 100 concurrent downloads × 64KB buffer = 6.4MB memory
```

---

### Garbage Collection Tuning

**Tune V8 garbage collector:**

```bash
# Increase heap size
node --max-old-space-size=4096 dist/index.js

# More frequent GC (lower memory, slightly slower)
node --max-old-space-size=2048 --gc-interval=100 dist/index.js

# Less frequent GC (higher memory, slightly faster)
node --max-old-space-size=8192 --gc-interval=1000 dist/index.js
```

---

## Real-World Examples

### High-Traffic API

**Optimized for 100K+ req/s:**

```typescript
import { createApp } from 'ramapi';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const app = createApp({
  // Use uWebSockets for max performance
  adapter: { type: 'uwebsockets' },

  // Minimal observability in production
  observability: {
    metrics: { enabled: true },
    logging: { level: 'error' },
  },
});

// Cache layer
const cache = {
  async get(key: string): Promise<any> {
    const cached = await redis.get(key);
    return cached ? JSON.parse(cached) : null;
  },
  async set(key: string, value: any, ttl: number): Promise<void> {
    await redis.setex(key, ttl, JSON.stringify(value));
  },
};

// Minimal middleware
const authenticate = async (ctx: Context, next: () => Promise<void>) => {
  const token = ctx.headers.authorization;
  if (!token) {
    ctx.json({ error: 'Unauthorized' }, 401);
    return;
  }
  ctx.state.userId = await verifyToken(token);
  await next();
};

// Optimized endpoints
app.get('/api/users/:id', authenticate, async (ctx) => {
  const cacheKey = `user:${ctx.params.id}`;

  // Check cache first
  const cached = await cache.get(cacheKey);
  if (cached) {
    ctx.json(cached);
    return;
  }

  // Query database
  const user = await db.prepare('SELECT * FROM users WHERE id = ?').get(ctx.params.id);

  if (!user) {
    ctx.json({ error: 'Not found' }, 404);
    return;
  }

  // Cache for 5 minutes
  await cache.set(cacheKey, user, 300);

  ctx.json({ user });
});

await app.listen(3000);
```

**Result:** 85,000+ req/s with database, 300,000+ req/s with cache hits

---

### Cost-Optimized Microservice

**Minimize server costs:**

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// Aggressive caching
const configCache = await loadConfig();
const dataCache = new Map<string, any>();

// Health check (no overhead)
app.get('/health', async (ctx) => {
  ctx.json({ status: 'ok' });
});

// Cached config endpoint
app.get('/api/config', async (ctx) => {
  ctx.setHeader('Cache-Control', 'public, max-age=3600');
  ctx.json(configCache);
});

// Efficient data endpoint
app.get('/api/data/:key', async (ctx) => {
  const cached = dataCache.get(ctx.params.key);

  if (cached) {
    ctx.json(cached);
    return;
  }

  const data = await fetchData(ctx.params.key);
  dataCache.set(ctx.params.key, data);

  ctx.json(data);
});

await app.listen(3000);
```

**Result:** Handle 5x more traffic with same server, reduce costs by 80%

---

## Performance Checklist

- [ ] Use uWebSockets adapter
- [ ] Minimize middleware (≤ 3 per route)
- [ ] Cache expensive operations
- [ ] Use connection pooling
- [ ] Add database indexes
- [ ] Enable production mode
- [ ] Use prepared statements
- [ ] Implement HTTP caching
- [ ] Profile slow endpoints
- [ ] Monitor key metrics
- [ ] Stream large responses
- [ ] Validate efficiently
- [ ] Avoid N+1 queries
- [ ] Tune garbage collection
- [ ] Use Redis for distributed cache

---

## Next Steps

- [Performance Overview](overview.md)
- [HTTP Adapters](http-adapters.md)
- [Benchmarks](benchmarks.md)

---

**Need help optimizing?** Check the [Profiling Guide](../observability/profiling.md) or [GitHub Discussions](https://github.com/yourusername/ramapi/discussions).
