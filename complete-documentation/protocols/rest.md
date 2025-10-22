# REST API

Build traditional RESTful APIs with RamAPI. This guide covers HTTP methods, routing, response formats, and best practices.

## Table of Contents

1. [Overview](#overview)
2. [HTTP Methods](#http-methods)
3. [Routing](#routing)
4. [Request/Response](#requestresponse)
5. [Status Codes](#status-codes)
6. [Best Practices](#best-practices)

---

## Overview

REST (Representational State Transfer) uses HTTP methods and URLs to represent resources.

### Basic Example

```typescript
import { createApp } from 'ramapi';

const app = createApp();

// GET /users - List all users
app.get('/users', async (ctx) => {
  const users = await db.users.findAll();
  ctx.json({ users });
});

// GET /users/:id - Get single user
app.get('/users/:id', async (ctx) => {
  const user = await db.users.findById(ctx.params.id);
  ctx.json({ user });
});

// POST /users - Create user
app.post('/users', async (ctx) => {
  const user = await db.users.create(ctx.body);
  ctx.json({ user }, 201);
});

// PUT /users/:id - Update user
app.put('/users/:id', async (ctx) => {
  const user = await db.users.update(ctx.params.id, ctx.body);
  ctx.json({ user });
});

// DELETE /users/:id - Delete user
app.delete('/users/:id', async (ctx) => {
  await db.users.delete(ctx.params.id);
  ctx.status(204);
});

app.listen(3000);
```

---

## HTTP Methods

### GET - Retrieve Resources

```typescript
// List collection
app.get('/posts', async (ctx) => {
  const posts = await db.posts.findAll();
  ctx.json({ posts });
});

// Get single resource
app.get('/posts/:id', async (ctx) => {
  const post = await db.posts.findById(ctx.params.id);
  if (!post) {
    throw new HTTPError(404, 'Post not found');
  }
  ctx.json({ post });
});

// Get with query parameters
app.get('/posts', async (ctx) => {
  const { page = 1, limit = 10, search } = ctx.query;
  const posts = await db.posts.find({ page, limit, search });
  ctx.json({ posts });
});
```

### POST - Create Resources

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

app.post('/posts',
  validate({
    body: z.object({
      title: z.string().min(1),
      content: z.string(),
    }),
  }),
  async (ctx) => {
    const post = await db.posts.create(ctx.body);
    ctx.json({ post }, 201); // 201 Created
  }
);
```

### PUT - Full Update

```typescript
app.put('/posts/:id',
  validate({
    body: z.object({
      title: z.string().min(1),
      content: z.string(),
    }),
  }),
  async (ctx) => {
    const post = await db.posts.update(ctx.params.id, ctx.body);
    ctx.json({ post });
  }
);
```

### PATCH - Partial Update

```typescript
app.patch('/posts/:id',
  validate({
    body: z.object({
      title: z.string().optional(),
      content: z.string().optional(),
    }),
  }),
  async (ctx) => {
    const post = await db.posts.update(ctx.params.id, ctx.body);
    ctx.json({ post });
  }
);
```

### DELETE - Remove Resources

```typescript
app.delete('/posts/:id', async (ctx) => {
  await db.posts.delete(ctx.params.id);
  ctx.status(204); // 204 No Content
});
```

---

## Routing

### Route Parameters

```typescript
// Single parameter
app.get('/users/:id', async (ctx) => {
  const { id } = ctx.params;
  ctx.json({ id });
});

// Multiple parameters
app.get('/users/:userId/posts/:postId', async (ctx) => {
  const { userId, postId } = ctx.params;
  ctx.json({ userId, postId });
});
```

### Query Parameters

```typescript
// GET /search?q=ramapi&page=1&limit=10
app.get('/search', async (ctx) => {
  const { q, page = '1', limit = '10' } = ctx.query;
  const results = await search(q, parseInt(page), parseInt(limit));
  ctx.json({ results });
});
```

### Route Groups

```typescript
// Group related routes
app.group('/api/v1', (router) => {
  router.get('/users', getUsersHandler);
  router.post('/users', createUserHandler);
  router.get('/users/:id', getUserHandler);
});
```

### Nested Routes

```typescript
// /api/users/:userId/posts
app.get('/api/users/:userId/posts', async (ctx) => {
  const posts = await db.posts.findByUserId(ctx.params.userId);
  ctx.json({ posts });
});
```

---

## Request/Response

### Request Body

```typescript
// Automatic JSON parsing
app.post('/users', async (ctx) => {
  const { name, email } = ctx.body;
  ctx.json({ name, email });
});
```

### Response Methods

```typescript
// JSON response
ctx.json({ message: 'Success' });

// JSON with status code
ctx.json({ error: 'Not found' }, 404);

// Text response
ctx.text('Hello, World!');

// Set status code
ctx.status(201);

// Set headers
ctx.setHeader('X-Custom-Header', 'value');

// Method chaining
ctx.status(201).setHeader('Location', '/users/123');
```

### Content Negotiation

```typescript
app.get('/data', async (ctx) => {
  const data = await getData();

  const accept = ctx.headers.accept;

  if (accept?.includes('application/json')) {
    ctx.json(data);
  } else if (accept?.includes('text/csv')) {
    ctx.setHeader('Content-Type', 'text/csv');
    ctx.text(toCSV(data));
  } else {
    ctx.json(data); // Default to JSON
  }
});
```

---

## Status Codes

### Success Codes

```typescript
// 200 OK - General success
ctx.json({ data }, 200);

// 201 Created - Resource created
ctx.json({ user }, 201);

// 204 No Content - Success with no response body
ctx.status(204);
```

### Client Error Codes

```typescript
// 400 Bad Request - Invalid input
throw new HTTPError(400, 'Invalid email format');

// 401 Unauthorized - Authentication required
throw new HTTPError(401, 'Authentication required');

// 403 Forbidden - No permission
throw new HTTPError(403, 'Insufficient permissions');

// 404 Not Found - Resource doesn't exist
throw new HTTPError(404, 'User not found');

// 409 Conflict - Resource conflict
throw new HTTPError(409, 'Email already exists');

// 422 Unprocessable Entity - Validation failed
throw new HTTPError(422, 'Validation failed', { errors });
```

### Server Error Codes

```typescript
// 500 Internal Server Error - Generic server error
throw new HTTPError(500, 'Database connection failed');

// 503 Service Unavailable - Service temporarily unavailable
throw new HTTPError(503, 'Service under maintenance');
```

---

## Best Practices

### 1. Use Plural Nouns for Collections

```typescript
// Good
app.get('/users', listUsers);
app.get('/posts', listPosts);

// Avoid
app.get('/user', listUsers);
app.get('/post', listPosts);
```

### 2. Use HTTP Methods Correctly

```typescript
// Good
app.get('/users', listUsers);      // Read
app.post('/users', createUser);    // Create
app.put('/users/:id', updateUser); // Full update
app.patch('/users/:id', patchUser); // Partial update
app.delete('/users/:id', deleteUser); // Delete

// Avoid
app.post('/users/get', listUsers); // Don't use POST for reads
app.get('/users/delete', deleteUser); // Don't use GET for deletes
```

### 3. Version Your API

```typescript
// Version prefix
app.group('/api/v1', (router) => {
  router.get('/users', getUsersV1);
});

app.group('/api/v2', (router) => {
  router.get('/users', getUsersV2);
});
```

### 4. Return Consistent Error Format

```typescript
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    const err = error as HTTPError;
    ctx.json({
      error: true,
      message: err.message,
      code: err.statusCode,
      details: err.details,
    }, err.statusCode || 500);
  }
});
```

### 5. Use Validation

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

app.post('/users',
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(8),
    }),
  }),
  createUserHandler
);
```

### 6. Implement Pagination

```typescript
app.get('/posts', async (ctx) => {
  const page = parseInt(ctx.query.page as string) || 1;
  const limit = parseInt(ctx.query.limit as string) || 10;

  const posts = await db.posts.paginate(page, limit);
  const total = await db.posts.count();

  ctx.json({
    data: posts,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit),
    },
  });
});
```

### 7. Filter and Sort

```typescript
app.get('/posts', async (ctx) => {
  const { sort = 'createdAt', order = 'desc', status } = ctx.query;

  const posts = await db.posts.find({
    sort,
    order,
    ...(status && { status }),
  });

  ctx.json({ posts });
});
```

### 8. Use HATEOAS (Optional)

```typescript
app.get('/users/:id', async (ctx) => {
  const user = await db.users.findById(ctx.params.id);

  ctx.json({
    data: user,
    links: {
      self: `/users/${user.id}`,
      posts: `/users/${user.id}/posts`,
      comments: `/users/${user.id}/comments`,
    },
  });
});
```

---

## Complete Example

```typescript
import { createApp, validate, authenticate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

// Schemas
const createPostSchema = {
  body: z.object({
    title: z.string().min(1).max(200),
    content: z.string().min(1),
    tags: z.array(z.string()).optional(),
  }),
};

const updatePostSchema = {
  body: z.object({
    title: z.string().min(1).max(200).optional(),
    content: z.string().min(1).optional(),
    tags: z.array(z.string()).optional(),
  }),
};

// Routes
app.group('/api/posts', (router) => {
  // List posts with pagination
  router.get('/', async (ctx) => {
    const page = parseInt(ctx.query.page as string) || 1;
    const limit = parseInt(ctx.query.limit as string) || 10;

    const posts = await db.posts.paginate(page, limit);
    const total = await db.posts.count();

    ctx.json({
      posts,
      pagination: { page, limit, total },
    });
  });

  // Get single post
  router.get('/:id', async (ctx) => {
    const post = await db.posts.findById(ctx.params.id);
    if (!post) {
      throw new HTTPError(404, 'Post not found');
    }
    ctx.json({ post });
  });

  // Create post (authenticated)
  router.post('/',
    authenticate(jwtService),
    validate(createPostSchema),
    async (ctx) => {
      const post = await db.posts.create({
        ...ctx.body,
        authorId: ctx.user.sub,
      });
      ctx.json({ post }, 201);
    }
  );

  // Update post (authenticated)
  router.patch('/:id',
    authenticate(jwtService),
    validate(updatePostSchema),
    async (ctx) => {
      const post = await db.posts.update(ctx.params.id, ctx.body);
      ctx.json({ post });
    }
  );

  // Delete post (authenticated)
  router.delete('/:id',
    authenticate(jwtService),
    async (ctx) => {
      await db.posts.delete(ctx.params.id);
      ctx.status(204);
    }
  );
});

app.listen(3000);
```

---

## Next Steps

- [GraphQL](graphql.md)
- [gRPC](grpc.md)
- [Multi-Protocol Overview](overview.md)

---

**Need help?** Check the [API Reference](../api-reference/server.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
