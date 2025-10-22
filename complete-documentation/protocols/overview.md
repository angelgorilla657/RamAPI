# Multi-Protocol Overview

RamAPI supports multiple API protocols from a single codebase. Write your business logic once and expose it via REST, GraphQL, and gRPC simultaneously.

## Table of Contents

1. [Why Multi-Protocol?](#why-multi-protocol)
2. [Supported Protocols](#supported-protocols)
3. [Protocol Manager](#protocol-manager)
4. [Unified Operations](#unified-operations)
5. [Quick Start](#quick-start)
6. [Use Cases](#use-cases)

---

## Why Multi-Protocol?

### Traditional Approach

Different clients need different protocols:

```
Mobile App -----> REST API
Web App --------> GraphQL API
Microservices --> gRPC API
```

This typically requires maintaining **three separate implementations** with duplicated logic.

### RamAPI Approach

Write once, expose everywhere:

```typescript
// Define operation once
const getUser = {
  name: 'getUser',
  handler: async (input, ctx) => {
    return await db.users.findById(input.id);
  },
  rest: { method: 'GET', path: '/users/:id' },
  graphql: { type: 'query' },
  grpc: { service: 'UserService', method: 'GetUser' },
};

// Available via all protocols automatically!
```

### Benefits

1. **Single Source of Truth**: Business logic defined once
2. **Automatic Protocol Translation**: RamAPI handles protocol differences
3. **Unified Tracing**: Track requests across all protocols
4. **Consistent Validation**: Zod schemas work for all protocols
5. **Less Code**: ~70% reduction in boilerplate

---

## Supported Protocols

### REST (HTTP/JSON)

Traditional RESTful API with HTTP methods:

```typescript
GET    /api/users/:id
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id
```

**Best for:**
- Web browsers
- Simple integrations
- Public APIs
- Mobile apps

**See:** [REST Documentation](rest.md)

### GraphQL

Query language with flexible data fetching:

```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

**Best for:**
- Complex data requirements
- Mobile apps (reduce over-fetching)
- Single-page applications
- Third-party integrations

**See:** [GraphQL Documentation](graphql.md)

### gRPC

High-performance RPC with Protocol Buffers:

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}
```

**Best for:**
- Microservices communication
- Low-latency requirements
- Streaming data
- Internal APIs

**See:** [gRPC Documentation](grpc.md)

---

## Protocol Manager

### Configuration

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  protocols: {
    // Enable GraphQL at /graphql
    graphql: {
      path: '/graphql',
      playground: true,
    },

    // Enable gRPC on port 50051
    grpc: {
      port: 50051,
    },
  },
});
```

### Protocol Negotiation

The Protocol Manager automatically routes requests:

```
Request → Protocol Manager → [GraphQL Adapter]
                          ↘ [gRPC Adapter]
                          ↘ [REST Router] (fallback)
```

---

## Unified Operations

### Operation Interface

```typescript
interface Operation<TInput, TOutput> {
  name: string;
  description?: string;
  input?: ZodSchema<TInput>;
  output?: ZodSchema<TOutput>;
  handler: (input: TInput, ctx: Context) => Promise<TOutput>;

  // Protocol-specific metadata
  rest?: {
    method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
    path: string;
  };
  graphql?: {
    type: 'query' | 'mutation' | 'subscription';
  };
  grpc?: {
    service: string;
    method: string;
  };
}
```

### Example: Multi-Protocol Operation

```typescript
import { z } from 'zod';

const getUserOperation = {
  name: 'getUser',
  description: 'Get user by ID',

  // Shared validation
  input: z.object({
    id: z.string().uuid(),
  }),

  output: z.object({
    id: z.string(),
    name: z.string(),
    email: z.string().email(),
  }),

  // Shared business logic
  handler: async (input, ctx) => {
    const user = await db.users.findById(input.id);
    if (!user) {
      throw new HTTPError(404, 'User not found');
    }
    return user;
  },

  // REST endpoint
  rest: {
    method: 'GET',
    path: '/users/:id',
  },

  // GraphQL query
  graphql: {
    type: 'query',
  },

  // gRPC method
  grpc: {
    service: 'UserService',
    method: 'GetUser',
  },
};

// Register operation
protocolManager.registerOperation(getUserOperation);
```

Now available via:

```bash
# REST
curl http://localhost:3000/users/123

# GraphQL
curl -X POST http://localhost:3000/graphql \
  -d '{"query": "{ getUser(id: \"123\") { name email } }"}'

# gRPC
grpcurl -d '{"id":"123"}' localhost:50051 UserService/GetUser
```

---

## Quick Start

### 1. Enable Protocols

```typescript
import { createApp, ProtocolManager } from 'ramapi';

const app = createApp({
  protocols: {
    graphql: true,
    grpc: { port: 50051 },
  },
});
```

### 2. Define Operations

```typescript
// Create operations
const operations = [
  {
    name: 'listUsers',
    handler: async () => await db.users.findAll(),
    rest: { method: 'GET', path: '/users' },
    graphql: { type: 'query' },
    grpc: { service: 'UserService', method: 'ListUsers' },
  },
  {
    name: 'createUser',
    input: z.object({
      name: z.string(),
      email: z.string().email(),
    }),
    handler: async (input) => await db.users.create(input),
    rest: { method: 'POST', path: '/users' },
    graphql: { type: 'mutation' },
    grpc: { service: 'UserService', method: 'CreateUser' },
  },
];

// Register with protocol manager
const protocolManager = app.getProtocolManager();
operations.forEach(op => protocolManager?.registerOperation(op));
```

### 3. Start Server

```typescript
await app.listen(3000);
// REST: http://localhost:3000
// GraphQL: http://localhost:3000/graphql
// gRPC: localhost:50051
```

---

## Use Cases

### Use Case 1: Unified API for Multiple Clients

```typescript
// Single operation definition
const searchProducts = {
  name: 'searchProducts',
  input: z.object({
    query: z.string(),
    limit: z.number().optional(),
  }),
  handler: async (input) => {
    return await db.products.search(input.query, input.limit);
  },
  rest: { method: 'GET', path: '/products/search' },
  graphql: { type: 'query' },
  grpc: { service: 'ProductService', method: 'Search' },
};

// Available to all clients:
// - Web: GraphQL for flexible queries
// - Mobile: REST for simplicity
// - Backend: gRPC for performance
```

### Use Case 2: Gradual Migration

```typescript
// Start with REST
const app = createApp();
app.get('/users', getUsers);
app.post('/users', createUser);

// Add GraphQL for new features
app = createApp({
  protocols: {
    graphql: true,
  },
});

// Migrate to gRPC for microservices
app = createApp({
  protocols: {
    graphql: true,
    grpc: { port: 50051 },
  },
});
```

### Use Case 3: Resource-Based APIs

```typescript
import { Resource } from 'ramapi';

// Define resource once
const userResource: Resource<User> = {
  name: 'User',
  schema: z.object({
    id: z.string(),
    name: z.string(),
    email: z.string().email(),
  }),
  operations: {
    list: async () => await db.users.findAll(),
    get: async (id) => await db.users.findById(id),
    create: async (input) => await db.users.create(input),
    update: async (id, input) => await db.users.update(id, input),
    delete: async (id) => await db.users.delete(id),
  },
};

// Automatically generates:
// REST: GET /users, POST /users, GET /users/:id, etc.
// GraphQL: users query, user(id) query, createUser mutation, etc.
// gRPC: ListUsers, GetUser, CreateUser, etc.
```

---

## Protocol Comparison

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| **Ease of Use** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Performance** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Flexibility** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Tooling** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Browser Support** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Streaming** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Type Safety** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## Complete Example

```typescript
import { createApp, ProtocolManager } from 'ramapi';
import { z } from 'zod';

const app = createApp({
  protocols: {
    graphql: {
      path: '/graphql',
      playground: true,
    },
    grpc: {
      port: 50051,
    },
  },
});

// Define operations
const operations = [
  {
    name: 'getUser',
    input: z.object({ id: z.string() }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => {
      return await db.users.findById(input.id);
    },
    rest: { method: 'GET', path: '/users/:id' },
    graphql: { type: 'query' },
    grpc: { service: 'UserService', method: 'GetUser' },
  },
  {
    name: 'createUser',
    input: z.object({
      name: z.string(),
      email: z.string().email(),
    }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => {
      return await db.users.create(input);
    },
    rest: { method: 'POST', path: '/users' },
    graphql: { type: 'mutation' },
    grpc: { service: 'UserService', method: 'CreateUser' },
  },
];

// Register operations
const protocolManager = app.getProtocolManager();
operations.forEach(op => protocolManager?.registerOperation(op));

await app.listen(3000);
console.log('REST API: http://localhost:3000');
console.log('GraphQL: http://localhost:3000/graphql');
console.log('gRPC: localhost:50051');
```

---

## Next Steps

- **[REST API](rest.md)** - HTTP/JSON endpoints
- **[GraphQL](graphql.md)** - Query language and schema
- **[gRPC](grpc.md)** - Protocol Buffers and services

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
