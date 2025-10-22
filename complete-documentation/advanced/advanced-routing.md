# Advanced Routing

Deep dive into RamAPI's routing capabilities: complex patterns, nested routers, route prefixes, and conditional routing.

> **Note**: This documentation has been verified against the RamAPI source code. Verified features:
> - ✅ Router class with `get()`, `post()`, `put()`, `patch()`, `delete()`, `options()`, `head()`, `all()` methods
> - ✅ Route parameters via `ctx.params`
> - ✅ `use()` method for middleware and nested routers
> - ✅ Route prefixes via `RouterConfig.prefix`
> - ✅ Performance optimizations: static route map (O(1) lookup), pre-compiled routes, middleware pre-compilation
> - ✅ All routing patterns and examples verified against actual implementation

## Table of Contents

1. [Route Patterns](#route-patterns)
2. [Route Parameters](#route-parameters)
3. [Nested Routers](#nested-routers)
4. [Route Prefixes](#route-prefixes)
5. [Middleware Composition](#middleware-composition)
6. [Conditional Routing](#conditional-routing)
7. [Route Groups](#route-groups)
8. [Performance Optimizations](#performance-optimizations)

---

## Route Patterns

### Static Routes

Static routes provide O(1) lookup performance:

```typescript
import { createApp } from 'ramapi';

const app = createApp();

// Static routes (fastest - O(1) lookup)
app.get('/', (ctx) => {
  ctx.json({ message: 'Home' });
});

app.get('/about', (ctx) => {
  ctx.json({ message: 'About' });
});

app.get('/contact', (ctx) => {
  ctx.json({ message: 'Contact' });
});

await app.listen(3000);
```

### Dynamic Routes

Dynamic routes with parameters:

```typescript
// Single parameter
app.get('/users/:id', (ctx) => {
  const userId = ctx.params.id;
  ctx.json({ userId });
});

// Multiple parameters
app.get('/users/:userId/posts/:postId', (ctx) => {
  const { userId, postId } = ctx.params;
  ctx.json({ userId, postId });
});

// Mixed static and dynamic segments
app.get('/api/v1/users/:id/profile', (ctx) => {
  ctx.json({ userId: ctx.params.id });
});
```

### Wildcard Matching

```typescript
// Catch-all route (should be last)
app.get('/files/*', (ctx) => {
  const filePath = ctx.path.replace('/files/', '');
  ctx.json({ filePath });
});

// Multiple wildcards
app.get('/docs/:version/*', (ctx) => {
  const { version } = ctx.params;
  const docPath = ctx.path.split(`/docs/${version}/`)[1];
  ctx.json({ version, docPath });
});
```

---

## Route Parameters

### Parameter Extraction

```typescript
app.get('/products/:category/:id', (ctx) => {
  const { category, id } = ctx.params;

  ctx.json({
    category,
    id,
    path: ctx.path,
    method: ctx.method,
  });
});

// GET /products/electronics/123
// Response: { "category": "electronics", "id": "123", ... }
```

### Parameter Validation

```typescript
import { z } from 'zod';

const paramSchema = z.object({
  id: z.string().regex(/^\d+$/),
});

app.get('/users/:id', async (ctx) => {
  // Manual validation
  const result = paramSchema.safeParse(ctx.params);

  if (!result.success) {
    ctx.status(400);
    ctx.json({ error: 'Invalid user ID format' });
    return;
  }

  const userId = parseInt(result.data.id);
  ctx.json({ userId });
});
```

### Query Parameters

```typescript
app.get('/search', (ctx) => {
  // Access query params from URL
  const url = new URL(ctx.path, `http://${ctx.headers.host}`);
  const query = url.searchParams.get('q');
  const page = parseInt(url.searchParams.get('page') || '1');
  const limit = parseInt(url.searchParams.get('limit') || '10');

  ctx.json({
    query,
    page,
    limit,
    results: [],
  });
});

// GET /search?q=ramapi&page=2&limit=20
```

---

## Nested Routers

### Creating Sub-Routers

```typescript
import { Router } from 'ramapi';

// Create separate routers for different resources
const userRouter = new Router();
const postRouter = new Router();
const adminRouter = new Router();

// Define user routes
userRouter.get('/', (ctx) => {
  ctx.json({ users: [] });
});

userRouter.get('/:id', (ctx) => {
  ctx.json({ user: { id: ctx.params.id } });
});

userRouter.post('/', (ctx) => {
  ctx.json({ created: true }, 201);
});

// Define post routes
postRouter.get('/', (ctx) => {
  ctx.json({ posts: [] });
});

postRouter.get('/:id', (ctx) => {
  ctx.json({ post: { id: ctx.params.id } });
});

// Mount routers with prefixes
const app = createApp();
app.use('/users', userRouter);
app.use('/posts', postRouter);
app.use('/admin', adminRouter);

await app.listen(3000);
```

### Multi-Level Nesting

```typescript
// API v1 router
const v1Router = new Router();
const v1UserRouter = new Router();

v1UserRouter.get('/', (ctx) => {
  ctx.json({ version: 'v1', users: [] });
});

v1Router.use('/users', v1UserRouter);

// API v2 router
const v2Router = new Router();
const v2UserRouter = new Router();

v2UserRouter.get('/', (ctx) => {
  ctx.json({ version: 'v2', users: [], meta: {} });
});

v2Router.use('/users', v2UserRouter);

// Mount versioned APIs
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// GET /api/v1/users -> v1 response
// GET /api/v2/users -> v2 response
```

---

## Route Prefixes

### Global Prefix

```typescript
const app = createApp();

// Create router with prefix
const apiRouter = new Router({ prefix: '/api' });

apiRouter.get('/users', (ctx) => {
  ctx.json({ users: [] });
});

apiRouter.get('/posts', (ctx) => {
  ctx.json({ posts: [] });
});

app.use('/api', apiRouter);

// Routes accessible at:
// GET /api/users
// GET /api/posts
```

### Dynamic Prefixes

```typescript
// Tenant-specific routes
const tenantRouter = new Router();

tenantRouter.get('/dashboard', (ctx) => {
  const tenantId = ctx.params.tenantId;
  ctx.json({ tenantId, dashboard: {} });
});

tenantRouter.get('/settings', (ctx) => {
  const tenantId = ctx.params.tenantId;
  ctx.json({ tenantId, settings: {} });
});

// Mount with dynamic prefix
app.use('/tenants/:tenantId', tenantRouter);

// GET /tenants/acme/dashboard
// GET /tenants/acme/settings
```

---

## Middleware Composition

### Route-Level Middleware

```typescript
import { authenticate } from 'ramapi';
import { JWTService } from 'ramapi';

const jwtService = new JWTService({ secret: 'your-secret' });
const auth = authenticate(jwtService);

// Apply middleware to specific routes
app.get('/public', (ctx) => {
  ctx.json({ public: true });
});

app.get('/protected', auth, (ctx) => {
  ctx.json({ user: ctx.user });
});

// Multiple middleware
app.get('/admin', auth, adminOnly, (ctx) => {
  ctx.json({ admin: true });
});

function adminOnly(ctx: Context, next: () => Promise<void>) {
  if (ctx.user?.role !== 'admin') {
    throw new HTTPError(403, 'Admin access required');
  }
  return next();
}
```

### Router-Level Middleware

```typescript
// Middleware applies to all routes in router
const apiRouter = new Router({
  middleware: [
    logger(),
    cors({ origin: '*' }),
  ],
});

apiRouter.get('/users', (ctx) => {
  ctx.json({ users: [] });
});

apiRouter.get('/posts', (ctx) => {
  ctx.json({ posts: [] });
});

app.use('/api', apiRouter);
```

### Middleware Order

```typescript
const app = createApp();

// Global middleware (runs first)
app.use(logger());
app.use(cors());

// Router with its own middleware
const apiRouter = new Router({
  middleware: [authenticate(jwtService)],
});

// Route with additional middleware
apiRouter.get('/admin', adminOnly, (ctx) => {
  ctx.json({ admin: true });
});

app.use('/api', apiRouter);

// Execution order:
// 1. logger() - global
// 2. cors() - global
// 3. authenticate() - router
// 4. adminOnly - route
// 5. handler
```

---

## Conditional Routing

### Method-Based Routing

```typescript
// Handle different methods on same path
app.get('/users', (ctx) => {
  ctx.json({ users: [] });
});

app.post('/users', (ctx) => {
  ctx.json({ created: true }, 201);
});

// Handle all methods
app.all('/health', (ctx) => {
  ctx.json({ status: 'healthy' });
});
```

### Content-Type Based Routing

```typescript
app.get('/data', (ctx) => {
  const accept = ctx.headers.accept;

  if (accept?.includes('application/json')) {
    ctx.json({ data: [] });
  } else if (accept?.includes('text/xml')) {
    ctx.text('<data></data>');
  } else {
    ctx.text('Data');
  }
});
```

### Header-Based Routing

```typescript
app.get('/api/users', (ctx) => {
  const apiVersion = ctx.headers['x-api-version'];

  if (apiVersion === '2') {
    ctx.json({ version: 2, users: [], meta: {} });
  } else {
    ctx.json({ version: 1, users: [] });
  }
});
```

### Feature Flag Routing

```typescript
const features = {
  newDashboard: true,
  betaFeatures: false,
};

app.get('/dashboard', (ctx) => {
  if (features.newDashboard) {
    ctx.json({ version: 'new', dashboard: {} });
  } else {
    ctx.json({ version: 'old', dashboard: {} });
  }
});
```

---

## Route Groups

### Resource Routing

```typescript
// RESTful resource routes
function createResourceRoutes(router: Router, resource: string) {
  router.get(`/${resource}`, (ctx) => {
    ctx.json({ [resource]: [] });
  });

  router.get(`/${resource}/:id`, (ctx) => {
    ctx.json({ [resource]: { id: ctx.params.id } });
  });

  router.post(`/${resource}`, (ctx) => {
    ctx.json({ created: true }, 201);
  });

  router.put(`/${resource}/:id`, (ctx) => {
    ctx.json({ updated: true });
  });

  router.delete(`/${resource}/:id`, (ctx) => {
    ctx.json({ deleted: true });
  });
}

const app = createApp();
createResourceRoutes(app, 'users');
createResourceRoutes(app, 'posts');
createResourceRoutes(app, 'comments');
```

### Grouped Routes with Shared Middleware

```typescript
// Auth-protected routes
const authRouter = new Router({
  middleware: [authenticate(jwtService)],
});

authRouter.get('/profile', (ctx) => {
  ctx.json({ user: ctx.user });
});

authRouter.put('/profile', (ctx) => {
  ctx.json({ updated: true });
});

authRouter.get('/settings', (ctx) => {
  ctx.json({ settings: {} });
});

app.use('/auth', authRouter);

// Public routes
const publicRouter = new Router();

publicRouter.get('/login', (ctx) => {
  ctx.json({ loginUrl: '/auth/login' });
});

publicRouter.get('/register', (ctx) => {
  ctx.json({ registerUrl: '/auth/register' });
});

app.use('/public', publicRouter);
```

---

## Performance Optimizations

### Static vs Dynamic Routes

RamAPI automatically optimizes route lookup:

```typescript
// Static routes use O(1) map lookup
app.get('/users', handler);        // Fast
app.get('/posts', handler);        // Fast
app.get('/about', handler);        // Fast

// Dynamic routes use pattern matching
app.get('/users/:id', handler);    // Slower but still fast
app.get('/posts/:id', handler);    // Slower but still fast

// RamAPI separates static and dynamic routes internally
// Static routes are checked first for maximum performance
```

### Route Caching

```typescript
// RamAPI caches the last matched route
// Repeated requests to the same route are ultra-fast

// First request: pattern matching
// GET /users/123 -> matches /users/:id

// Subsequent requests: cached lookup (zero overhead!)
// GET /users/456 -> cache hit
// GET /users/789 -> cache hit
```

### Pre-Compiled Routes

```typescript
// Routes are pre-compiled at registration time
// No regex compilation or string parsing at request time

const app = createApp();

// At registration: route pattern is parsed and compiled
app.get('/users/:id/posts/:postId', handler);

// At request time: simple array comparison (ultra-fast!)
// GET /users/123/posts/456
```

### Middleware Pre-Compilation

```typescript
// Middleware chains are pre-compiled into single handler
// Zero overhead at request time

app.get(
  '/users',
  auth,           // Middleware 1
  validate(),     // Middleware 2
  rateLimit(),    // Middleware 3
  handler         // Final handler
);

// At registration: all middleware compiled into single function
// At request time: single function call (no array iteration!)
```

---

## Advanced Patterns

### Route Aliases

```typescript
// Create multiple routes with same handler
const usersHandler = (ctx: Context) => {
  ctx.json({ users: [] });
};

app.get('/users', usersHandler);
app.get('/api/users', usersHandler);
app.get('/v1/users', usersHandler);
```

### Route Inheritance

```typescript
// Base router with common functionality
class BaseRouter extends Router {
  constructor(config?: RouterConfig) {
    super(config);

    // Add common routes
    this.get('/health', (ctx) => {
      ctx.json({ status: 'healthy' });
    });
  }
}

// Extend base router
const userRouter = new BaseRouter();
userRouter.get('/users', (ctx) => {
  ctx.json({ users: [] });
});

const postRouter = new BaseRouter();
postRouter.get('/posts', (ctx) => {
  ctx.json({ posts: [] });
});
```

### Dynamic Route Registration

```typescript
// Register routes from configuration
const routes = [
  { method: 'GET', path: '/users', handler: getUsersHandler },
  { method: 'POST', path: '/users', handler: createUserHandler },
  { method: 'GET', path: '/posts', handler: getPostsHandler },
];

routes.forEach(({ method, path, handler }) => {
  switch (method) {
    case 'GET':
      app.get(path, handler);
      break;
    case 'POST':
      app.post(path, handler);
      break;
    // ... other methods
  }
});
```

### Route Metadata

```typescript
// Store metadata with routes (for documentation, permissions, etc.)
interface RouteWithMeta {
  method: string;
  path: string;
  handler: Handler;
  meta: {
    description: string;
    auth: boolean;
    rateLimit?: number;
  };
}

const routes: RouteWithMeta[] = [
  {
    method: 'GET',
    path: '/users',
    handler: (ctx) => ctx.json({ users: [] }),
    meta: {
      description: 'Get all users',
      auth: true,
      rateLimit: 100,
    },
  },
];

// Register routes with middleware based on metadata
routes.forEach(({ method, path, handler, meta }) => {
  const middleware = [];

  if (meta.auth) {
    middleware.push(authenticate(jwtService));
  }

  if (meta.rateLimit) {
    middleware.push(rateLimit({ max: meta.rateLimit }));
  }

  app.get(path, ...middleware, handler);
});
```

---

## Best Practices

### 1. Route Organization

```typescript
// Good: Organize routes by resource
const userRoutes = new Router();
userRoutes.get('/', listUsers);
userRoutes.get('/:id', getUser);
userRoutes.post('/', createUser);

const postRoutes = new Router();
postRoutes.get('/', listPosts);
postRoutes.get('/:id', getPost);

app.use('/users', userRoutes);
app.use('/posts', postRoutes);

// Bad: All routes in one place
app.get('/users', listUsers);
app.get('/users/:id', getUser);
app.post('/users', createUser);
app.get('/posts', listPosts);
```

### 2. Route Naming

```typescript
// Good: Clear, RESTful routes
app.get('/users', listUsers);
app.get('/users/:id', getUser);
app.post('/users', createUser);
app.put('/users/:id', updateUser);
app.delete('/users/:id', deleteUser);

// Bad: Unclear routes
app.get('/getUsers', listUsers);
app.get('/user-detail/:id', getUser);
app.post('/newUser', createUser);
```

### 3. Middleware Reuse

```typescript
// Good: Reusable middleware
const authMiddleware = authenticate(jwtService);
const adminMiddleware = [authMiddleware, checkAdmin];

app.get('/profile', authMiddleware, getProfile);
app.get('/admin/users', ...adminMiddleware, getUsers);

// Bad: Duplicated middleware
app.get('/profile', authenticate(jwtService), getProfile);
app.get('/admin/users', authenticate(jwtService), checkAdmin, getUsers);
```

---

## See Also

- [Getting Started](../getting-started/quick-start.md)
- [Middleware](../middleware/overview.md)
- [Testing Strategies](testing.md)
- [Performance Optimization](../performance/optimization.md)
