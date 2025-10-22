# gRPC

High-performance RPC with Protocol Buffers for microservices communication. RamAPI automatically generates proto files and handles gRPC server setup.

## Table of Contents

1. [Overview](#overview)
2. [Setup](#setup)
3. [Proto Generation](#proto-generation)
4. [Service Definitions](#service-definitions)
5. [gRPC Clients](#grpc-clients)

---

## Overview

gRPC uses Protocol Buffers for efficient, type-safe communication:

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
}
```

**Benefits:**
- 2-3x faster than REST/JSON
- Type-safe contracts
- Built-in streaming
- Perfect for microservices

---

## Setup

### Enable gRPC

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  protocols: {
    grpc: {
      port: 50051,           // gRPC port
      host: '0.0.0.0',       // Bind address
    },
  },
});

await app.listen(3000);
// HTTP: localhost:3000
// gRPC: localhost:50051
```

### Install Dependencies

```bash
npm install @grpc/grpc-js @grpc/proto-loader
```

---

## Proto Generation

### Automatic Generation

RamAPI automatically generates proto files from your operations:

```typescript
import { z } from 'zod';

const getUserOperation = {
  name: 'getUser',

  input: z.object({
    id: z.string().uuid(),
  }),

  output: z.object({
    id: z.string(),
    name: z.string(),
    email: z.string(),
  }),

  handler: async (input) => {
    return await db.users.findById(input.id);
  },

  grpc: {
    service: 'UserService',
    method: 'GetUser',
  },
};

// Automatically generates:
// service UserService {
//   rpc GetUser (GetUserRequest) returns (GetUserResponse);
// }
```

### Development Mode

In development, protos are generated at runtime:

```typescript
// Automatic proto generation and loading
const app = createApp({
  protocols: {
    grpc: { port: 50051 },
  },
});

// Proto files generated in .ramapi/protos/
```

### Production Mode

For production, pre-compile protos during build:

```bash
# Add to package.json
{
  "scripts": {
    "build:protos": "ramapi build-protos"
  }
}
```

```typescript
// Uses pre-compiled protos (faster startup)
NODE_ENV=production node server.js
```

---

## Service Definitions

### Simple Service

```typescript
const operations = [
  {
    name: 'getUser',
    input: z.object({ id: z.string() }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => await db.users.findById(input.id),
    grpc: {
      service: 'UserService',
      method: 'GetUser',
    },
  },
];
```

**Generated Proto:**
```protobuf
syntax = "proto3";

package ramapi;

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

### Full CRUD Service

```typescript
const userOperations = [
  // List users
  {
    name: 'listUsers',
    output: z.object({
      users: z.array(z.object({
        id: z.string(),
        name: z.string(),
      })),
    }),
    handler: async () => ({
      users: await db.users.findAll(),
    }),
    grpc: { service: 'UserService', method: 'ListUsers' },
  },

  // Get user
  {
    name: 'getUser',
    input: z.object({ id: z.string() }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => await db.users.findById(input.id),
    grpc: { service: 'UserService', method: 'GetUser' },
  },

  // Create user
  {
    name: 'createUser',
    input: z.object({
      name: z.string(),
      email: z.string(),
    }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => await db.users.create(input),
    grpc: { service: 'UserService', method: 'CreateUser' },
  },

  // Update user
  {
    name: 'updateUser',
    input: z.object({
      id: z.string(),
      name: z.string().optional(),
      email: z.string().optional(),
    }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => {
      const { id, ...updates } = input;
      return await db.users.update(id, updates);
    },
    grpc: { service: 'UserService', method: 'UpdateUser' },
  },

  // Delete user
  {
    name: 'deleteUser',
    input: z.object({ id: z.string() }),
    output: z.object({ success: z.boolean() }),
    handler: async (input) => {
      await db.users.delete(input.id);
      return { success: true };
    },
    grpc: { service: 'UserService', method: 'DeleteUser' },
  },
];

// Register all operations
const protocolManager = app.getProtocolManager();
userOperations.forEach(op => protocolManager?.registerOperation(op));
```

---

## gRPC Clients

### Node.js Client

```typescript
import grpc from '@grpc/grpc-js';
import protoLoader from '@grpc/proto-loader';

// Load proto
const packageDefinition = protoLoader.loadSync('user.proto');
const proto = grpc.loadPackageDefinition(packageDefinition);

// Create client
const client = new proto.ramapi.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Call methods
client.GetUser({ id: '123' }, (error, response) => {
  if (error) {
    console.error('Error:', error);
  } else {
    console.log('User:', response);
  }
});

// Async/await wrapper
function getUserAsync(id: string): Promise<any> {
  return new Promise((resolve, reject) => {
    client.GetUser({ id }, (error, response) => {
      if (error) reject(error);
      else resolve(response);
    });
  });
}

const user = await getUserAsync('123');
```

### grpcurl (CLI Testing)

```bash
# Install grpcurl
brew install grpcurl

# List services
grpcurl -plaintext localhost:50051 list

# List methods
grpcurl -plaintext localhost:50051 list UserService

# Call method
grpcurl -plaintext \
  -d '{"id":"123"}' \
  localhost:50051 \
  ramapi.UserService/GetUser
```

### Other Languages

gRPC supports many languages:

**Python:**
```python
import grpc
import user_pb2
import user_pb2_grpc

channel = grpc.insecure_channel('localhost:50051')
stub = user_pb2_grpc.UserServiceStub(channel)

response = stub.GetUser(user_pb2.GetUserRequest(id='123'))
print(response)
```

**Go:**
```go
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
client := pb.NewUserServiceClient(conn)

response, _ := client.GetUser(context.Background(), &pb.GetUserRequest{
    Id: "123",
})
fmt.Println(response)
```

---

## Complete Example

```typescript
import { createApp } from 'ramapi';
import { z } from 'zod';

const app = createApp({
  protocols: {
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
    handler: async (input) => await db.users.findById(input.id),
    grpc: { service: 'UserService', method: 'GetUser' },
  },
  {
    name: 'createUser',
    input: z.object({
      name: z.string(),
      email: z.string(),
    }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => await db.users.create(input),
    grpc: { service: 'UserService', method: 'CreateUser' },
  },
];

// Register operations
const protocolManager = app.getProtocolManager();
operations.forEach(op => protocolManager?.registerOperation(op));

await app.listen(3000);
console.log('gRPC server: localhost:50051');
```

**Test with grpcurl:**
```bash
# Get user
grpcurl -plaintext -d '{"id":"123"}' localhost:50051 ramapi.UserService/GetUser

# Create user
grpcurl -plaintext -d '{"name":"John","email":"john@example.com"}' \
  localhost:50051 ramapi.UserService/CreateUser
```

---

## Next Steps

- [REST API](rest.md)
- [GraphQL](graphql.md)
- [Multi-Protocol Overview](overview.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
