# Routing

RamAPI's routing system is designed for performance and developer experience. It provides an intuitive API for defining routes, supports dynamic parameters, and offers ultra-fast route matching with O(1) lookup for static routes.

## Table of Contents

1. [Basic Routing](#basic-routing)
2. [HTTP Methods](#http-methods)
3. [Route Parameters](#route-parameters)
4. [Route Groups](#route-groups)
5. [Route Middleware](#route-middleware)
6. [Route Organization](#route-organization)
7. [Performance Optimizations](#performance-optimizations)
8. [Best Practices](#best-practices)

---

## Basic Routing

Routes are defined using HTTP method helpers on the app instance.

### Simple Route

```typescript
import { createApp } from 'ramapi';

const app = createApp();

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello, World!' });
});

app.listen(3000);
```

### Multiple Routes

```typescript
app.get('/', async (ctx) => {
  ctx.json({ message: 'Home' });
});

app.get('/about', async (ctx) => {
  ctx.json({ message: 'About' });
});

app.get('/contact', async (ctx) => {
  ctx.json({ message: 'Contact' });
});
```

---

## HTTP Methods

RamAPI supports all standard HTTP methods.

### GET

Retrieve data.

```typescript
app.get('/users', async (ctx) => {
  const users = await database.getUsers();
  ctx.json({ users });
});
```

### POST

Create new resources.

```typescript
app.post('/users', async (ctx) => {
  const user = await database.createUser(ctx.body);
  ctx.json({ user }, 201);
});
```

### PUT

Update entire resources (replace).

```typescript
app.put('/users/:id', async (ctx) => {
  const user = await database.replaceUser(ctx.params.id, ctx.body);
  ctx.json({ user });
});
```

### PATCH

Partially update resources.

```typescript
app.patch('/users/:id', async (ctx) => {
  const user = await database.updateUser(ctx.params.id, ctx.body);
  ctx.json({ user });
});
```

### DELETE

Remove resources.

```typescript
app.delete('/users/:id', async (ctx) => {
  await database.deleteUser(ctx.params.id);
  ctx.json({ message: 'User deleted' });
});
```

### OPTIONS

Handle preflight requests (usually handled by CORS middleware).

```typescript
app.options('/users', async (ctx) => {
  ctx.status(204).res.end();
});
```

### HEAD

Return headers without body (like GET but no response body).

```typescript
app.head('/users/:id', async (ctx) => {
  const exists = await database.userExists(ctx.params.id);
  ctx.status(exists ? 200 : 404).res.end();
});
```

### ALL

Handle all HTTP methods for a route.

```typescript
app.all('/debug', async (ctx) => {
  ctx.json({
    method: ctx.method,
    path: ctx.path,
    message: 'Handled by ALL'
  });
});
```

---

## Route Parameters

Define dynamic segments in your routes using the `:` prefix.

### Single Parameter

```typescript
app.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;

  const user = await database.findUser(userId);
  ctx.json({ user });
});
```

**Test it:**
```bash
curl http://localhost:3000/users/123
# ctx.params.id = '123'
```

### Multiple Parameters

```typescript
app.get('/users/:userId/posts/:postId', async (ctx) => {
  const { userId, postId } = ctx.params;

  const post = await database.findPost(userId, postId);
  ctx.json({ post });
});
```

**Test it:**
```bash
curl http://localhost:3000/users/123/posts/456
# ctx.params.userId = '123'
# ctx.params.postId = '456'
```

### Parameter Validation

Validate route parameters using Zod schemas:

```typescript
import { z } from 'zod';
import { validate } from 'ramapi';

const paramsSchema = z.object({
  id: z.string().uuid('Invalid user ID format'),
});

app.get('/users/:id',
  validate({ params: paramsSchema }),
  async (ctx) => {
    // ctx.params.id is now validated as a UUID
    const { id } = ctx.params as z.infer<typeof paramsSchema>;

    const user = await database.findUser(id);
    ctx.json({ user });
  }
);
```

### Parameter Types

Route parameters are always strings. Convert them as needed:

```typescript
app.get('/posts/:id', async (ctx) => {
  // Parameter is a string
  const idString = ctx.params.id;

  // Convert to number
  const id = parseInt(idString, 10);

  if (isNaN(id)) {
    throw new HTTPError(400, 'Invalid ID');
  }

  const post = await database.findPost(id);
  ctx.json({ post });
});
```

**Better approach with validation:**

```typescript
const paramsSchema = z.object({
  id: z.string().regex(/^\d+$/).transform(Number),
});

app.get('/posts/:id',
  validate({ params: paramsSchema }),
  async (ctx) => {
    // id is now a number
    const { id } = ctx.params as z.infer<typeof paramsSchema>;
    const post = await database.findPost(id);
    ctx.json({ post });
  }
);
```

---

## Route Groups

Organize routes with shared prefixes and middleware.

### Basic Group

```typescript
app.group('/api', (api) => {
  api.get('/users', listUsers);
  api.get('/posts', listPosts);
});

// Creates routes:
// GET /api/users
// GET /api/posts
```

### Nested Groups

```typescript
app.group('/api', (api) => {
  api.group('/v1', (v1) => {
    v1.get('/users', listUsers);
    v1.get('/posts', listPosts);
  });

  api.group('/v2', (v2) => {
    v2.get('/users', listUsersV2);
    v2.get('/posts', listPostsV2);
  });
});

// Creates routes:
// GET /api/v1/users
// GET /api/v1/posts
// GET /api/v2/users
// GET /api/v2/posts
```

### Groups with Middleware

Apply middleware to all routes in a group:

```typescript
import { authenticate, logger } from 'ramapi';

const jwt = new JWTService({ secret: 'secret' });

app.group('/api', (api) => {
  // Public routes
  api.post('/login', loginHandler);
  api.post('/register', registerHandler);

  // Protected routes
  api.group('/users', (users) => {
    // Apply auth middleware to all user routes
    users.use(authenticate(jwt));

    users.get('/', listUsers);           // GET /api/users
    users.get('/:id', getUser);          // GET /api/users/:id
    users.post('/', createUser);         // POST /api/users
    users.put('/:id', updateUser);       // PUT /api/users/:id
    users.delete('/:id', deleteUser);    // DELETE /api/users/:id
  });
});
```

### Multiple Middleware in Groups

```typescript
import { authenticate, rateLimit, logger } from 'ramapi';

app.group('/api/admin', (admin) => {
  // Apply multiple middleware
  admin.use(logger());
  admin.use(authenticate(jwt));
  admin.use(rateLimit({ maxRequests: 10, windowMs: 60000 }));

  admin.get('/users', listAllUsers);
  admin.delete('/users/:id', deleteAnyUser);
  admin.post('/settings', updateSettings);
});
```

---

## Route Middleware

Apply middleware to specific routes.

### Single Route Middleware

```typescript
import { validate } from 'ramapi';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

app.post('/users',
  validate({ body: userSchema }),
  async (ctx) => {
    // Handler runs after validation
    const user = await createUser(ctx.body);
    ctx.json({ user }, 201);
  }
);
```

### Multiple Route Middleware

Middleware runs in the order specified:

```typescript
const checkAuth = authenticate(jwt);
const checkAdmin = async (ctx, next) => {
  if (ctx.user.role !== 'admin') {
    throw new HTTPError(403, 'Admin access required');
  }
  await next();
};

app.delete('/users/:id',
  checkAuth,      // First: authenticate
  checkAdmin,     // Second: check admin role
  async (ctx) => {
    // Third: handle the request
    await deleteUser(ctx.params.id);
    ctx.json({ message: 'User deleted' });
  }
);
```

### Middleware Execution Order

```typescript
// Global middleware (runs for all routes)
app.use(logger());
app.use(cors());

// Group middleware (runs for all routes in group)
app.group('/api', (api) => {
  api.use(rateLimit({ maxRequests: 100, windowMs: 60000 }));

  // Route middleware (runs for specific route)
  api.post('/users',
    validate({ body: userSchema }),
    createUserHandler
  );
});

// Execution order:
// 1. logger() - global
// 2. cors() - global
// 3. rateLimit() - group
// 4. validate() - route
// 5. createUserHandler - handler
```

---

## Route Organization

Organize routes for maintainability.

### Separate Route Files

**routes/users.ts**
```typescript
import { Router } from 'ramapi';

export function registerUserRoutes(router: Router) {
  router.get('/users', async (ctx) => {
    ctx.json({ users: [] });
  });

  router.get('/users/:id', async (ctx) => {
    ctx.json({ user: { id: ctx.params.id } });
  });

  router.post('/users', async (ctx) => {
    ctx.json({ message: 'User created' }, 201);
  });
}
```

**routes/posts.ts**
```typescript
import { Router } from 'ramapi';

export function registerPostRoutes(router: Router) {
  router.get('/posts', async (ctx) => {
    ctx.json({ posts: [] });
  });

  router.post('/posts', async (ctx) => {
    ctx.json({ message: 'Post created' }, 201);
  });
}
```

**index.ts**
```typescript
import { createApp } from 'ramapi';
import { registerUserRoutes } from './routes/users.js';
import { registerPostRoutes } from './routes/posts.js';

const app = createApp();

// Register route modules
registerUserRoutes(app.getRouter());
registerPostRoutes(app.getRouter());

app.listen(3000);
```

### Controller Pattern

**controllers/UserController.ts**
```typescript
import { Context } from 'ramapi';

export class UserController {
  async list(ctx: Context) {
    const users = await database.getUsers();
    ctx.json({ users });
  }

  async get(ctx: Context) {
    const user = await database.getUser(ctx.params.id);
    ctx.json({ user });
  }

  async create(ctx: Context) {
    const user = await database.createUser(ctx.body);
    ctx.json({ user }, 201);
  }

  async update(ctx: Context) {
    const user = await database.updateUser(ctx.params.id, ctx.body);
    ctx.json({ user });
  }

  async delete(ctx: Context) {
    await database.deleteUser(ctx.params.id);
    ctx.json({ message: 'User deleted' });
  }
}
```

**routes/users.ts**
```typescript
import { Router } from 'ramapi';
import { UserController } from '../controllers/UserController.js';

const controller = new UserController();

export function registerUserRoutes(router: Router) {
  router.get('/users', controller.list);
  router.get('/users/:id', controller.get);
  router.post('/users', controller.create);
  router.put('/users/:id', controller.update);
  router.delete('/users/:id', controller.delete);
}
```

### Modular API Versions

```typescript
// api/v1/index.ts
export function registerV1Routes(app: Server) {
  app.group('/api/v1', (v1) => {
    registerUserRoutes(v1);
    registerPostRoutes(v1);
  });
}

// api/v2/index.ts
export function registerV2Routes(app: Server) {
  app.group('/api/v2', (v2) => {
    registerUserRoutesV2(v2);
    registerPostRoutesV2(v2);
  });
}

// index.ts
import { createApp } from 'ramapi';
import { registerV1Routes } from './api/v1/index.js';
import { registerV2Routes } from './api/v2/index.js';

const app = createApp();

registerV1Routes(app);
registerV2Routes(app);

app.listen(3000);
```

---

## Performance Optimizations

RamAPI's router is optimized for speed.

### O(1) Static Route Lookup

Static routes (no parameters) use a hash map for instant lookup:

```typescript
// These routes have O(1) lookup time
app.get('/users', handler);
app.get('/posts', handler);
app.get('/api/v1/health', handler);
```

### Pre-compiled Route Patterns

Dynamic routes are pre-compiled at registration time:

```typescript
// Pattern compiled once during registration
app.get('/users/:id/posts/:postId', handler);

// At request time: Fast pattern matching (no regex)
```

### Route Caching

The last matched route is cached for hot path optimization:

```typescript
// First request: Full lookup
GET /users/123 -> finds route

// Subsequent requests to same route: Instant cache hit
GET /users/456 -> cache hit (same route pattern)
```

### Middleware Pre-compilation

Middleware chains are compiled into a single handler at registration time:

```typescript
// Compiled once at registration
app.get('/users',
  middleware1,
  middleware2,
  middleware3,
  handler
);

// At request time: Single function call (zero overhead)
```

---

## Best Practices

### 1. Use Specific Routes Before Generic

```typescript
// Order matters - specific routes first
app.get('/users/me', getCurrentUser);
app.get('/users/:id', getUser);

// Not recommended - generic route shadows specific
app.get('/users/:id', getUser);
app.get('/users/me', getCurrentUser); // Never matches!
```

### 2. Validate Route Parameters

```typescript
import { z } from 'zod';
import { validate } from 'ramapi';

const paramsSchema = z.object({
  id: z.string().uuid(),
});

app.get('/users/:id',
  validate({ params: paramsSchema }),
  async (ctx) => {
    // Parameters are validated
    const { id } = ctx.params as z.infer<typeof paramsSchema>;
  }
);
```

### 3. Use Route Groups for Organization

```typescript
// Good - organized with groups
app.group('/api/v1', (api) => {
  api.group('/users', registerUserRoutes);
  api.group('/posts', registerPostRoutes);
  api.group('/comments', registerCommentRoutes);
});

// Not recommended - flat structure
app.get('/api/v1/users', handler1);
app.get('/api/v1/users/:id', handler2);
app.get('/api/v1/posts', handler3);
// ... many more routes
```

### 4. Apply Middleware at the Right Level

```typescript
// Apply at the right level for efficiency
app.use(logger());  // All routes

app.group('/api', (api) => {
  api.use(cors());  // All API routes

  api.group('/admin', (admin) => {
    admin.use(authenticate(jwt));  // Only admin routes
    admin.use(checkAdmin);
  });
});
```

### 5. Return After Sending Response

```typescript
app.get('/users/:id', async (ctx) => {
  const user = await findUser(ctx.params.id);

  if (!user) {
    ctx.status(404).json({ error: 'User not found' });
    return; // Important: stop execution
  }

  // This won't execute if user not found
  ctx.json({ user });
});
```

### 6. Use Consistent Naming Conventions

```typescript
// RESTful conventions
app.get('/users', listUsers);           // List all
app.get('/users/:id', getUser);         // Get one
app.post('/users', createUser);         // Create
app.put('/users/:id', updateUser);      // Update (full)
app.patch('/users/:id', patchUser);     // Update (partial)
app.delete('/users/:id', deleteUser);   // Delete
```

### 7. Separate Route Definition from Logic

```typescript
// Don't do this - logic in route
app.get('/users/:id', async (ctx) => {
  const user = await db.users.findOne({ id: ctx.params.id });
  if (!user) throw new HTTPError(404, 'Not found');
  const posts = await db.posts.find({ userId: user.id });
  // ... lots more logic
  ctx.json({ user, posts });
});

// Do this - separate concerns
app.get('/users/:id', async (ctx) => {
  const userService = new UserService();
  const data = await userService.getUserWithPosts(ctx.params.id);
  ctx.json(data);
});
```

---

## Common Patterns

### Resource Routes

```typescript
// Standard CRUD routes for a resource
app.group('/api/users', (users) => {
  users.get('/', listUsers);           // GET /api/users
  users.get('/:id', getUser);          // GET /api/users/:id
  users.post('/', createUser);         // POST /api/users
  users.put('/:id', updateUser);       // PUT /api/users/:id
  users.patch('/:id', patchUser);      // PATCH /api/users/:id
  users.delete('/:id', deleteUser);    // DELETE /api/users/:id
});
```

### Nested Resources

```typescript
app.group('/users/:userId', (user) => {
  user.get('/posts', getUserPosts);           // GET /users/:userId/posts
  user.post('/posts', createUserPost);        // POST /users/:userId/posts
  user.get('/posts/:postId', getUserPost);    // GET /users/:userId/posts/:postId
  user.delete('/posts/:postId', deletePost);  // DELETE /users/:userId/posts/:postId
});
```

### Health Check Routes

```typescript
app.get('/health', async (ctx) => {
  ctx.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

app.get('/health/db', async (ctx) => {
  const isHealthy = await database.ping();
  ctx.json({ status: isHealthy ? 'ok' : 'error' });
});
```

---

## Next Steps

- [Learn about Middleware](middleware.md)
- [Understand Validation](validation.md)
- [Explore Error Handling](error-handling.md)
- [See Complete Examples](../examples/todo-api.md)

---

**Need help?** Check the [Router API Reference](../api-reference/router.md) for complete documentation.
