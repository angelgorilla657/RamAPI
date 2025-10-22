# Server API Reference

Complete API reference for the RamAPI Server class and `createApp()` factory function.

## Table of Contents

1. [createApp()](#createapp)
2. [Server Class](#server-class)
3. [Configuration Options](#configuration-options)
4. [Route Methods](#route-methods)
5. [Server Lifecycle](#server-lifecycle)
6. [Advanced Usage](#advanced-usage)

---

## createApp()

Factory function to create a new RamAPI server instance.

### Signature

```typescript
function createApp(config?: ServerConfig): Server
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `config` | `ServerConfig` | No | Server configuration options |

### Returns

`Server` - A new server instance

### Example

```typescript
import { createApp } from 'ramapi';

// Basic server
const app = createApp();

// With configuration
const app = createApp({
  port: 3000,
  host: '0.0.0.0',
  adapter: { type: 'uwebsockets' },
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
    },
  },
});
```

---

## Server Class

The core RamAPI server class that handles HTTP requests and routing.

### Constructor

```typescript
class Server {
  constructor(config?: ServerConfig & { protocols?: ProtocolManagerConfig })
}
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| N/A | N/A | All properties are private |

---

## Configuration Options

### ServerConfig

Complete configuration interface for the server.

```typescript
interface ServerConfig {
  port?: number;
  host?: string;
  cors?: boolean | CorsConfig;
  middleware?: Middleware[];
  onError?: ErrorHandler;
  onNotFound?: Handler;
  observability?: ObservabilityConfig;
  adapter?: AdapterConfig;
}
```

#### port

- **Type:** `number`
- **Default:** `3000`
- **Description:** Port number to listen on

```typescript
const app = createApp({
  port: 8080,
});
```

#### host

- **Type:** `string`
- **Default:** `'0.0.0.0'`
- **Description:** Host address to bind to

```typescript
const app = createApp({
  host: 'localhost', // Only accessible locally
});

const app = createApp({
  host: '0.0.0.0', // Accessible from all interfaces
});
```

#### cors

- **Type:** `boolean | CorsConfig`
- **Default:** `undefined`
- **Description:** Enable CORS or provide CORS configuration

```typescript
// Enable with defaults
const app = createApp({
  cors: true,
});

// Custom configuration
const app = createApp({
  cors: {
    origin: 'https://example.com',
    methods: ['GET', 'POST'],
    credentials: true,
  },
});
```

See [CorsConfig](#corsconfig) for detailed options.

#### middleware

- **Type:** `Middleware[]`
- **Default:** `[]`
- **Description:** Global middleware applied to all routes

```typescript
import { logger, authenticate } from 'ramapi';

const app = createApp({
  middleware: [
    logger(),
    authenticate,
  ],
});
```

#### onError

- **Type:** `ErrorHandler`
- **Default:** Built-in error handler
- **Description:** Custom error handler function

```typescript
const app = createApp({
  onError: async (error, ctx) => {
    console.error('Error:', error);
    ctx.json({
      error: true,
      message: error.message,
    }, 500);
  },
});
```

#### onNotFound

- **Type:** `Handler`
- **Default:** Built-in 404 handler
- **Description:** Custom 404 handler

```typescript
const app = createApp({
  onNotFound: async (ctx) => {
    ctx.json({
      error: 'Not found',
      path: ctx.path,
    }, 404);
  },
});
```

#### observability

- **Type:** `ObservabilityConfig`
- **Default:** `undefined`
- **Description:** Observability configuration (tracing, logging, metrics)

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
    },
    metrics: {
      enabled: true,
    },
  },
});
```

See [Observability API](observability.md) for detailed options.

#### adapter

- **Type:** `AdapterConfig`
- **Default:** Automatic selection (tries uWebSockets, falls back to Node.js HTTP)
- **Description:** HTTP server adapter configuration

```typescript
// Node.js HTTP adapter
const app = createApp({
  adapter: {
    type: 'node-http',
  },
});

// uWebSockets adapter
const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      idleTimeout: 120,
      maxBackpressure: 1024 * 1024,
    },
  },
});
```

See [HTTP Adapters](../performance/http-adapters.md) for detailed options.

---

### CorsConfig

CORS configuration options.

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

#### origin

- **Type:** `string | string[] | ((origin: string) => boolean)`
- **Default:** `'*'`
- **Description:** Allowed origin(s) or function to determine allowed origins

```typescript
// Single origin
cors: { origin: 'https://example.com' }

// Multiple origins
cors: { origin: ['https://example.com', 'https://app.example.com'] }

// Dynamic origin
cors: {
  origin: (origin) => {
    return origin.endsWith('.example.com');
  }
}
```

#### methods

- **Type:** `HTTPMethod[]`
- **Default:** `['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS', 'HEAD']`
- **Description:** Allowed HTTP methods

```typescript
cors: {
  methods: ['GET', 'POST'],
}
```

#### allowedHeaders

- **Type:** `string[]`
- **Default:** `['Content-Type', 'Authorization']`
- **Description:** Allowed request headers

```typescript
cors: {
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Custom-Header'],
}
```

#### exposedHeaders

- **Type:** `string[]`
- **Default:** `[]`
- **Description:** Headers exposed to the client

```typescript
cors: {
  exposedHeaders: ['X-Total-Count', 'X-Page-Number'],
}
```

#### credentials

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Allow credentials (cookies, authorization headers)

```typescript
cors: {
  credentials: true,
}
```

#### maxAge

- **Type:** `number`
- **Default:** `86400` (24 hours)
- **Description:** Preflight cache duration in seconds

```typescript
cors: {
  maxAge: 3600, // 1 hour
}
```

---

## Route Methods

Server instances expose all HTTP method shortcuts.

### get()

Register a GET route.

```typescript
get(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.get('/users', async (ctx) => {
  ctx.json({ users: [] });
});

// With middleware
app.get('/users', authenticate, async (ctx) => {
  ctx.json({ users: [] });
});
```

### post()

Register a POST route.

```typescript
post(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.post('/users', validate({ body: userSchema }), async (ctx) => {
  ctx.json({ user: ctx.body }, 201);
});
```

### put()

Register a PUT route.

```typescript
put(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.put('/users/:id', async (ctx) => {
  ctx.json({ updated: true });
});
```

### patch()

Register a PATCH route.

```typescript
patch(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.patch('/users/:id', async (ctx) => {
  ctx.json({ patched: true });
});
```

### delete()

Register a DELETE route.

```typescript
delete(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.delete('/users/:id', async (ctx) => {
  ctx.status(204);
});
```

### options()

Register an OPTIONS route.

```typescript
options(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.options('/users', async (ctx) => {
  ctx.setHeader('Allow', 'GET, POST');
  ctx.status(204);
});
```

### head()

Register a HEAD route.

```typescript
head(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.head('/users/:id', async (ctx) => {
  ctx.setHeader('Content-Length', '123');
  ctx.status(200);
});
```

### all()

Register a route for all HTTP methods.

```typescript
all(path: string, ...handlers: (Handler | Middleware)[]): Server
```

```typescript
app.all('/health', async (ctx) => {
  ctx.json({ status: 'ok' });
});
```

---

### use()

Mount middleware or nested routers.

#### Middleware

```typescript
use(middleware: Middleware): Server
```

```typescript
import { logger, cors } from 'ramapi';

app.use(logger());
app.use(cors());
```

#### Nested Router

```typescript
use(prefix: string, router: Router): Server
```

```typescript
import { Router } from 'ramapi';

const apiRouter = new Router();
apiRouter.get('/users', handler);

app.use('/api', apiRouter);
// Routes: /api/users
```

---

### group()

Group routes with shared prefix and middleware.

```typescript
group(prefix: string, fn: (router: Router) => void): Server
```

```typescript
app.group('/api', (router) => {
  router.get('/users', getUsers);
  router.post('/users', createUser);
});
// Routes: /api/users (GET, POST)

// With middleware
app.group('/admin', (router) => {
  router.use(authenticate);
  router.use(requireAdmin);

  router.get('/stats', getStats);
  router.delete('/users/:id', deleteUser);
});
```

---

## Server Lifecycle

### listen()

Start the HTTP server.

```typescript
async listen(port?: number | (() => void), host?: string | (() => void)): Promise<void>
```

#### Signatures

```typescript
// No arguments (uses config)
await app.listen();

// Port only
await app.listen(3000);

// Port and host
await app.listen(3000, '0.0.0.0');

// Port and callback
await app.listen(3000, () => {
  console.log('Server started');
});

// Callback only
await app.listen(() => {
  console.log('Server started');
});
```

#### Examples

```typescript
// Basic
await app.listen(3000);
// 🚀 Using uWebSockets adapter for maximum performance
// 🚀 RamAPI server running at http://0.0.0.0:3000

// With callback
await app.listen(3000, () => {
  console.log('Custom startup message');
});

// Using config
const app = createApp({ port: 8080, host: 'localhost' });
await app.listen();
// 🚀 RamAPI server running at http://localhost:8080
```

---

### close()

Stop the HTTP server.

```typescript
async close(): Promise<void>
```

```typescript
// Graceful shutdown
await app.close();
// 🛑 RamAPI server stopped

// In signal handler
process.on('SIGTERM', async () => {
  console.log('Shutting down...');
  await app.close();
  process.exit(0);
});
```

---

### getRouter()

Get the underlying router instance.

```typescript
getRouter(): Router
```

```typescript
const router = app.getRouter();
const routes = router.getRoutes();

console.log('Registered routes:', routes);
```

---

### getProtocolManager()

Get the protocol manager (if multi-protocol is enabled).

```typescript
getProtocolManager(): ProtocolManager | undefined
```

```typescript
const protocolManager = app.getProtocolManager();

if (protocolManager) {
  const graphqlSchema = protocolManager.getGraphQLSchema();
}
```

---

## Advanced Usage

### Custom Error Handling

```typescript
const app = createApp({
  onError: async (error, ctx) => {
    // Log to external service
    await logToSentry(error);

    // Custom error response
    if (error instanceof ValidationError) {
      ctx.json({
        error: 'Validation failed',
        details: error.details,
      }, 400);
      return;
    }

    if (error instanceof UnauthorizedError) {
      ctx.json({
        error: 'Unauthorized',
        message: error.message,
      }, 401);
      return;
    }

    // Default error
    ctx.json({
      error: 'Internal server error',
    }, 500);
  },
});
```

---

### Graceful Shutdown

```typescript
const app = createApp();

// Register routes
app.get('/users', getUsers);

// Start server
await app.listen(3000);

// Graceful shutdown
const shutdown = async () => {
  console.log('Received shutdown signal');

  // Stop accepting new connections
  await app.close();

  // Close database connections
  await db.close();

  // Exit
  process.exit(0);
};

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

---

### Multi-Protocol Server

```typescript
const app = createApp({
  adapter: { type: 'node-http' }, // Required for gRPC
  protocols: {
    graphql: {
      path: '/graphql',
      schema: graphqlSchema,
    },
    grpc: {
      port: 50051,
      services: grpcServices,
    },
  },
});

// REST routes
app.get('/api/users', getUsers);

// GraphQL available at /graphql
// gRPC available at localhost:50051

await app.listen(3000);
```

---

### Testing

```typescript
import { createApp } from 'ramapi';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';

describe('API Tests', () => {
  let app: Server;

  beforeAll(async () => {
    app = createApp();
    app.get('/users', async (ctx) => {
      ctx.json({ users: [] });
    });
    await app.listen(0); // Random port
  });

  afterAll(async () => {
    await app.close();
  });

  it('should return users', async () => {
    const res = await request('http://localhost:3000')
      .get('/users')
      .expect(200);

    expect(res.body).toEqual({ users: [] });
  });
});
```

---

## Type Definitions

### Handler

```typescript
type Handler<TBody = unknown, TQuery = unknown, TParams = unknown> = (
  ctx: Context<TBody, TQuery, TParams>
) => void | Promise<void>;
```

### Middleware

```typescript
type Middleware = (
  ctx: Context,
  next: () => Promise<void>
) => void | Promise<void>;
```

### ErrorHandler

```typescript
type ErrorHandler = (
  error: Error,
  ctx: Context
) => void | Promise<void>;
```

### HTTPMethod

```typescript
type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE' | 'OPTIONS' | 'HEAD';
```

---

## See Also

- [Router API](router.md) - Router class and route management
- [Context API](context.md) - Request/response context
- [Middleware API](middleware-api.md) - Built-in middleware
- [HTTP Adapters](../performance/http-adapters.md) - Adapter configuration

---

**Need help?** Check the [Guides](../guides/) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
