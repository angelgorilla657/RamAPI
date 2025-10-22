# Types Reference

Complete TypeScript type definitions for RamAPI.

## Table of Contents

1. [Core Types](#core-types)
2. [HTTP Types](#http-types)
3. [Handler Types](#handler-types)
4. [Configuration Types](#configuration-types)
5. [Error Types](#error-types)
6. [Utility Types](#utility-types)

---

## Core Types

### Context

The main context object passed to handlers and middleware.

```typescript
interface Context<TBody = unknown, TQuery = unknown, TParams = unknown> {
  // Request
  req: IncomingMessage;
  res: ServerResponse;
  method: HTTPMethod;
  url: string;
  path: string;
  query: TQuery;
  params: TParams;
  body: TBody;
  headers: Record<string, string | string[] | undefined>;

  // Response
  json: (data: unknown, status?: number) => void;
  text: (data: string, status?: number) => void;
  status: (code: number) => Context<TBody, TQuery, TParams>;
  setHeader: (key: string, value: string) => Context<TBody, TQuery, TParams>;

  // State
  state: Record<string, unknown>;
  user?: unknown;

  // Observability
  trace?: TraceContext;
  startSpan?: (name: string, attributes?: Record<string, any>) => Span | undefined;
  endSpan?: (span: Span | undefined, error?: Error) => void;
  addEvent?: (name: string, attributes?: Record<string, any>) => void;
  setAttributes?: (attributes: Record<string, any>) => void;
}
```

**Type Parameters:**

- `TBody` - Type of request body (default: `unknown`)
- `TQuery` - Type of query parameters (default: `unknown`)
- `TParams` - Type of route parameters (default: `unknown`)

**Example:**

```typescript
import { Context } from 'ramapi';

interface UserBody {
  name: string;
  email: string;
}

interface UserParams {
  id: string;
}

app.post('/users/:id', async (ctx: Context<UserBody, unknown, UserParams>) => {
  const userId = ctx.params.id; // string
  const { name, email } = ctx.body; // { name: string, email: string }
});
```

---

### Handler

Function type for route handlers.

```typescript
type Handler<TBody = unknown, TQuery = unknown, TParams = unknown> = (
  ctx: Context<TBody, TQuery, TParams>
) => void | Promise<void>;
```

**Type Parameters:**

- `TBody` - Type of request body
- `TQuery` - Type of query parameters
- `TParams` - Type of route parameters

**Example:**

```typescript
import { Handler, Context } from 'ramapi';

const getUser: Handler = async (ctx) => {
  const user = await db.getUser(ctx.params.id);
  ctx.json({ user });
};

// Type-safe handler
const createUser: Handler<{ name: string; email: string }> = async (ctx) => {
  const { name, email } = ctx.body; // Typed!
  const user = await db.createUser({ name, email });
  ctx.json({ user }, 201);
};
```

---

### Middleware

Function type for middleware.

```typescript
type Middleware = (
  ctx: Context,
  next: () => Promise<void>
) => void | Promise<void>;
```

**Example:**

```typescript
import { Middleware } from 'ramapi';

const logger: Middleware = async (ctx, next) => {
  console.log(`${ctx.method} ${ctx.path}`);
  await next();
};

const authenticate: Middleware = async (ctx, next) => {
  const token = ctx.headers.authorization;
  if (!token) {
    ctx.json({ error: 'Unauthorized' }, 401);
    return;
  }
  await next();
};
```

---

## HTTP Types

### HTTPMethod

Supported HTTP methods.

```typescript
type HTTPMethod =
  | 'GET'
  | 'POST'
  | 'PUT'
  | 'PATCH'
  | 'DELETE'
  | 'OPTIONS'
  | 'HEAD';
```

**Example:**

```typescript
import { HTTPMethod } from 'ramapi';

const methods: HTTPMethod[] = ['GET', 'POST', 'PUT', 'DELETE'];

function logRequest(method: HTTPMethod, path: string) {
  console.log(`${method} ${path}`);
}
```

---

### Route

Route definition with metadata.

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

**Example:**

```typescript
import { Route } from 'ramapi';

const userRoutes: Route[] = [
  {
    method: 'GET',
    path: '/users',
    handler: getUsers,
    meta: {
      description: 'List all users',
      tags: ['users'],
      auth: false,
    },
  },
  {
    method: 'POST',
    path: '/users',
    handler: createUser,
    middleware: [authenticate],
    meta: {
      description: 'Create new user',
      tags: ['users'],
      auth: true,
    },
  },
];
```

---

## Handler Types

### ErrorHandler

Custom error handler function.

```typescript
type ErrorHandler = (
  error: Error,
  ctx: Context
) => void | Promise<void>;
```

**Example:**

```typescript
import { ErrorHandler } from 'ramapi';

const errorHandler: ErrorHandler = async (error, ctx) => {
  console.error('Error:', error);

  if (error instanceof ValidationError) {
    ctx.json({ error: error.message, details: error.details }, 400);
    return;
  }

  ctx.json({ error: 'Internal server error' }, 500);
};

const app = createApp({
  onError: errorHandler,
});
```

---

## Configuration Types

### ServerConfig

Main server configuration.

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

**Example:**

```typescript
import { ServerConfig } from 'ramapi';

const config: ServerConfig = {
  port: 3000,
  host: '0.0.0.0',
  cors: true,
  middleware: [logger()],
  onError: customErrorHandler,
  observability: {
    tracing: { enabled: true, serviceName: 'my-api' },
  },
};

const app = createApp(config);
```

---

### RouterConfig

Router configuration.

```typescript
interface RouterConfig {
  prefix?: string;
  middleware?: Middleware[];
}
```

**Example:**

```typescript
import { Router, RouterConfig } from 'ramapi';

const config: RouterConfig = {
  prefix: '/api/v1',
  middleware: [authenticate, logger()],
};

const router = new Router(config);
```

---

### CorsConfig

CORS configuration.

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

**Example:**

```typescript
import { CorsConfig } from 'ramapi';

const corsConfig: CorsConfig = {
  origin: 'https://example.com',
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400,
};

const app = createApp({ cors: corsConfig });
```

---

### AdapterConfig

HTTP adapter configuration.

```typescript
interface AdapterConfig {
  type?: 'node-http' | 'uwebsockets';
  options?: Record<string, any>;
}
```

**Example:**

```typescript
import { AdapterConfig } from 'ramapi';

const adapterConfig: AdapterConfig = {
  type: 'uwebsockets',
  options: {
    idleTimeout: 120,
    maxBackpressure: 1024 * 1024,
  },
};

const app = createApp({ adapter: adapterConfig });
```

---

## Error Types

### HTTPError

HTTP error class with status code.

```typescript
class HTTPError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public details?: unknown
  )
}
```

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `statusCode` | `number` | HTTP status code |
| `message` | `string` | Error message |
| `details` | `unknown` | Additional error details |

**Example:**

```typescript
import { HTTPError } from 'ramapi';

// Throw HTTP error
throw new HTTPError(404, 'User not found');

// With details
throw new HTTPError(400, 'Validation failed', {
  errors: [
    { field: 'email', message: 'Invalid email' },
  ],
});

// Catch and handle
try {
  await doSomething();
} catch (error) {
  if (error instanceof HTTPError) {
    ctx.json({ error: error.message }, error.statusCode);
  }
}
```

---

### ValidationError

Validation error structure.

```typescript
interface ValidationError {
  field: string;
  message: string;
  code: string;
}
```

**Example:**

```typescript
import { ValidationError } from 'ramapi';

const errors: ValidationError[] = [
  {
    field: 'body.email',
    message: 'Invalid email address',
    code: 'invalid_string',
  },
  {
    field: 'body.age',
    message: 'Must be at least 18',
    code: 'too_small',
  },
];

throw new HTTPError(400, 'Validation failed', { errors });
```

---

## Utility Types

### InferSchema

Infer TypeScript type from Zod schema.

```typescript
type InferSchema<T extends ZodSchema> = z.infer<T>;
```

**Example:**

```typescript
import { InferSchema } from 'ramapi';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int(),
});

type User = InferSchema<typeof userSchema>;
// { name: string; email: string; age: number }

const createUser: Handler<User> = async (ctx) => {
  const user: User = ctx.body; // Fully typed
};
```

---

### ServerAdapter

Interface for HTTP server adapters.

```typescript
interface ServerAdapter {
  readonly name: string;
  listen(port: number, host: string): Promise<void>;
  close(): Promise<void>;
  onRequest(handler: RequestHandler): void;
  getRequestInfo(raw: any): RawRequestInfo;
  sendResponse(raw: any, statusCode: number, headers: Record<string, string>, body: Buffer | string): void;
  parseBody(raw: any): Promise<unknown>;
  readonly supportsStreaming?: boolean;
  readonly supportsHTTP2?: boolean;
}
```

**Example:**

```typescript
import { ServerAdapter } from 'ramapi';

class CustomAdapter implements ServerAdapter {
  readonly name = 'custom';

  async listen(port: number, host: string): Promise<void> {
    // Implementation
  }

  async close(): Promise<void> {
    // Implementation
  }

  // ... other methods
}
```

---

### RequestHandler

Adapter request handler type.

```typescript
type RequestHandler = (
  requestInfo: RawRequestInfo,
  rawRequest: any
) => Promise<RawResponseData>;
```

---

### RawRequestInfo

Normalized request information from adapter.

```typescript
interface RawRequestInfo {
  method: string;
  url: string;
  headers: Record<string, string | string[]>;
}
```

---

### RawResponseData

Response data returned to adapter.

```typescript
interface RawResponseData {
  statusCode: number;
  headers: Record<string, string>;
  body: Buffer | string;
}
```

---

## Complete Type Example

```typescript
import {
  Context,
  Handler,
  Middleware,
  HTTPMethod,
  ServerConfig,
  CorsConfig,
  HTTPError,
  ValidationError,
  InferSchema,
} from 'ramapi';
import { z } from 'zod';

// Schema
const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

// Infer type
type User = InferSchema<typeof userSchema>;

// Type-safe handler
const createUser: Handler<User> = async (ctx) => {
  const { name, email } = ctx.body;

  // Validation
  if (!name || !email) {
    throw new HTTPError(400, 'Invalid user data');
  }

  const user = await db.createUser({ name, email });
  ctx.json({ user }, 201);
};

// Middleware
const logger: Middleware = async (ctx, next) => {
  console.log(`${ctx.method} ${ctx.path}`);
  await next();
};

// CORS config
const corsConfig: CorsConfig = {
  origin: 'https://example.com',
  credentials: true,
};

// Server config
const config: ServerConfig = {
  port: 3000,
  cors: corsConfig,
  middleware: [logger],
};

// Create app
const app = createApp(config);
app.post('/users', validate({ body: userSchema }), createUser);
```

---

## See Also

- [Context API](context.md) - Context reference
- [Server API](server.md) - Server reference
- [Router API](router.md) - Router reference

---

**Need help?** Check the [TypeScript Guide](../guides/typescript.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
