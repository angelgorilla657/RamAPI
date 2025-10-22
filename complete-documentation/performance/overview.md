# Performance Overview

RamAPI is built for speed, delivering 350K+ requests/second with uWebSockets.js and 124K+ req/s with Node.js HTTP. Learn about performance features and optimization strategies.

## Table of Contents

1. [Performance Features](#performance-features)
2. [Benchmark Results](#benchmark-results)
3. [Architecture Optimizations](#architecture-optimizations)
4. [When to Optimize](#when-to-optimize)

---

## Performance Features

### 1. Multiple HTTP Adapters

Choose your performance level:

```typescript
// Node.js HTTP: 124K req/s, stable, production-ready
const app = createApp({
  adapter: { type: 'node-http' },
});

// uWebSockets.js: 350K+ req/s, 2-3x faster
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// Auto-select: Tries uWebSockets, falls back to Node.js
const app = createApp(); // Automatic selection
```

**See:** [HTTP Adapters Guide](http-adapters.md)

### 2. Pre-Compiled Middleware

Middleware is compiled into optimized chains at registration time:

```typescript
// Traditional approach: Execute chain on every request
middleware1 → middleware2 → middleware3 → handler

// RamAPI: Pre-compiled into single function
compiledChain(ctx) // Zero overhead
```

**Result:** ~15% faster middleware execution

### 3. Optimized Routing

- **O(1) static route lookup**: Hash map for exact matches
- **Pre-compiled patterns**: Regex compiled once at startup
- **Route caching**: Last-used route cached for hot paths
- **Separate storage**: Static vs dynamic routes

```typescript
// Static route: O(1) lookup
app.get('/users', handler); // ~0.001ms lookup

// Dynamic route: Pre-compiled pattern
app.get('/users/:id', handler); // ~0.005ms lookup
```

### 4. Zero-Copy Body Parsing

Body parsing optimized for performance:

```typescript
// Adapter mode: Body parsed once
const body = await adapter.parseBody(raw); // Single parse

// Traditional: Multiple parses avoided
```

---

## Benchmark Results

### Simple JSON Response

```
Framework            Requests/sec    Latency (avg)
────────────────────────────────────────────────
RamAPI (uWebSockets) 350,000        0.28ms
RamAPI (Node.js)     124,000        0.80ms
Fastify              112,000        0.89ms
Express              45,000         2.22ms
```

### With Middleware Stack

```
Framework            Requests/sec    Latency (avg)
────────────────────────────────────────────────
RamAPI (uWebSockets) 285,000        0.35ms
RamAPI (Node.js)     98,000         1.02ms
Fastify              89,000         1.12ms
Express              38,000         2.63ms
```

### Real-World Application

```
Scenario: Auth + Validation + DB Query + Response

Framework            Requests/sec    P95 Latency
────────────────────────────────────────────────
RamAPI (uWebSockets) 85,000         2.8ms
RamAPI (Node.js)     72,000         3.2ms
Fastify              65,000         3.8ms
Express              42,000         5.5ms
```

**See:** [Detailed Benchmarks](benchmarks.md)

---

## Architecture Optimizations

### 1. Adapter Pattern

Abstracts HTTP server implementation:

```typescript
// Single codebase, multiple backends
ServerAdapter
  ↓
[Node.js HTTP] or [uWebSockets.js]
```

Benefits:
- Switch adapters without code changes
- Optimize per-environment
- Future-proof architecture

### 2. Context Reuse

Context objects optimized for reuse:

```typescript
// Lightweight context creation
const ctx = {
  req, res,
  method, url, path,
  query: {}, params: {}, // Lazy-parsed
  state: {}, // Reused object
};
```

### 3. Smart Body Parsing

Only parse when needed:

```typescript
// GET request: No body parsing
app.get('/users', handler); // Skip parsing

// POST request: Parse once
app.post('/users', handler); // Parse only if needed
```

### 4. Route Compilation

Routes compiled at registration:

```typescript
// At registration time (once)
app.get('/users/:id', handler);
// ↓ Compiled to:
const route = {
  pattern: /^\/users\/([^\/]+)$/,
  paramNames: ['id'],
  handler: compiledMiddlewareChain,
};

// At request time (fast)
const match = staticMap.get(path) || pattern.exec(path);
```

---

## When to Optimize

### Start with Defaults

```typescript
// Default setup is already fast
const app = createApp();
app.get('/api/users', getUsers);
app.listen(3000);

// Handles 124K+ req/s out of the box
```

### Optimize When:

1. **High Traffic**: >50K req/s
2. **Latency Sensitive**: P95 <10ms required
3. **Cost Optimization**: Reduce server count
4. **Specific Bottleneck**: Profiling shows clear issue

### Don't Optimize When:

1. **Premature**: No performance problem yet
2. **Wrong Layer**: Database is the bottleneck
3. **Diminishing Returns**: Already fast enough
4. **Complexity Cost**: Optimization hurts maintainability

---

## Performance Comparison

### Node.js HTTP vs uWebSockets

| Metric | Node.js HTTP | uWebSockets.js |
|--------|--------------|----------------|
| **Req/s** | 124,000 | 350,000 |
| **Latency (avg)** | 0.80ms | 0.28ms |
| **Memory** | Normal | Lower |
| **Stability** | Excellent | Good |
| **Compatibility** | 100% | 95% |
| **Setup** | Zero config | Requires native module |

### When to Use Each

**Node.js HTTP:**
- Default choice
- Maximum compatibility
- Cloud platforms (Vercel, Railway, etc.)
- Windows development
- <100K req/s needed

**uWebSockets.js:**
- Maximum performance needed
- >100K req/s traffic
- Cost optimization
- Linux/macOS servers
- Microservices

---

## Quick Wins

### 1. Enable uWebSockets

```typescript
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// 2-3x performance improvement
```

### 2. Use Validation Efficiently

```typescript
// Bad: Validate everything
validate({
  body: hugeSchema,
  query: hugeSchema,
  params: hugeSchema,
})

// Good: Validate only what's needed
validate({
  body: z.object({
    email: z.string().email(),
  }),
})
```

### 3. Minimize Middleware

```typescript
// Bad: Many middleware
app.use(mw1, mw2, mw3, mw4, mw5);

// Good: Only necessary middleware
app.use(authenticate, logger);
```

### 4. Cache Expensive Operations

```typescript
// Bad: Query on every request
app.get('/config', async (ctx) => {
  const config = await db.config.find();
  ctx.json(config);
});

// Good: Cache static data
const configCache = await db.config.find();
app.get('/config', async (ctx) => {
  ctx.json(configCache);
});
```

### 5. Use Profiling

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 100, // Flag >100ms
    },
  },
});

// Automatically detects bottlenecks
```

---

## Performance Monitoring

### Built-in Metrics

```typescript
import { getMetrics } from 'ramapi';

app.get('/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.json({
    requestsPerSecond: metrics.requestsPerSecond,
    p95Latency: metrics.p95Latency,
    p99Latency: metrics.p99Latency,
  });
});
```

### Profiling Dashboard

```typescript
app.get('/debug/slow', async (ctx) => {
  const profiles = await getProfiles({ slowOnly: true });
  ctx.json({ profiles });
});
```

---

## Complete Example

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  // Use uWebSockets for maximum performance
  adapter: {
    type: 'uwebsockets',
  },

  // Enable profiling
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 100,
      autoDetectBottlenecks: true,
    },
    metrics: {
      enabled: true,
    },
  },
});

// Optimized route
app.get('/api/fast', async (ctx) => {
  ctx.json({ message: 'Fast!' });
});

// Metrics endpoint
app.get('/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.json(metrics);
});

await app.listen(3000);
console.log('Running with maximum performance');
```

---

## Next Steps

- [HTTP Adapters Guide](http-adapters.md)
- [Benchmarks](benchmarks.md)
- [Optimization Guide](optimization-guide.md)

---

**Need help?** Check the [Performance Profiling](../observability/profiling.md) guide or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
