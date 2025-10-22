# Router API Reference

Complete API reference for the RamAPI Router class.

## Table of Contents

1. [Router Class](#router-class)
2. [Route Registration](#route-registration)
3. [Route Groups](#route-groups)
4. [Middleware](#middleware)
5. [Route Matching](#route-matching)
6. [Performance Optimizations](#performance-optimizations)
7. [Advanced Usage](#advanced-usage)

---

## Router Class

The Router class manages routes and middleware with high-performance route matching.

### Constructor

```typescript
class Router {
  constructor(config?: RouterConfig)
}
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `config` | `RouterConfig` | No | Router configuration |

#### RouterConfig

```typescript
interface RouterConfig {
  prefix?: string;
  middleware?: Middleware[];
}
```

### Example

```typescript
import { Router } from 'ramapi';

// Basic router
const router = new Router();

// With prefix
const apiRouter = new Router({
  prefix: '/api',
});

// With middleware
const adminRouter = new Router({
  prefix: '/admin',
  middleware: [authenticate, requireAdmin],
});
```

---

## Route Registration

### HTTP Method Shortcuts

Register routes using HTTP method shortcuts.

#### get()

```typescript
get(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a GET route.

```typescript
router.get('/users', async (ctx) => {
  ctx.json({ users: [] });
});

// With middleware
router.get('/users', authenticate, async (ctx) => {
  ctx.json({ users: [] });
});

// Multiple middleware
router.get('/users',
  authenticate,
  rateLimit({ max: 100 }),
  logger(),
  async (ctx) => {
    ctx.json({ users: [] });
  }
);
```

#### post()

```typescript
post(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a POST route.

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

router.post('/users',
  validate({ body: userSchema }),
  async (ctx) => {
    const user = ctx.body;
    ctx.json({ user }, 201);
  }
);
```

#### put()

```typescript
put(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a PUT route.

```typescript
router.put('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  ctx.json({ updated: true });
});
```

#### patch()

```typescript
patch(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a PATCH route.

```typescript
router.patch('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  ctx.json({ patched: true });
});
```

#### delete()

```typescript
delete(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a DELETE route.

```typescript
router.delete('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  ctx.status(204);
});
```

#### options()

```typescript
options(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register an OPTIONS route.

```typescript
router.options('/users', async (ctx) => {
  ctx.setHeader('Allow', 'GET, POST, PUT, DELETE');
  ctx.status(204);
});
```

#### head()

```typescript
head(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a HEAD route.

```typescript
router.head('/users/:id', async (ctx) => {
  const exists = await db.userExists(ctx.params.id);
  ctx.status(exists ? 200 : 404);
});
```

#### all()

```typescript
all(path: string, ...handlers: (Handler | Middleware)[]): Router
```

Register a route for all HTTP methods.

```typescript
router.all('/health', async (ctx) => {
  ctx.json({ status: 'ok' });
});
```

---

## Route Groups

Group routes with shared prefix and middleware.

### group()

```typescript
group(prefix: string, fn: (router: Router) => void): Router
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `prefix` | `string` | URL prefix for grouped routes |
| `fn` | `(router: Router) => void` | Function to register routes in the group |

#### Examples

**Basic grouping:**

```typescript
router.group('/api', (r) => {
  r.get('/users', getUsers);
  r.post('/users', createUser);
  r.get('/users/:id', getUser);
});

// Results in:
// GET  /api/users
// POST /api/users
// GET  /api/users/:id
```

**Nested groups:**

```typescript
router.group('/api', (api) => {
  api.group('/v1', (v1) => {
    v1.get('/users', getUsersV1);
  });

  api.group('/v2', (v2) => {
    v2.get('/users', getUsersV2);
  });
});

// Results in:
// GET /api/v1/users
// GET /api/v2/users
```

**Groups with middleware:**

```typescript
router.group('/admin', (admin) => {
  // Middleware for all admin routes
  admin.use(authenticate);
  admin.use(requireAdmin);

  admin.get('/stats', getStats);
  admin.delete('/users/:id', deleteUser);
});

// All routes in /admin group have authenticate + requireAdmin middleware
```

**Multiple groups:**

```typescript
// Public API
router.group('/api/public', (r) => {
  r.get('/posts', getPosts);
  r.get('/posts/:id', getPost);
});

// Authenticated API
router.group('/api/private', (r) => {
  r.use(authenticate);
  r.get('/profile', getProfile);
  r.put('/profile', updateProfile);
});

// Admin API
router.group('/api/admin', (r) => {
  r.use(authenticate);
  r.use(requireAdmin);
  r.get('/users', getAllUsers);
  r.delete('/users/:id', deleteUser);
});
```

---

## Middleware

### use()

Mount middleware or nested routers.

#### Global Middleware

```typescript
use(middleware: Middleware): Router
```

Apply middleware to all routes in this router.

```typescript
import { logger, cors } from 'ramapi';

router.use(logger());
router.use(cors());

// All routes below have logger + cors middleware
router.get('/users', getUsers);
router.post('/users', createUser);
```

#### Nested Router

```typescript
use(prefix: string, router: Router): Router
```

Mount another router with a prefix.

```typescript
const apiRouter = new Router();
apiRouter.get('/users', getUsers);
apiRouter.get('/posts', getPosts);

const mainRouter = new Router();
mainRouter.use('/api', apiRouter);

// Results in:
// GET /api/users
// GET /api/posts
```

#### Examples

**Combining middleware:**

```typescript
const router = new Router();

// Global middleware
router.use(logger());
router.use(cors());

// Route-specific middleware
router.get('/public', async (ctx) => {
  ctx.json({ public: true });
});

router.get('/private',
  authenticate, // Route-specific
  async (ctx) => {
    ctx.json({ user: ctx.user });
  }
);
```

**Nested routers with prefixes:**

```typescript
const usersRouter = new Router();
usersRouter.get('/', getUsers);
usersRouter.get('/:id', getUser);
usersRouter.post('/', createUser);

const postsRouter = new Router();
postsRouter.get('/', getPosts);
postsRouter.get('/:id', getPost);

const apiRouter = new Router();
apiRouter.use('/users', usersRouter);
apiRouter.use('/posts', postsRouter);

const app = createApp();
app.use('/api', apiRouter);

// Results in:
// GET  /api/users
// GET  /api/users/:id
// POST /api/users
// GET  /api/posts
// GET  /api/posts/:id
```

---

## Route Matching

### Dynamic Routes

Routes can contain dynamic segments using `:param` syntax.

```typescript
// Single parameter
router.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  ctx.json({ id: userId });
});

// Multiple parameters
router.get('/users/:userId/posts/:postId', async (ctx) => {
  const { userId, postId } = ctx.params;
  ctx.json({ userId, postId });
});

// Optional segments (use separate routes)
router.get('/posts', getAllPosts);
router.get('/posts/:id', getPostById);
```

### Route Priority

Routes are matched in this order:

1. **Exact static routes** (O(1) hash map lookup)
2. **Dynamic routes** (in registration order)

```typescript
// This will match first (static)
router.get('/users/me', async (ctx) => {
  ctx.json({ user: 'current user' });
});

// This will match if above doesn't (dynamic)
router.get('/users/:id', async (ctx) => {
  ctx.json({ user: ctx.params.id });
});

// GET /users/me → matches first route
// GET /users/123 → matches second route
```

### Route Parameters

Parameters are extracted from the URL and available in `ctx.params`.

```typescript
router.get('/users/:userId/posts/:postId/comments/:commentId',
  async (ctx) => {
    console.log(ctx.params);
    // {
    //   userId: '123',
    //   postId: '456',
    //   commentId: '789'
    // }
  }
);
```

---

## Performance Optimizations

The Router implements several performance optimizations:

### 1. Static Route Map (O(1) Lookup)

Static routes use a hash map for instant lookups.

```typescript
router.get('/users', getUsers);
router.get('/posts', getPosts);
router.get('/comments', getComments);

// Each route: O(1) lookup time
// No regex matching, no iteration
```

### 2. Pre-compiled Route Patterns

Dynamic routes are compiled once at registration time.

```typescript
router.get('/users/:id/posts/:postId', handler);
// Compiled at registration: parts = ['users', ':id', 'posts', ':postId']
// At request time: direct array comparison, no parsing
```

### 3. Pre-compiled Middleware Chains

Middleware chains are compiled once at registration.

```typescript
router.get('/users',
  authenticate,
  logger(),
  rateLimit(),
  handler
);

// Middleware chain compiled once at registration
// At request time: single function call, no chain iteration
```

### 4. Last Route Cache

Hot path optimization for repeated requests to the same route.

```typescript
// First request to /users
router.findRoute('GET', '/users'); // Full lookup

// Subsequent requests
router.findRoute('GET', '/users'); // Cache hit (instant)
```

### 5. Separate Static/Dynamic Storage

Static and dynamic routes stored separately.

```typescript
// Static routes (fast path)
router.get('/users', handler);
router.get('/posts', handler);

// Dynamic routes (fallback)
router.get('/users/:id', handler);

// GET /users → checks static routes only (O(1))
// GET /users/123 → static miss, checks dynamic routes
```

---

## Advanced Usage

### Type-Safe Routes

Use TypeScript for type-safe route handlers.

```typescript
import { Context } from 'ramapi';

interface User {
  id: number;
  name: string;
  email: string;
}

interface UserParams {
  id: string;
}

router.get('/users/:id', async (ctx: Context<unknown, unknown, UserParams>) => {
  const userId = parseInt(ctx.params.id);
  const user: User = await db.getUser(userId);
  ctx.json({ user });
});
```

### Custom Route Metadata

```typescript
interface Route {
  method: HTTPMethod;
  path: string;
  handler: Handler;
  middleware?: Middleware[];
  schema?: {
    body?: ZodSchema;
    query?: ZodSchema;
    params?: ZodSchema;
  };
  meta?: {
    description?: string;
    tags?: string[];
    auth?: boolean;
  };
}
```

### Accessing Routes

```typescript
const routes = router.getRoutes();

routes.forEach((route) => {
  console.log(`${route.method} ${route.path}`);
});

// Generate documentation
const docs = routes.map((route) => ({
  method: route.method,
  path: route.path,
  description: route.meta?.description,
  tags: route.meta?.tags,
}));
```

### Manual Route Handling

```typescript
const match = router.findRoute('GET', '/users/123');

if (match) {
  const { route, params } = match;
  console.log('Route found:', route.path);
  console.log('Params:', params); // { id: '123' }
}
```

---

## Complete Example

```typescript
import { Router, createApp } from 'ramapi';
import { authenticate, validate, logger } from 'ramapi';
import { z } from 'zod';

// Create routers
const apiRouter = new Router({
  prefix: '/api',
  middleware: [logger()],
});

// Public routes
apiRouter.get('/health', async (ctx) => {
  ctx.json({ status: 'ok' });
});

// User routes
apiRouter.group('/users', (users) => {
  users.get('/', async (ctx) => {
    ctx.json({ users: [] });
  });

  users.get('/:id', async (ctx) => {
    ctx.json({ user: { id: ctx.params.id } });
  });

  users.post('/',
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
});

// Admin routes
apiRouter.group('/admin', (admin) => {
  admin.use(authenticate);
  admin.use(requireAdmin);

  admin.get('/stats', async (ctx) => {
    ctx.json({ stats: {} });
  });

  admin.delete('/users/:id', async (ctx) => {
    ctx.status(204);
  });
});

// Mount router
const app = createApp();
app.use('/', apiRouter);

await app.listen(3000);

// Routes available:
// GET    /api/health
// GET    /api/users
// GET    /api/users/:id
// POST   /api/users
// GET    /api/admin/stats (requires auth + admin)
// DELETE /api/admin/users/:id (requires auth + admin)
```

---

## See Also

- [Server API](server.md) - Server class and configuration
- [Context API](context.md) - Request/response context
- [Middleware API](middleware-api.md) - Built-in middleware
- [Validation API](validation.md) - Request validation

---

**Need help?** Check the [Routing Guide](../core-concepts/routing.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
