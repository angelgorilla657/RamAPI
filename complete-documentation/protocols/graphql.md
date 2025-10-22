# GraphQL

Expose your API via GraphQL for flexible, efficient data fetching. RamAPI automatically generates GraphQL schemas from your operations.

## Table of Contents

1. [Overview](#overview)
2. [Setup](#setup)
3. [Schema Builder](#schema-builder)
4. [Queries](#queries)
5. [Mutations](#mutations)
6. [Playground](#playground)

---

## Overview

GraphQL allows clients to request exactly the data they need:

```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

---

## Setup

### Enable GraphQL

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  protocols: {
    graphql: {
      path: '/graphql',      // GraphQL endpoint
      playground: true,       // Enable playground
      introspection: true,    // Enable schema introspection
    },
  },
});
```

### Register Operations

```typescript
import { z } from 'zod';

const getUserOperation = {
  name: 'getUser',
  description: 'Get user by ID',

  input: z.object({
    id: z.string().uuid(),
  }),

  output: z.object({
    id: z.string(),
    name: z.string(),
    email: z.string().email(),
  }),

  handler: async (input, ctx) => {
    return await db.users.findById(input.id);
  },

  // GraphQL metadata
  graphql: {
    type: 'query',
  },
};

// Register with protocol manager
const protocolManager = app.getProtocolManager();
protocolManager?.registerOperation(getUserOperation);
```

---

## Schema Builder

### Automatic Schema Generation

RamAPI automatically generates GraphQL schema from Zod schemas:

```typescript
// Zod schema
const userSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  age: z.number().int().optional(),
});

// Automatically becomes GraphQL type:
// type User {
//   id: String!
//   name: String!
//   email: String!
//   age: Int
// }
```

### Type Mapping

| Zod Type | GraphQL Type |
|----------|-------------|
| `z.string()` | `String!` |
| `z.number()` | `Float!` |
| `z.number().int()` | `Int!` |
| `z.boolean()` | `Boolean!` |
| `z.array(T)` | `[T!]!` |
| `z.optional()` | Nullable |
| `z.object()` | Type |

---

## Queries

### Simple Query

```typescript
const getPostOperation = {
  name: 'getPost',

  input: z.object({
    id: z.string(),
  }),

  output: z.object({
    id: z.string(),
    title: z.string(),
    content: z.string(),
  }),

  handler: async (input) => {
    return await db.posts.findById(input.id);
  },

  graphql: { type: 'query' },
};
```

**Query:**
```graphql
query {
  getPost(id: "123") {
    id
    title
    content
  }
}
```

### List Query

```typescript
const listPostsOperation = {
  name: 'listPosts',

  input: z.object({
    limit: z.number().int().optional(),
    offset: z.number().int().optional(),
  }),

  output: z.array(z.object({
    id: z.string(),
    title: z.string(),
  })),

  handler: async (input) => {
    return await db.posts.find(input);
  },

  graphql: { type: 'query' },
};
```

**Query:**
```graphql
query {
  listPosts(limit: 10, offset: 0) {
    id
    title
  }
}
```

### Nested Queries

```typescript
const getUserWithPostsOperation = {
  name: 'getUser',

  input: z.object({
    id: z.string(),
  }),

  output: z.object({
    id: z.string(),
    name: z.string(),
    posts: z.array(z.object({
      id: z.string(),
      title: z.string(),
    })),
  }),

  handler: async (input) => {
    const user = await db.users.findById(input.id);
    const posts = await db.posts.findByUserId(input.id);
    return { ...user, posts };
  },

  graphql: { type: 'query' },
};
```

**Query:**
```graphql
query {
  getUser(id: "123") {
    name
    posts {
      title
    }
  }
}
```

---

## Mutations

### Create Mutation

```typescript
const createUserOperation = {
  name: 'createUser',

  input: z.object({
    name: z.string().min(1),
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

  graphql: { type: 'mutation' },
};
```

**Mutation:**
```graphql
mutation {
  createUser(name: "John", email: "john@example.com") {
    id
    name
    email
  }
}
```

### Update Mutation

```typescript
const updateUserOperation = {
  name: 'updateUser',

  input: z.object({
    id: z.string(),
    name: z.string().optional(),
    email: z.string().email().optional(),
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

  graphql: { type: 'mutation' },
};
```

**Mutation:**
```graphql
mutation {
  updateUser(id: "123", name: "Jane") {
    id
    name
    email
  }
}
```

### Delete Mutation

```typescript
const deleteUserOperation = {
  name: 'deleteUser',

  input: z.object({
    id: z.string(),
  }),

  output: z.boolean(),

  handler: async (input) => {
    await db.users.delete(input.id);
    return true;
  },

  graphql: { type: 'mutation' },
};
```

**Mutation:**
```graphql
mutation {
  deleteUser(id: "123")
}
```

---

## Playground

### Accessing Playground

Navigate to: `http://localhost:3000/graphql`

The playground provides:
- Interactive query editor
- Schema documentation
- Query history
- Variable editor

### Example Session

```graphql
# Query users
query GetUsers {
  listUsers {
    id
    name
    email
  }
}

# Create user
mutation CreateUser {
  createUser(name: "John", email: "john@example.com") {
    id
    name
  }
}

# With variables
query GetUser($id: String!) {
  getUser(id: $id) {
    name
    email
  }
}

# Variables pane:
{
  "id": "123"
}
```

---

## Complete Example

```typescript
import { createApp } from 'ramapi';
import { z } from 'zod';

const app = createApp({
  protocols: {
    graphql: {
      path: '/graphql',
      playground: true,
    },
  },
});

// Define operations
const operations = [
  // Query: List users
  {
    name: 'listUsers',
    output: z.array(z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    })),
    handler: async () => await db.users.findAll(),
    graphql: { type: 'query' },
  },

  // Query: Get user
  {
    name: 'getUser',
    input: z.object({ id: z.string() }),
    output: z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }),
    handler: async (input) => await db.users.findById(input.id),
    graphql: { type: 'query' },
  },

  // Mutation: Create user
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
    handler: async (input) => await db.users.create(input),
    graphql: { type: 'mutation' },
  },

  // Mutation: Update user
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
    graphql: { type: 'mutation' },
  },
];

// Register operations
const protocolManager = app.getProtocolManager();
operations.forEach(op => protocolManager?.registerOperation(op));

await app.listen(3000);
console.log('GraphQL Playground: http://localhost:3000/graphql');
```

---

## Next Steps

- [gRPC](grpc.md)
- [REST API](rest.md)
- [Multi-Protocol Overview](overview.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
