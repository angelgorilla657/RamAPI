# gRPC Service Example

Complete gRPC service example with proto definitions, server implementation, and client usage.

> **Note:** This is a documentation example showing how to use RamAPI in your own project. The code assumes you have RamAPI installed via npm.

## Prerequisites

```bash
# Install RamAPI
npm install ramapi

# Install dependencies
npm install @grpc/grpc-js @grpc/proto-loader better-sqlite3
npm install -D @types/better-sqlite3
```

## Overview

This example demonstrates:
- Protocol Buffers (proto) definition
- gRPC server implementation
- Service methods (unary, server streaming, client streaming, bidirectional)
- gRPC client usage
- Error handling

## Proto Definition

### user.proto

```protobuf
syntax = "proto3";

package user;

service UserService {
  // Unary RPC
  rpc GetUser (GetUserRequest) returns (User);
  rpc CreateUser (CreateUserRequest) returns (User);
  rpc UpdateUser (UpdateUserRequest) returns (User);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);

  // Server streaming RPC
  rpc ListUsers (ListUsersRequest) returns (stream User);

  // Client streaming RPC
  rpc CreateMultipleUsers (stream CreateUserRequest) returns (CreateUsersResponse);

  // Bidirectional streaming RPC
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  string created_at = 5;
}

message GetUserRequest {
  string id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
}

message UpdateUserRequest {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
}

message DeleteUserRequest {
  string id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
}

message CreateUsersResponse {
  repeated User users = 1;
  int32 count = 2;
}

message ChatMessage {
  string user_id = 1;
  string message = 2;
  string timestamp = 3;
}
```

## Server Implementation

### index.ts

```typescript
import { createApp } from 'ramapi';
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import Database from 'better-sqlite3';

// Database setup
const db = new Database('grpc.db');

db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    age INTEGER,
    created_at TEXT NOT NULL
  )
`);

// Load proto
const packageDefinition = protoLoader.loadSync('user.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const proto = grpc.loadPackageDefinition(packageDefinition) as any;

// Helper
function generateId(): string {
  return `user-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

// gRPC Service Implementation
const userService = {
  // Unary RPC: GetUser
  GetUser: (call: any, callback: any) => {
    const { id } = call.request;

    const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);

    if (!user) {
      callback({
        code: grpc.status.NOT_FOUND,
        message: 'User not found',
      });
      return;
    }

    callback(null, user);
  },

  // Unary RPC: CreateUser
  CreateUser: (call: any, callback: any) => {
    const { name, email, age } = call.request;

    try {
      const id = generateId();
      const created_at = new Date().toISOString();

      db.prepare(`
        INSERT INTO users (id, name, email, age, created_at)
        VALUES (?, ?, ?, ?, ?)
      `).run(id, name, email, age, created_at);

      const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
      callback(null, user);
    } catch (error: any) {
      callback({
        code: grpc.status.ALREADY_EXISTS,
        message: 'Email already exists',
      });
    }
  },

  // Unary RPC: UpdateUser
  UpdateUser: (call: any, callback: any) => {
    const { id, name, email, age } = call.request;

    const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
    if (!user) {
      callback({
        code: grpc.status.NOT_FOUND,
        message: 'User not found',
      });
      return;
    }

    const updates: string[] = [];
    const values: any[] = [];

    if (name) { updates.push('name = ?'); values.push(name); }
    if (email) { updates.push('email = ?'); values.push(email); }
    if (age) { updates.push('age = ?'); values.push(age); }

    if (updates.length > 0) {
      values.push(id);
      db.prepare(`UPDATE users SET ${updates.join(', ')} WHERE id = ?`).run(...values);
    }

    const updatedUser = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
    callback(null, updatedUser);
  },

  // Unary RPC: DeleteUser
  DeleteUser: (call: any, callback: any) => {
    const { id } = call.request;

    const result = db.prepare('DELETE FROM users WHERE id = ?').run(id);

    callback(null, { success: result.changes > 0 });
  },

  // Server Streaming RPC: ListUsers
  ListUsers: (call: any) => {
    const { page = 1, limit = 10 } = call.request;
    const offset = (page - 1) * limit;

    const users = db.prepare(`
      SELECT * FROM users
      LIMIT ? OFFSET ?
    `).all(limit, offset);

    // Stream each user
    users.forEach((user: any) => {
      call.write(user);
    });

    call.end();
  },

  // Client Streaming RPC: CreateMultipleUsers
  CreateMultipleUsers: (call: any, callback: any) => {
    const createdUsers: any[] = [];

    call.on('data', (request: any) => {
      try {
        const id = generateId();
        const created_at = new Date().toISOString();

        db.prepare(`
          INSERT INTO users (id, name, email, age, created_at)
          VALUES (?, ?, ?, ?, ?)
        `).run(id, request.name, request.email, request.age, created_at);

        const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
        createdUsers.push(user);
      } catch (error) {
        // Skip duplicates
      }
    });

    call.on('end', () => {
      callback(null, {
        users: createdUsers,
        count: createdUsers.length,
      });
    });
  },

  // Bidirectional Streaming RPC: Chat
  Chat: (call: any) => {
    call.on('data', (message: any) => {
      // Echo message back to all clients (simplified)
      call.write({
        user_id: message.user_id,
        message: message.message,
        timestamp: new Date().toISOString(),
      });
    });

    call.on('end', () => {
      call.end();
    });
  },
};

// Create RamAPI server with gRPC
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

// Optional REST endpoints
app.get('/api/users', async (ctx) => {
  const users = db.prepare('SELECT * FROM users').all();
  ctx.json({ users });
});

await app.listen(3000);
console.log('🚀 gRPC server running on localhost:50051');
console.log('🌐 REST API running at http://localhost:3000');
```

## Client Usage

### client.ts

```typescript
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

// Load proto
const packageDefinition = protoLoader.loadSync('user.proto');
const proto = grpc.loadPackageDefinition(packageDefinition) as any;

// Create client
const client = new proto.user.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Unary RPC: CreateUser
function createUser(name: string, email: string, age: number) {
  return new Promise((resolve, reject) => {
    client.CreateUser({ name, email, age }, (error: any, response: any) => {
      if (error) reject(error);
      else resolve(response);
    });
  });
}

// Unary RPC: GetUser
function getUser(id: string) {
  return new Promise((resolve, reject) => {
    client.GetUser({ id }, (error: any, response: any) => {
      if (error) reject(error);
      else resolve(response);
    });
  });
}

// Server Streaming RPC: ListUsers
function listUsers(page: number, limit: number) {
  return new Promise((resolve, reject) => {
    const users: any[] = [];

    const call = client.ListUsers({ page, limit });

    call.on('data', (user: any) => {
      users.push(user);
    });

    call.on('end', () => {
      resolve(users);
    });

    call.on('error', (error: any) => {
      reject(error);
    });
  });
}

// Client Streaming RPC: CreateMultipleUsers
function createMultipleUsers(users: any[]) {
  return new Promise((resolve, reject) => {
    const call = client.CreateMultipleUsers((error: any, response: any) => {
      if (error) reject(error);
      else resolve(response);
    });

    users.forEach((user) => {
      call.write(user);
    });

    call.end();
  });
}

// Usage examples
async function main() {
  try {
    // Create user
    const user = await createUser('Alice', 'alice@example.com', 30);
    console.log('Created user:', user);

    // Get user
    const fetched = await getUser((user as any).id);
    console.log('Fetched user:', fetched);

    // List users
    const users = await listUsers(1, 10);
    console.log('Users:', users);

    // Create multiple users
    const result = await createMultipleUsers([
      { name: 'Bob', email: 'bob@example.com', age: 25 },
      { name: 'Charlie', email: 'charlie@example.com', age: 35 },
    ]);
    console.log('Created multiple users:', result);
  } catch (error) {
    console.error('Error:', error);
  }
}

main();
```

## Error Handling

```typescript
try {
  const user = await getUser('invalid-id');
} catch (error: any) {
  console.error('gRPC Error:');
  console.error('Code:', error.code); // grpc.status.NOT_FOUND
  console.error('Message:', error.message); // 'User not found'
}
```

## See Also

- [gRPC Guide](../multi-protocol/grpc.md)
- [Multi-Protocol Example](multi-protocol.md)
- [gRPC Documentation](https://grpc.io)
