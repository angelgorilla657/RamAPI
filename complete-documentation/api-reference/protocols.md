# Protocols API Reference

Complete API reference for RamAPI's multi-protocol support (REST, GraphQL, gRPC).

## Table of Contents

1. [ProtocolManager](#protocolmanager)
2. [GraphQL Configuration](#graphql-configuration)
3. [gRPC Configuration](#grpc-configuration)
4. [Protocol Detection](#protocol-detection)
5. [Complete Examples](#complete-examples)

---

## ProtocolManager

Manager class for handling multiple protocols in a single server.

### Configuration

```typescript
interface ProtocolManagerConfig {
  graphql?: GraphQLConfig | boolean;
  grpc?: GRPCConfig | boolean;
}
```

### Setup

```typescript
import { createApp } from 'ramapi';

const app = createApp({
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

await app.listen(3000);
// REST: http://localhost:3000
// GraphQL: http://localhost:3000/graphql
// gRPC: localhost:50051
```

---

## GraphQL Configuration

### GraphQLConfig

Configuration for GraphQL endpoint.

```typescript
interface GraphQLConfig {
  path?: string;
  schema: GraphQLSchema;
  context?: (req: IncomingMessage) => any;
  playground?: boolean;
  introspection?: boolean;
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `path` | `string` | `'/graphql'` | GraphQL endpoint path |
| `schema` | `GraphQLSchema` | - | GraphQL schema (required) |
| `context` | `Function` | `undefined` | Context factory function |
| `playground` | `boolean` | `true` | Enable GraphQL Playground |
| `introspection` | `boolean` | `true` | Enable schema introspection |

### Example

```typescript
import { createApp } from 'ramapi';
import { buildSchema } from 'graphql';

const schema = buildSchema(`
  type Query {
    hello: String
    users: [User]
  }

  type User {
    id: ID!
    name: String!
    email: String!
  }

  type Mutation {
    createUser(name: String!, email: String!): User
  }
`);

const root = {
  hello: () => 'Hello world!',
  users: () => [
    { id: '1', name: 'Alice', email: 'alice@example.com' },
  ],
  createUser: ({ name, email }) => {
    return { id: '2', name, email };
  },
};

const app = createApp({
  protocols: {
    graphql: {
      path: '/graphql',
      schema,
      rootValue: root,
      playground: true,
    },
  },
});

await app.listen(3000);
// GraphQL available at http://localhost:3000/graphql
```

### With Context

```typescript
const app = createApp({
  protocols: {
    graphql: {
      path: '/graphql',
      schema,
      context: (req) => ({
        userId: req.headers['x-user-id'],
        token: req.headers.authorization,
      }),
    },
  },
});

// Resolvers can access context
const root = {
  me: (_args, context) => {
    const userId = context.userId;
    return getUser(userId);
  },
};
```

### GraphQL Queries

**Query:**

```graphql
query {
  hello
  users {
    id
    name
    email
  }
}
```

**Mutation:**

```graphql
mutation {
  createUser(name: "Bob", email: "bob@example.com") {
    id
    name
    email
  }
}
```

---

## gRPC Configuration

### GRPCConfig

Configuration for gRPC server.

```typescript
interface GRPCConfig {
  port?: number;
  host?: string;
  services: any;
  credentials?: ServerCredentials;
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `port` | `number` | `50051` | gRPC server port |
| `host` | `string` | `'0.0.0.0'` | gRPC server host |
| `services` | `any` | - | gRPC service implementations (required) |
| `credentials` | `ServerCredentials` | Insecure | Server credentials |

### Example

**Define proto file (`user.proto`):**

```protobuf
syntax = "proto3";

package user;

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser (CreateUserRequest) returns (User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}
```

**Implement service:**

```typescript
import { createApp } from 'ramapi';
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

// Load proto
const packageDefinition = protoLoader.loadSync('user.proto');
const proto = grpc.loadPackageDefinition(packageDefinition);

// Implement service
const userService = {
  GetUser: (call, callback) => {
    const userId = call.request.id;
    const user = { id: userId, name: 'Alice', email: 'alice@example.com' };
    callback(null, user);
  },

  ListUsers: (call, callback) => {
    const users = [
      { id: '1', name: 'Alice', email: 'alice@example.com' },
      { id: '2', name: 'Bob', email: 'bob@example.com' },
    ];
    callback(null, { users, total: 2 });
  },

  CreateUser: (call, callback) => {
    const { name, email } = call.request;
    const user = { id: '3', name, email };
    callback(null, user);
  },
};

const app = createApp({
  adapter: { type: 'node-http' }, // Required for gRPC
  protocols: {
    grpc: {
      port: 50051,
      services: {
        'user.UserService': userService,
      },
    },
  },
});

await app.listen(3000);
// REST: http://localhost:3000
// gRPC: localhost:50051
```

### gRPC Client Example

```typescript
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDefinition = protoLoader.loadSync('user.proto');
const proto = grpc.loadPackageDefinition(packageDefinition);

const client = new proto.user.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Get user
client.GetUser({ id: '1' }, (error, response) => {
  console.log('User:', response);
});

// List users
client.ListUsers({ page: 1, limit: 10 }, (error, response) => {
  console.log('Users:', response.users);
});

// Create user
client.CreateUser(
  { name: 'Charlie', email: 'charlie@example.com' },
  (error, response) => {
    console.log('Created:', response);
  }
);
```

---

## Protocol Detection

The ProtocolManager automatically detects and routes requests to the correct protocol handler.

### Detection Logic

1. **GraphQL:** Requests to the GraphQL path (default: `/graphql`)
2. **gRPC:** gRPC protocol on separate port
3. **REST:** All other HTTP requests

### Example

```typescript
const app = createApp({
  protocols: {
    graphql: { path: '/graphql', schema },
    grpc: { port: 50051, services },
  },
});

// REST routes
app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});

await app.listen(3000);

// GET http://localhost:3000/api/users → REST
// POST http://localhost:3000/graphql → GraphQL
// gRPC localhost:50051 → gRPC
```

---

## Complete Examples

### Multi-Protocol Server

```typescript
import { createApp } from 'ramapi';
import { buildSchema } from 'graphql';
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

// GraphQL Schema
const graphqlSchema = buildSchema(`
  type Query {
    users: [User]
    user(id: ID!): User
  }

  type User {
    id: ID!
    name: String!
    email: String!
  }

  type Mutation {
    createUser(name: String!, email: String!): User
  }
`);

const graphqlRoot = {
  users: () => db.getAllUsers(),
  user: ({ id }) => db.getUser(id),
  createUser: ({ name, email }) => db.createUser({ name, email }),
};

// gRPC Service
const packageDefinition = protoLoader.loadSync('user.proto');
const proto = grpc.loadPackageDefinition(packageDefinition);

const grpcService = {
  GetUser: async (call, callback) => {
    const user = await db.getUser(call.request.id);
    callback(null, user);
  },
  ListUsers: async (call, callback) => {
    const users = await db.getAllUsers();
    callback(null, { users, total: users.length });
  },
};

// Create server with all protocols
const app = createApp({
  adapter: { type: 'node-http' }, // Required for gRPC
  protocols: {
    // GraphQL
    graphql: {
      path: '/graphql',
      schema: graphqlSchema,
      rootValue: graphqlRoot,
      playground: true,
    },

    // gRPC
    grpc: {
      port: 50051,
      services: {
        'user.UserService': grpcService,
      },
    },
  },

  observability: {
    tracing: {
      enabled: true,
      serviceName: 'multi-protocol-api',
    },
  },
});

// REST routes
app.get('/api/users', async (ctx) => {
  const users = await db.getAllUsers();
  ctx.json({ users });
});

app.get('/api/users/:id', async (ctx) => {
  const user = await db.getUser(ctx.params.id);
  ctx.json({ user });
});

app.post('/api/users', async (ctx) => {
  const user = await db.createUser(ctx.body);
  ctx.json({ user }, 201);
});

await app.listen(3000);

console.log('🚀 Multi-protocol server running:');
console.log('   REST API:    http://localhost:3000/api');
console.log('   GraphQL:     http://localhost:3000/graphql');
console.log('   gRPC:        localhost:50051');
```

### Shared Business Logic

```typescript
// Shared business logic
class UserService {
  async getUser(id: string) {
    return db.getUser(id);
  }

  async listUsers(page: number, limit: number) {
    return db.listUsers(page, limit);
  }

  async createUser(data: { name: string; email: string }) {
    return db.createUser(data);
  }
}

const userService = new UserService();

// REST handlers
app.get('/api/users/:id', async (ctx) => {
  const user = await userService.getUser(ctx.params.id);
  ctx.json({ user });
});

// GraphQL resolvers
const graphqlRoot = {
  user: ({ id }) => userService.getUser(id),
  users: () => userService.listUsers(1, 100),
  createUser: ({ name, email }) => userService.createUser({ name, email }),
};

// gRPC handlers
const grpcService = {
  GetUser: async (call, callback) => {
    const user = await userService.getUser(call.request.id);
    callback(null, user);
  },
  ListUsers: async (call, callback) => {
    const users = await userService.listUsers(call.request.page, call.request.limit);
    callback(null, { users, total: users.length });
  },
};
```

### Protocol-Specific Features

```typescript
const app = createApp({
  protocols: {
    // GraphQL with authentication
    graphql: {
      path: '/graphql',
      schema,
      context: (req) => ({
        user: verifyToken(req.headers.authorization),
      }),
    },

    // gRPC with SSL
    grpc: {
      port: 50051,
      services: grpcServices,
      credentials: grpc.ServerCredentials.createSsl(
        null,
        [{ private_key: privateKey, cert_chain: certChain }],
        false
      ),
    },
  },
});
```

---

## getProtocolManager()

Access the protocol manager from the server instance.

```typescript
const app = createApp({
  protocols: {
    graphql: { path: '/graphql', schema },
  },
});

const protocolManager = app.getProtocolManager();

if (protocolManager) {
  const graphqlSchema = protocolManager.getGraphQLSchema();
  console.log('GraphQL schema:', graphqlSchema);
}
```

---

## See Also

- [Multi-Protocol Guide](../multi-protocol/overview.md) - Multi-protocol overview
- [GraphQL Guide](../multi-protocol/graphql.md) - GraphQL setup guide
- [gRPC Guide](../multi-protocol/grpc.md) - gRPC setup guide

---

**Need help?** Check the [Multi-Protocol Guide](../multi-protocol/overview.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
