# Context & Handlers

Understanding the Context object and Handler functions is fundamental to building applications with RamAPI. This guide covers everything you need to know about working with request contexts and writing handlers.

## Table of Contents

1. [Context Object](#context-object)
2. [Handler Functions](#handler-functions)
3. [Request Properties](#request-properties)
4. [Response Helpers](#response-helpers)
5. [State Management](#state-management)
6. [Observability Helpers](#observability-helpers)
7. [Best Practices](#best-practices)

---

## Context Object

The **Context** object is passed to every handler and middleware function. It encapsulates the HTTP request and response, providing a clean API for accessing request data and sending responses.

### Basic Structure

```typescript
interface Context<TBody = unknown, TQuery = unknown, TParams = unknown> {
  // Request properties
  req: IncomingMessage;
  res: ServerResponse;
  method: HTTPMethod;
  url: string;
  path: string;
  query: TQuery;
  params: TParams;
  body: TBody;
  headers: Record<string, string | string[] | undefined>;

  // Response helpers
  json: (data: unknown, status?: number) => void;
  text: (data: string, status?: number) => void;
  status: (code: number) => Context;
  setHeader: (key: string, value: string) => Context;

  // Shared state
  state: Record<string, unknown>;
  user?: unknown;

  // Observability
  trace?: TraceContext;
  startSpan?: (name: string, attributes?: Record<string, any>) => Span;
  endSpan?: (span: Span, error?: Error) => void;
  addEvent?: (name: string, attributes?: Record<string, any>) => void;
  setAttributes?: (attributes: Record<string, any>) => void;
}
```

---

## Handler Functions

Handlers are functions that process HTTP requests. They receive a Context object and can be synchronous or asynchronous.

### Handler Signature

```typescript
type Handler<TBody = unknown, TQuery = unknown, TParams = unknown> = (
  ctx: Context<TBody, TQuery, TParams>
) => void | Promise<void>;
```

### Basic Handler

```typescript
import { createApp } from 'ramapi';

const app = createApp();

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello, World!' });
});
```

### Async Handler

```typescript
app.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;

  // Async database call
  const user = await database.findUser(userId);

  ctx.json(user);
});
```

### Handler with Type Safety

```typescript
import { z } from 'zod';
import { validate } from 'ramapi';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number(),
});

app.post('/users',
  validate({ body: userSchema }),
  async (ctx) => {
    // ctx.body is now typed as { name: string; email: string; age: number }
    const user = ctx.body as z.infer<typeof userSchema>;

    await database.createUser(user);

    ctx.json({ message: 'User created', user }, 201);
  }
);
```

---

## Request Properties

### Method

The HTTP method of the request.

```typescript
app.all('/debug', async (ctx) => {
  console.log(ctx.method); // 'GET', 'POST', 'PUT', etc.

  ctx.json({ method: ctx.method });
});
```

### URL and Path

- `ctx.url` - Full URL including query string
- `ctx.path` - Path without query string

```typescript
app.get('/search', async (ctx) => {
  console.log(ctx.url);  // '/search?q=typescript&page=2'
  console.log(ctx.path); // '/search'

  ctx.json({ url: ctx.url, path: ctx.path });
});
```

### Query Parameters

Query string parameters, automatically parsed.

```typescript
app.get('/search', async (ctx) => {
  // URL: /search?q=typescript&page=2&limit=10
  const { q, page, limit } = ctx.query;

  console.log(q);     // 'typescript'
  console.log(page);  // '2' (string)
  console.log(limit); // '10' (string)

  // Convert to numbers if needed
  const pageNum = parseInt(page as string);
  const limitNum = parseInt(limit as string);

  ctx.json({
    query: q,
    page: pageNum,
    limit: limitNum
  });
});
```

**Note**: Query parameters are always strings. Use validation middleware to parse and validate them:

```typescript
const querySchema = z.object({
  page: z.string().regex(/^\d+$/).transform(Number).default('1'),
  limit: z.string().regex(/^\d+$/).transform(Number).default('10'),
});

app.get('/items',
  validate({ query: querySchema }),
  async (ctx) => {
    const { page, limit } = ctx.query as z.infer<typeof querySchema>;
    // page and limit are now numbers
  }
);
```

### Route Parameters

Dynamic route parameters from the URL path.

```typescript
app.get('/users/:userId/posts/:postId', async (ctx) => {
  const { userId, postId } = ctx.params;

  console.log(userId); // '123'
  console.log(postId); // '456'

  ctx.json({ userId, postId });
});
```

### Request Body

The parsed request body (available for POST, PUT, PATCH).

```typescript
app.post('/users', async (ctx) => {
  const userData = ctx.body;

  console.log(userData);
  // { name: 'John', email: 'john@example.com' }

  ctx.json({ received: userData });
});
```

**Body Parsing**: RamAPI automatically parses:
- `application/json` - Parsed as JSON
- `application/x-www-form-urlencoded` - Parsed as form data
- Other types - Available as raw string

### Headers

Access request headers.

```typescript
app.get('/info', async (ctx) => {
  const userAgent = ctx.headers['user-agent'];
  const contentType = ctx.headers['content-type'];
  const authorization = ctx.headers['authorization'];

  ctx.json({
    userAgent,
    contentType,
    authorization
  });
});
```

### Raw Request/Response

Access underlying Node.js objects when needed.

```typescript
app.get('/raw', async (ctx) => {
  // Access IncomingMessage
  console.log(ctx.req.socket.remoteAddress);

  // Access ServerResponse
  ctx.res.setHeader('X-Custom-Header', 'value');

  ctx.json({ message: 'Using raw req/res' });
});
```

---

## Response Helpers

### JSON Response

Send JSON response with optional status code.

```typescript
app.get('/users/:id', async (ctx) => {
  const user = { id: ctx.params.id, name: 'John' };

  // Default 200 status
  ctx.json(user);

  // Or specify status
  ctx.json(user, 200);
});
```

### Text Response

Send plain text response.

```typescript
app.get('/health', async (ctx) => {
  ctx.text('OK');

  // With status code
  ctx.text('Service Unavailable', 503);
});
```

### Status Code

Set response status code (chainable).

```typescript
app.delete('/users/:id', async (ctx) => {
  await deleteUser(ctx.params.id);

  // Method 1: Chain with json()
  ctx.status(204).json({ message: 'Deleted' });

  // Method 2: Set separately
  ctx.status(204);
  ctx.res.end();
});
```

### Set Headers

Set response headers (chainable).

```typescript
app.get('/download', async (ctx) => {
  ctx
    .setHeader('Content-Disposition', 'attachment; filename="data.json"')
    .setHeader('X-Custom-Header', 'custom-value')
    .json({ data: 'value' });
});
```

### Multiple Response Types

```typescript
app.get('/data', async (ctx) => {
  const accept = ctx.headers['accept'];

  const data = { message: 'Hello' };

  if (accept?.includes('application/json')) {
    ctx.json(data);
  } else if (accept?.includes('text/plain')) {
    ctx.text(JSON.stringify(data));
  } else {
    ctx.json(data); // Default to JSON
  }
});
```

---

## State Management

The `ctx.state` object allows middleware to share data with handlers and other middleware.

### Basic State Usage

```typescript
// Middleware sets state
app.use(async (ctx, next) => {
  ctx.state.requestId = generateRequestId();
  ctx.state.startTime = Date.now();

  await next();

  const duration = Date.now() - ctx.state.startTime;
  console.log(`Request ${ctx.state.requestId} took ${duration}ms`);
});

// Handler accesses state
app.get('/data', async (ctx) => {
  const requestId = ctx.state.requestId;

  ctx.json({ requestId, data: 'value' });
});
```

### User Context

The `ctx.user` property is set by authentication middleware.

```typescript
import { JWTService, authenticate } from 'ramapi';

const jwt = new JWTService({ secret: 'secret' });

app.get('/profile',
  authenticate(jwt),
  async (ctx) => {
    // ctx.user contains decoded JWT payload
    console.log(ctx.user);
    // { sub: '123', email: 'user@example.com' }

    // Also available in ctx.state
    const userId = ctx.state.userId;

    ctx.json({ user: ctx.user });
  }
);
```

### Typed State

Define types for better type safety:

```typescript
interface AppState {
  requestId: string;
  userId?: string;
  startTime: number;
}

// In your handler
app.get('/data', async (ctx) => {
  const state = ctx.state as AppState;

  ctx.json({
    requestId: state.requestId,
    userId: state.userId
  });
});
```

---

## Observability Helpers

Context provides helpers for distributed tracing and observability.

### Start Span

Create a span to track an operation:

```typescript
app.get('/users/:id', async (ctx) => {
  const span = ctx.startSpan?.('fetchUser', {
    userId: ctx.params.id
  });

  try {
    const user = await database.findUser(ctx.params.id);
    ctx.json(user);
  } catch (error) {
    ctx.endSpan?.(span, error as Error);
    throw error;
  }

  ctx.endSpan?.(span);
});
```

### Add Event

Add an event to the current trace:

```typescript
app.post('/orders', async (ctx) => {
  ctx.addEvent?.('orderValidation', {
    itemCount: ctx.body.items.length
  });

  // Process order...

  ctx.addEvent?.('orderProcessed', {
    orderId: order.id,
    total: order.total
  });

  ctx.json({ order });
});
```

### Set Attributes

Add attributes to the current span:

```typescript
app.get('/search', async (ctx) => {
  ctx.setAttributes?.({
    'search.query': ctx.query.q,
    'search.resultsCount': results.length
  });

  ctx.json({ results });
});
```

**Note**: Observability helpers are only available when tracing is enabled in server configuration.

---

## Best Practices

### 1. Always Handle Errors

```typescript
app.get('/users/:id', async (ctx) => {
  try {
    const user = await database.findUser(ctx.params.id);

    if (!user) {
      throw new HTTPError(404, 'User not found');
    }

    ctx.json(user);
  } catch (error) {
    // Error will be caught by error handler
    throw error;
  }
});
```

### 2. Validate Input

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

app.post('/register',
  validate({ body: schema }),
  async (ctx) => {
    // Input is now validated
    const { email, password } = ctx.body as z.infer<typeof schema>;
  }
);
```

### 3. Use Type Inference

```typescript
// Define schema
const userSchema = z.object({
  name: z.string(),
  age: z.number(),
});

type User = z.infer<typeof userSchema>;

// Use typed handler
app.post('/users',
  validate({ body: userSchema }),
  async (ctx) => {
    const user = ctx.body as User;
    // TypeScript knows user structure
  }
);
```

### 4. Separate Business Logic

```typescript
// Don't do this
app.get('/users/:id', async (ctx) => {
  const user = await db.users.findOne({ id: ctx.params.id });
  const posts = await db.posts.find({ userId: user.id });
  const comments = await db.comments.find({ userId: user.id });
  // ... lots of logic
  ctx.json({ user, posts, comments });
});

// Do this
app.get('/users/:id', async (ctx) => {
  const userService = new UserService();
  const data = await userService.getUserWithDetails(ctx.params.id);
  ctx.json(data);
});
```

### 5. Set Appropriate Status Codes

```typescript
// Create - 201
app.post('/users', async (ctx) => {
  const user = await createUser(ctx.body);
  ctx.json(user, 201);
});

// Delete - 204
app.delete('/users/:id', async (ctx) => {
  await deleteUser(ctx.params.id);
  ctx.status(204).res.end();
});

// Not Modified - 304
app.get('/data', async (ctx) => {
  const etag = ctx.headers['if-none-match'];
  if (etag === currentEtag) {
    ctx.status(304).res.end();
    return;
  }
  ctx.setHeader('ETag', currentEtag).json(data);
});
```

### 6. Don't Modify Context Structure

```typescript
// Don't do this
app.use(async (ctx: any, next) => {
  ctx.myCustomProperty = 'value'; // Bad!
  await next();
});

// Do this
app.use(async (ctx, next) => {
  ctx.state.myCustomProperty = 'value'; // Good!
  await next();
});
```

### 7. Handle Response Timing

```typescript
app.get('/data', async (ctx) => {
  // Only send response once
  if (someCondition) {
    ctx.json({ data: 'value' });
    return; // Important: return after sending response
  }

  // This won't execute if condition is true
  ctx.json({ data: 'other' });
});
```

---

## Common Patterns

### Redirect

```typescript
app.get('/old-path', async (ctx) => {
  ctx.status(301).setHeader('Location', '/new-path');
  ctx.res.end();
});
```

### Conditional Response

```typescript
app.get('/users/:id', async (ctx) => {
  const user = await findUser(ctx.params.id);

  if (!user) {
    ctx.status(404).json({ error: 'User not found' });
    return;
  }

  ctx.json(user);
});
```

### File Download

```typescript
app.get('/download/:filename', async (ctx) => {
  const filename = ctx.params.filename;

  ctx
    .setHeader('Content-Disposition', `attachment; filename="${filename}"`)
    .setHeader('Content-Type', 'application/octet-stream')
    .text(fileContent);
});
```

### Cache Headers

```typescript
app.get('/static/data', async (ctx) => {
  ctx
    .setHeader('Cache-Control', 'public, max-age=3600')
    .setHeader('ETag', generateETag(data))
    .json(data);
});
```

---

## Next Steps

- [Learn about Routing](routing.md)
- [Understand Middleware](middleware.md)
- [Explore Validation](validation.md)
- [See Complete Examples](../examples/todo-api.md)

---

**Need help?** Check the [API Reference](../api-reference/context.md) for complete Context documentation.
