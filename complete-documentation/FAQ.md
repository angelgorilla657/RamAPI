# Frequently Asked Questions (FAQ)

Comprehensive answers to common questions about RamAPI.

> **Note**: All answers are based on verified RamAPI source code and real-world usage patterns.

## Table of Contents

1. [General Questions](#general-questions)
2. [Getting Started](#getting-started)
3. [Performance](#performance)
4. [Features & Capabilities](#features--capabilities)
5. [Deployment & Production](#deployment--production)
6. [Troubleshooting](#troubleshooting)
7. [Comparison with Other Frameworks](#comparison-with-other-frameworks)

---

## General Questions

### What is RamAPI?

RamAPI is an ultra-fast, TypeScript-first API framework for Node.js that provides:
- **Extreme performance** (400K+ requests/second)
- **Built-in observability** (distributed tracing, profiling, flow visualization)
- **Zero-overhead middleware** through pre-compilation
- **Multi-protocol support** (REST, GraphQL, gRPC)
- **Production-ready** features out of the box

### Why should I use RamAPI over Express/Fastify/Koa?

**Performance**: 10x faster than Express, 4x faster than Fastify
- RamAPI: 400K+ req/s
- Fastify: 90K req/s
- Express: 40K req/s

**Built-in Features**:
- Distributed tracing and profiling (no setup needed)
- Request flow visualization
- JWT authentication
- Input validation with Zod
- Multi-protocol support

**Developer Experience**:
- Full TypeScript support
- Modern async/await everywhere
- Zero-config for common tasks
- Excellent error messages

### Is RamAPI production-ready?

Yes! RamAPI includes production-ready features:
- Graceful shutdown
- Health checks
- Error handling
- Rate limiting
- CORS support
- Security best practices
- Comprehensive observability

### What's the license?

RamAPI is MIT licensed - free for commercial and personal use.

---

## Getting Started

### How do I install RamAPI?

```bash
npm install ramapi
```

Requirements:
- Node.js 18+ (for native fetch and performance APIs)
- TypeScript 5.0+ (recommended)

### What's the simplest example?

```typescript
import { createApp } from 'ramapi';

const app = createApp();

app.get('/', (ctx) => {
  ctx.json({ message: 'Hello, World!' });
});

app.listen(3000);
```

### Do I need TypeScript?

No, but highly recommended. RamAPI works with JavaScript but provides excellent TypeScript support with full type inference.

JavaScript example:
```javascript
import { createApp } from 'ramapi';

const app = createApp();
app.get('/', (ctx) => ctx.json({ hello: 'world' }));
app.listen(3000);
```

### How do I migrate from Express?

See our [Migration Guide](guides/migration-guide.md). Key differences:

**Express:**
```typescript
app.get('/users', (req, res) => {
  res.json({ users: [] });
});
```

**RamAPI:**
```typescript
app.get('/users', (ctx) => {
  ctx.json({ users: [] });
});
```

---

## Performance

### How is RamAPI so fast?

1. **Pre-compiled middleware chains** - Zero overhead at request time
2. **Optimized routing** - O(1) lookup for static routes, cached dynamic routes
3. **Minimal abstractions** - Direct access to Node.js internals
4. **Zero dependencies** for core (only for optional features)
5. **uWebSockets.js adapter** - Optional ultra-fast HTTP server

### What adapter should I use?

**Development**: `node-http` (default, stable)
```typescript
const app = createApp({
  adapter: { type: 'node-http' }
});
```

**Production**: `uwebsockets` (4-10x faster)
```typescript
const app = createApp({
  adapter: { type: 'uwebsockets' }
});
```

Note: uWebSockets requires native compilation but provides 400K+ req/s.

### Can I handle high concurrency?

Yes! RamAPI is designed for high concurrency:

**Cluster mode** (recommended):
```bash
pm2 start dist/index.js -i max
```

**Node.js cluster**:
```typescript
import cluster from 'cluster';
import { cpus } from 'os';

if (cluster.isPrimary) {
  for (let i = 0; i < cpus().length; i++) {
    cluster.fork();
  }
} else {
  const app = createApp();
  // ... routes
  app.listen(3000);
}
```

### What's the memory footprint?

- **Base**: ~50MB per instance
- **With tracing**: ~70MB
- **With profiling**: ~80MB
- **Production**: 100-200MB typical

Scales horizontally across multiple processes/machines.

---

## Features & Capabilities

### Does RamAPI support WebSockets?

Not directly. RamAPI focuses on HTTP APIs. For WebSockets, use alongside:

```typescript
import { WebSocketServer } from 'ws';

// RamAPI for HTTP
const app = createApp();
app.listen(3000);

// ws for WebSockets
const wss = new WebSocketServer({ port: 8080 });
wss.on('connection', (ws) => {
  ws.send('Hello!');
});
```

### Can I serve static files?

Yes, but use nginx in production:

**Development**:
```typescript
import { readFile } from 'fs/promises';

app.get('/static/*', async (ctx) => {
  const file = await readFile(`./public${ctx.path}`);
  ctx.send(file);
});
```

**Production**: Use nginx to serve static files (much faster).

### Does it support file uploads?

Yes, with a multipart parser:

```typescript
import formidable from 'formidable';

app.post('/upload', async (ctx) => {
  const form = formidable({ multiples: true });
  const [fields, files] = await form.parse(ctx.req);
  ctx.json({ files });
});
```

### Can I use GraphQL?

Yes! RamAPI has built-in GraphQL support:

```typescript
import { GraphQLAdapter } from 'ramapi';
import { buildSchema } from 'graphql';

const schema = buildSchema(`
  type Query {
    hello: String
  }
`);

const graphql = new GraphQLAdapter({
  schema,
  rootValue: {
    hello: () => 'Hello, World!'
  }
});

app.use('/graphql', graphql.middleware());
```

See [GraphQL Integration](multi-protocol/graphql.md) for details.

### Does it support gRPC?

Yes! See [gRPC Integration](multi-protocol/grpc.md):

```typescript
import { GRPCAdapter } from 'ramapi';

const grpc = new GRPCAdapter({
  protoPath: './protos/service.proto',
  package: 'myservice',
  service: 'MyService'
});

app.use('/grpc', grpc.middleware());
```

### How do I handle authentication?

RamAPI includes JWT authentication:

```typescript
import { JWTService, authenticate } from 'ramapi';

const jwtService = new JWTService({ secret: 'your-secret' });
const auth = authenticate(jwtService);

// Protected route
app.get('/profile', auth, (ctx) => {
  ctx.json({ user: ctx.user });
});
```

See [Adding Authentication](guides/adding-authentication.md) guide.

### How do I validate input?

Built-in validation with Zod:

```typescript
import { z } from 'zod';
import { validate } from 'ramapi';

const userSchema = z.object({
  name: z.string().min(1),
  email: z.string().email()
});

app.post('/users', validate({ body: userSchema }), (ctx) => {
  // ctx.body is validated and typed!
  ctx.json({ user: ctx.body });
});
```

### Can I use existing Express middleware?

Not directly. RamAPI middleware is different. You need to convert:

```typescript
// Express middleware
function expressMiddleware(req, res, next) { }

// Convert to RamAPI
function ramApiMiddleware(ctx, next) {
  return new Promise((resolve, reject) => {
    expressMiddleware(ctx.req, ctx.res, (err) => {
      if (err) reject(err);
      else next().then(resolve).catch(reject);
    });
  });
}
```

---

## Deployment & Production

### How do I deploy to production?

Multiple options:

**Docker**:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY dist ./dist
CMD ["node", "dist/index.js"]
```

**PM2**:
```bash
pm2 start dist/index.js -i max --name api
```

**Cloud platforms**: AWS ECS, Google Cloud Run, Azure App Service, Vercel, Railway

See [Cloud Deployment](deployment/cloud-deployment.md) guide.

### What environment variables should I set?

Essential:
```env
NODE_ENV=production
PORT=3000
JWT_SECRET=your-secret-key-here
DATABASE_URL=postgresql://...
```

Optional:
```env
TRACING_ENABLED=true
TRACING_SAMPLE_RATE=0.1
LOG_LEVEL=info
CORS_ORIGINS=https://yourdomain.com
```

### How do I monitor my API?

RamAPI includes built-in observability:

**Distributed Tracing**:
```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      exporter: 'otlp',
      endpoint: 'http://jaeger:4318/v1/traces'
    }
  }
});
```

**Integration**: Jaeger, Zipkin, Datadog, New Relic

See [Setup Observability](guides/setup-observability.md) guide.

### How do I handle errors in production?

Comprehensive error handling:

```typescript
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error: any) {
    // Log error
    console.error('Error:', error);

    // Send appropriate response
    if (error instanceof HTTPError) {
      ctx.status(error.statusCode);
      ctx.json({ error: error.message });
    } else {
      ctx.status(500);
      ctx.json({
        error: 'Internal server error',
        // Don't leak details in production
        ...(process.env.NODE_ENV !== 'production' && { stack: error.stack })
      });
    }
  }
});
```

### How do I implement graceful shutdown?

```typescript
const shutdown = async (signal: string) => {
  console.log(`${signal} received, shutting down gracefully`);

  // Stop accepting new requests
  await app.close();

  // Close database connections
  await db.end();

  // Exit
  process.exit(0);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

### What about rate limiting?

Built-in rate limiting:

```typescript
import { rateLimit } from 'ramapi';

// Global rate limit
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // 100 requests per window
}));

// Stricter limit for sensitive endpoints
app.post('/login',
  rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }),
  loginHandler
);
```

---

## Troubleshooting

### My requests hang - what's wrong?

Common causes:

1. **Missing `await next()`**:
```typescript
// ❌ Wrong
app.use(async (ctx, next) => {
  next(); // Missing await!
});

// ✅ Correct
app.use(async (ctx, next) => {
  await next();
});
```

2. **Not sending response**:
```typescript
// ❌ Wrong
app.get('/users', (ctx) => {
  const users = getUsers();
  // Missing ctx.json()!
});

// ✅ Correct
app.get('/users', (ctx) => {
  const users = getUsers();
  ctx.json({ users });
});
```

### Why is my route returning 404?

Check:

1. Route path matches exactly
2. HTTP method matches (GET vs POST)
3. Middleware isn't blocking the request
4. Route order (specific routes before dynamic ones)

```typescript
// ✅ Correct order
app.get('/users/me', getMe);      // Specific first
app.get('/users/:id', getUser);  // Dynamic second

// ❌ Wrong order
app.get('/users/:id', getUser);   // Catches everything
app.get('/users/me', getMe);     // Never reached!
```

### Tracing isn't working?

Requirements:

1. Tracing must be enabled:
```typescript
observability: {
  tracing: {
    enabled: true, // Must be true!
    sampleRate: 1.0 // Must be > 0
  }
}
```

2. For OTLP export, ensure collector is running:
```bash
docker ps | grep jaeger
```

See [Troubleshooting Guide](guides/troubleshooting.md) for more.

---

## Comparison with Other Frameworks

### RamAPI vs Express

| Feature | Express | RamAPI |
|---------|---------|--------|
| Performance | 40K req/s | 400K+ req/s |
| TypeScript | Add-on | Built-in |
| Validation | Manual | Built-in |
| Observability | Manual | Built-in |
| Learning Curve | Easy | Easy |
| Ecosystem | Huge | Growing |

**Migrate if**: You need better performance, observability, or TypeScript support.

### RamAPI vs Fastify

| Feature | Fastify | RamAPI |
|---------|---------|--------|
| Performance | 90K req/s | 400K+ req/s |
| Schema Validation | JSON Schema | Zod |
| Observability | Plugins | Built-in |
| TypeScript | Good | Excellent |
| Plugins | Many | Growing |

**Migrate if**: You need extreme performance or prefer Zod over JSON Schema.

### RamAPI vs NestJS

| Feature | NestJS | RamAPI |
|---------|--------|--------|
| Performance | ~40K req/s | 400K+ req/s |
| Architecture | Opinionated | Flexible |
| Learning Curve | Steep | Easy |
| TypeScript | Excellent | Excellent |
| DI Container | Yes | No (bring your own) |
| Observability | Plugins | Built-in |

**Migrate if**: You want simpler architecture with better performance.

### RamAPI vs Hono

| Feature | Hono | RamAPI |
|---------|------|--------|
| Performance | ~300K req/s | 400K+ req/s |
| Edge Runtime | Yes | No |
| Node.js | Yes | Yes |
| Observability | Manual | Built-in |
| Multi-Protocol | No | Yes (GraphQL, gRPC) |

**Choose RamAPI if**: You need observability or multi-protocol support.
**Choose Hono if**: You're deploying to edge (Cloudflare Workers, Deno Deploy).

---

## Best Practices

### Project Structure

```
src/
├── index.ts           # Server entry point
├── routes/
│   ├── users.ts       # User routes
│   └── posts.ts       # Post routes
├── middleware/
│   ├── auth.ts        # Auth middleware
│   └── logger.ts      # Logging
├── services/
│   └── database.ts    # Database layer
└── types.ts           # Shared types
```

### Error Handling

Always use try-catch in async handlers:

```typescript
app.get('/users/:id', async (ctx) => {
  try {
    const user = await db.getUser(ctx.params.id);
    if (!user) {
      throw new HTTPError(404, 'User not found');
    }
    ctx.json({ user });
  } catch (error) {
    throw error; // Re-throw to global error handler
  }
});
```

### Security

1. Use environment variables for secrets
2. Enable CORS properly
3. Add rate limiting
4. Validate all input
5. Use HTTPS in production
6. Keep dependencies updated

See [Security Best Practices](advanced/security.md).

### Performance

1. Use caching (Redis)
2. Enable connection pooling
3. Reduce sample rate in production (10%)
4. Use horizontal scaling
5. Add CDN for static assets

See [Scaling & Architecture](advanced/scaling.md).

---

## Getting Help

- **Documentation**: Complete guides in `complete-documentation/`
- **Examples**: Working examples in `examples/` directory
- **Troubleshooting**: [Troubleshooting Guide](guides/troubleshooting.md)
- **Issues**: Report bugs on GitHub
- **Community**: Join discussions on GitHub

---

## See Also

- [Quick Start](getting-started/quick-start.md)
- [Building a REST API](guides/building-rest-api.md)
- [Migration Guide](guides/migration-guide.md)
- [Troubleshooting](guides/troubleshooting.md)
