# Context API Reference

Complete API reference for the RamAPI Context object passed to handlers and middleware.

## Table of Contents

1. [Context Interface](#context-interface)
2. [Request Properties](#request-properties)
3. [Response Methods](#response-methods)
4. [State Management](#state-management)
5. [Observability Helpers](#observability-helpers)
6. [Type Safety](#type-safety)
7. [Advanced Usage](#advanced-usage)

---

## Context Interface

The Context object is the primary interface for handling requests and responses in RamAPI.

### Type Signature

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
  status: (code: number) => Context<TBody, TQuery, TParams>;
  setHeader: (key: string, value: string) => Context<TBody, TQuery, TParams>;

  // Shared state
  state: Record<string, unknown>;

  // User/auth context
  user?: unknown;

  // Observability
  trace?: TraceContext;
  startSpan?: (name: string, attributes?: Record<string, any>) => Span | undefined;
  endSpan?: (span: Span | undefined, error?: Error) => void;
  addEvent?: (name: string, attributes?: Record<string, any>) => void;
  setAttributes?: (attributes: Record<string, any>) => void;
}
```

---

## Request Properties

### req

Raw Node.js IncomingMessage object.

- **Type:** `IncomingMessage`
- **Read-only:** Yes
- **Description:** The underlying Node.js request object

```typescript
app.get('/example', async (ctx) => {
  console.log(ctx.req.socket.remoteAddress); // Client IP
  console.log(ctx.req.httpVersion); // HTTP version
});
```

---

### res

Raw Node.js ServerResponse object.

- **Type:** `ServerResponse`
- **Read-only:** Yes
- **Description:** The underlying Node.js response object

```typescript
app.get('/example', async (ctx) => {
  // Use ctx.res for advanced features
  ctx.res.setHeader('X-Custom', 'value');
  ctx.res.write('streaming data');
  ctx.res.end();
});
```

---

### method

HTTP method of the request.

- **Type:** `HTTPMethod`
- **Read-only:** Yes
- **Values:** `'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE' | 'OPTIONS' | 'HEAD'`

```typescript
app.all('/example', async (ctx) => {
  if (ctx.method === 'GET') {
    ctx.json({ message: 'GET request' });
  } else if (ctx.method === 'POST') {
    ctx.json({ message: 'POST request' });
  }
});
```

---

### url

Full request URL including query string.

- **Type:** `string`
- **Read-only:** Yes
- **Example:** `'/users?page=2&limit=10'`

```typescript
app.get('/users', async (ctx) => {
  console.log(ctx.url); // '/users?page=2&limit=10'
});
```

---

### path

Request pathname without query string.

- **Type:** `string`
- **Read-only:** Yes
- **Example:** `'/users'`

```typescript
app.get('/users', async (ctx) => {
  console.log(ctx.path); // '/users'
});
```

---

### query

Parsed query string parameters.

- **Type:** `TQuery` (default: `unknown`)
- **Read-only:** No
- **Lazy-loaded:** Yes (parsed on first access)

```typescript
app.get('/users', async (ctx) => {
  console.log(ctx.query);
  // GET /users?page=2&limit=10
  // { page: '2', limit: '10' }
});

// With validation
import { validate } from 'ramapi';
import { z } from 'zod';

app.get('/users',
  validate({
    query: z.object({
      page: z.string().transform(Number),
      limit: z.string().transform(Number),
    }),
  }),
  async (ctx) => {
    console.log(ctx.query);
    // { page: 2, limit: 10 } (typed as numbers)
  }
);
```

---

### params

Extracted route parameters.

- **Type:** `TParams` (default: `unknown`)
- **Read-only:** No
- **Description:** Dynamic segments from the route path

```typescript
app.get('/users/:userId/posts/:postId', async (ctx) => {
  console.log(ctx.params);
  // GET /users/123/posts/456
  // { userId: '123', postId: '456' }

  const userId = parseInt(ctx.params.userId);
  const postId = parseInt(ctx.params.postId);
});

// With validation
app.get('/users/:id',
  validate({
    params: z.object({
      id: z.string().regex(/^\d+$/).transform(Number),
    }),
  }),
  async (ctx) => {
    const userId: number = ctx.params.id; // Type-safe number
  }
);
```

---

### body

Parsed request body.

- **Type:** `TBody` (default: `unknown`)
- **Read-only:** No
- **Description:** Automatically parsed based on Content-Type
- **Supported types:** JSON, URL-encoded, raw text

```typescript
// JSON body
app.post('/users', async (ctx) => {
  console.log(ctx.body);
  // POST /users
  // Content-Type: application/json
  // { "name": "Alice", "email": "alice@example.com" }
  // { name: 'Alice', email: 'alice@example.com' }
});

// URL-encoded body
app.post('/form', async (ctx) => {
  console.log(ctx.body);
  // POST /form
  // Content-Type: application/x-www-form-urlencoded
  // name=Alice&email=alice@example.com
  // { name: 'Alice', email: 'alice@example.com' }
});

// With validation
app.post('/users',
  validate({
    body: z.object({
      name: z.string().min(2),
      email: z.string().email(),
    }),
  }),
  async (ctx) => {
    const { name, email } = ctx.body; // Type-safe
  }
);
```

---

### headers

Request headers.

- **Type:** `Record<string, string | string[] | undefined>`
- **Read-only:** Yes
- **Description:** All HTTP headers (lowercase keys)

```typescript
app.get('/example', async (ctx) => {
  const auth = ctx.headers.authorization;
  const contentType = ctx.headers['content-type'];
  const userAgent = ctx.headers['user-agent'];

  console.log({ auth, contentType, userAgent });
});

// Common patterns
app.use(async (ctx, next) => {
  const token = ctx.headers.authorization?.replace('Bearer ', '');
  const apiKey = ctx.headers['x-api-key'];
  const clientId = ctx.headers['x-client-id'];

  await next();
});
```

---

## Response Methods

### json()

Send a JSON response.

```typescript
json(data: unknown, status?: number): void
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `data` | `unknown` | Yes | - | Data to serialize as JSON |
| `status` | `number` | No | `200` | HTTP status code |

#### Examples

```typescript
// Basic JSON response
app.get('/users', async (ctx) => {
  ctx.json({ users: [] });
});

// With status code
app.post('/users', async (ctx) => {
  const user = await createUser(ctx.body);
  ctx.json({ user }, 201);
});

// Error response
app.get('/users/:id', async (ctx) => {
  const user = await getUser(ctx.params.id);

  if (!user) {
    ctx.json({ error: 'User not found' }, 404);
    return;
  }

  ctx.json({ user });
});
```

#### Performance

The `json()` method is highly optimized:
- Fast stringification for small objects
- Automatic Content-Length header
- No double-buffering

---

### text()

Send a plain text response.

```typescript
text(data: string, status?: number): void
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `data` | `string` | Yes | - | Text to send |
| `status` | `number` | No | `200` | HTTP status code |

#### Examples

```typescript
app.get('/health', async (ctx) => {
  ctx.text('OK');
});

app.get('/hello', async (ctx) => {
  ctx.text('Hello, World!', 200);
});

app.get('/error', async (ctx) => {
  ctx.text('Not Found', 404);
});
```

---

### status()

Set HTTP status code.

```typescript
status(code: number): Context
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `code` | `number` | Yes | HTTP status code |

#### Returns

`Context` - The context object (for chaining)

#### Examples

```typescript
// Chainable
app.delete('/users/:id', async (ctx) => {
  await deleteUser(ctx.params.id);
  ctx.status(204);
});

// With setHeader
app.get('/example', async (ctx) => {
  ctx.status(200)
     .setHeader('X-Custom', 'value')
     .json({ success: true });
});

// No content response
app.post('/webhook', async (ctx) => {
  await processWebhook(ctx.body);
  ctx.status(204); // No response body
});
```

---

### setHeader()

Set a response header.

```typescript
setHeader(key: string, value: string): Context
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | `string` | Yes | Header name |
| `value` | `string` | Yes | Header value |

#### Returns

`Context` - The context object (for chaining)

#### Examples

```typescript
// Single header
app.get('/example', async (ctx) => {
  ctx.setHeader('X-Custom-Header', 'value');
  ctx.json({ success: true });
});

// Multiple headers (chainable)
app.get('/example', async (ctx) => {
  ctx.setHeader('X-API-Version', '1.0')
     .setHeader('X-Request-ID', generateId())
     .setHeader('Cache-Control', 'no-cache')
     .json({ data: 'value' });
});

// Common patterns
app.get('/download', async (ctx) => {
  ctx.setHeader('Content-Type', 'application/octet-stream');
  ctx.setHeader('Content-Disposition', 'attachment; filename="file.pdf"');
  ctx.res.end(fileBuffer);
});

// Cache headers
app.get('/static', async (ctx) => {
  ctx.setHeader('Cache-Control', 'public, max-age=31536000');
  ctx.setHeader('ETag', generateETag(data));
  ctx.json(data);
});
```

---

## State Management

### state

Shared state object for passing data between middleware.

- **Type:** `Record<string, unknown>`
- **Read-only:** No
- **Description:** Arbitrary key-value storage for middleware communication

```typescript
// Set state in middleware
app.use(async (ctx, next) => {
  ctx.state.startTime = Date.now();
  ctx.state.requestId = generateId();
  await next();
});

// Use state in handler
app.get('/example', async (ctx) => {
  const requestId = ctx.state.requestId;
  console.log('Request ID:', requestId);
});

// Authentication middleware
const authenticate = async (ctx, next) => {
  const token = ctx.headers.authorization?.replace('Bearer ', '');
  const user = await verifyToken(token);

  ctx.state.user = user;
  ctx.state.authenticated = true;

  await next();
};

app.get('/profile', authenticate, async (ctx) => {
  const user = ctx.state.user;
  ctx.json({ profile: user });
});
```

---

### user

User object (populated by authentication middleware).

- **Type:** `unknown`
- **Read-only:** No
- **Description:** Authenticated user data

```typescript
import { authenticate } from 'ramapi';

app.get('/profile',
  authenticate,
  async (ctx) => {
    console.log(ctx.user);
    // { id: 123, email: 'user@example.com' }

    ctx.json({ user: ctx.user });
  }
);

// Custom auth middleware
const customAuth = async (ctx, next) => {
  const token = ctx.headers.authorization;
  const user = await verifyToken(token);

  ctx.user = user; // Set user

  await next();
};

app.get('/admin',
  customAuth,
  async (ctx) => {
    if (!ctx.user.isAdmin) {
      ctx.json({ error: 'Forbidden' }, 403);
      return;
    }

    ctx.json({ admin: true });
  }
);
```

---

## Observability Helpers

### trace

Trace context (when tracing is enabled).

- **Type:** `TraceContext | undefined`
- **Read-only:** Yes
- **Description:** Current trace context with span and trace IDs

```typescript
app.get('/example', async (ctx) => {
  if (ctx.trace) {
    console.log('Trace ID:', ctx.trace.traceId);
    console.log('Span ID:', ctx.trace.spanId);
    console.log('Protocol:', ctx.trace.protocol);
  }
});
```

---

### startSpan()

Start a new span for tracing operations.

```typescript
startSpan(name: string, attributes?: Record<string, any>): Span | undefined
```

```typescript
app.get('/users', async (ctx) => {
  const dbSpan = ctx.startSpan?.('database.query', {
    'db.system': 'postgresql',
    'db.operation': 'SELECT',
  });

  try {
    const users = await db.query('SELECT * FROM users');
    ctx.endSpan?.(dbSpan);
    ctx.json({ users });
  } catch (error) {
    ctx.endSpan?.(dbSpan, error);
    throw error;
  }
});
```

---

### endSpan()

End a span (with optional error).

```typescript
endSpan(span: Span | undefined, error?: Error): void
```

```typescript
app.get('/example', async (ctx) => {
  const span = ctx.startSpan?.('operation');

  try {
    await performOperation();
    ctx.endSpan?.(span);
  } catch (error) {
    ctx.endSpan?.(span, error); // Records error
    throw error;
  }
});
```

---

### addEvent()

Add an event to the current span.

```typescript
addEvent(name: string, attributes?: Record<string, any>): void
```

```typescript
app.post('/users', async (ctx) => {
  ctx.addEvent?.('user.validation', { valid: true });

  const user = await createUser(ctx.body);

  ctx.addEvent?.('user.created', { userId: user.id });

  ctx.json({ user }, 201);
});
```

---

### setAttributes()

Set attributes on the current span.

```typescript
setAttributes(attributes: Record<string, any>): void
```

```typescript
app.get('/users/:id', async (ctx) => {
  ctx.setAttributes?.({
    'user.id': ctx.params.id,
    'user.type': 'regular',
  });

  const user = await getUser(ctx.params.id);
  ctx.json({ user });
});
```

---

## Type Safety

### Typed Context

Use TypeScript generics for type-safe contexts.

```typescript
import { Context } from 'ramapi';
import { z } from 'zod';

// Define types
interface UserBody {
  name: string;
  email: string;
}

interface UserQuery {
  page: number;
  limit: number;
}

interface UserParams {
  id: string;
}

// Typed handler
app.post('/users',
  validate({
    body: z.object({
      name: z.string(),
      email: z.string().email(),
    }),
    query: z.object({
      page: z.string().transform(Number),
      limit: z.string().transform(Number),
    }),
  }),
  async (ctx: Context<UserBody, UserQuery>) => {
    const { name, email } = ctx.body; // Type-safe
    const { page, limit } = ctx.query; // Type-safe

    ctx.json({ success: true });
  }
);

// Typed params
app.get('/users/:id',
  async (ctx: Context<unknown, unknown, UserParams>) => {
    const userId = ctx.params.id; // Type-safe
  }
);
```

---

## Advanced Usage

### Streaming Responses

```typescript
import { createReadStream } from 'fs';

app.get('/download/:file', async (ctx) => {
  const filePath = `/files/${ctx.params.file}`;

  ctx.setHeader('Content-Type', 'application/octet-stream');
  ctx.setHeader('Content-Disposition', `attachment; filename="${ctx.params.file}"`);

  const stream = createReadStream(filePath);
  stream.pipe(ctx.res);
});
```

---

### Manual Response Handling

```typescript
app.get('/custom', async (ctx) => {
  // Don't use ctx.json() or ctx.text()
  ctx.res.writeHead(200, {
    'Content-Type': 'text/plain',
    'X-Custom': 'value',
  });

  ctx.res.write('Line 1\n');
  ctx.res.write('Line 2\n');
  ctx.res.end('Done');
});
```

---

### Conditional Responses

```typescript
app.get('/users/:id', async (ctx) => {
  const user = await getUser(ctx.params.id);

  if (!user) {
    ctx.json({ error: 'User not found' }, 404);
    return;
  }

  // Check ETag
  const etag = generateETag(user);
  if (ctx.headers['if-none-match'] === etag) {
    ctx.status(304); // Not Modified
    return;
  }

  ctx.setHeader('ETag', etag);
  ctx.json({ user });
});
```

---

## See Also

- [Server API](server.md) - Server class and configuration
- [Router API](router.md) - Router class and route management
- [Middleware API](middleware-api.md) - Built-in middleware
- [Validation API](validation.md) - Request validation

---

**Need help?** Check the [Context Guide](../core-concepts/context.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
