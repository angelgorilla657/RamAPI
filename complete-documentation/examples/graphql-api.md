# GraphQL API Example

Complete GraphQL server example with queries, mutations, and GraphQL Playground.

> **Note:** This is a documentation example showing how to use RamAPI in your own project. The code assumes you have RamAPI installed via npm.

## Prerequisites

```bash
# Install RamAPI
npm install ramapi

# Install dependencies
npm install graphql better-sqlite3
npm install -D @types/better-sqlite3
```

## Overview

This example demonstrates:
- GraphQL schema definition
- Queries and mutations
- Context and authentication
- GraphQL Playground
- Error handling
- TypeScript integration

## Complete Code

### index.ts

```typescript
import { createApp } from 'ramapi';
import { buildSchema } from 'graphql';
import Database from 'better-sqlite3';

// Database setup
const db = new Database('graphql.db');

db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    age INTEGER,
    created_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS posts (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    author_id TEXT NOT NULL,
    published INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    FOREIGN KEY (author_id) REFERENCES users(id)
  );
`);

// GraphQL Schema
const schema = buildSchema(`
  type User {
    id: ID!
    name: String!
    email: String!
    age: Int
    posts: [Post!]!
    createdAt: String!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    published: Boolean!
    createdAt: String!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
    posts(published: Boolean): [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    createUser(name: String!, email: String!, age: Int): User!
    updateUser(id: ID!, name: String, email: String, age: Int): User!
    deleteUser(id: ID!): Boolean!

    createPost(title: String!, content: String!, authorId: ID!, published: Boolean): Post!
    updatePost(id: ID!, title: String, content: String, published: Boolean): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!
  }
`);

// Helper functions
function generateId(prefix: string): string {
  return `${prefix}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

// Resolvers
const root = {
  // Queries
  users: () => {
    return db.prepare('SELECT * FROM users').all();
  },

  user: ({ id }: { id: string }) => {
    const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
    if (!user) throw new Error('User not found');
    return user;
  },

  posts: ({ published }: { published?: boolean }) => {
    if (published !== undefined) {
      return db.prepare('SELECT * FROM posts WHERE published = ?')
        .all(published ? 1 : 0);
    }
    return db.prepare('SELECT * FROM posts').all();
  },

  post: ({ id }: { id: string }) => {
    const post = db.prepare('SELECT * FROM posts WHERE id = ?').get(id);
    if (!post) throw new Error('Post not found');
    return post;
  },

  // Mutations
  createUser: ({ name, email, age }: { name: string; email: string; age?: number }) => {
    const id = generateId('user');
    const createdAt = new Date().toISOString();

    try {
      db.prepare(`
        INSERT INTO users (id, name, email, age, created_at)
        VALUES (?, ?, ?, ?, ?)
      `).run(id, name, email, age || null, createdAt);

      return db.prepare('SELECT * FROM users WHERE id = ?').get(id);
    } catch (error: any) {
      if (error.message.includes('UNIQUE constraint failed')) {
        throw new Error('Email already exists');
      }
      throw error;
    }
  },

  updateUser: ({ id, name, email, age }: any) => {
    const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
    if (!user) throw new Error('User not found');

    const updates: string[] = [];
    const values: any[] = [];

    if (name !== undefined) {
      updates.push('name = ?');
      values.push(name);
    }
    if (email !== undefined) {
      updates.push('email = ?');
      values.push(email);
    }
    if (age !== undefined) {
      updates.push('age = ?');
      values.push(age);
    }

    if (updates.length > 0) {
      values.push(id);
      db.prepare(`UPDATE users SET ${updates.join(', ')} WHERE id = ?`).run(...values);
    }

    return db.prepare('SELECT * FROM users WHERE id = ?').get(id);
  },

  deleteUser: ({ id }: { id: string }) => {
    const result = db.prepare('DELETE FROM users WHERE id = ?').run(id);
    return result.changes > 0;
  },

  createPost: ({ title, content, authorId, published }: any) => {
    const author = db.prepare('SELECT * FROM users WHERE id = ?').get(authorId);
    if (!author) throw new Error('Author not found');

    const id = generateId('post');
    const createdAt = new Date().toISOString();

    db.prepare(`
      INSERT INTO posts (id, title, content, author_id, published, created_at)
      VALUES (?, ?, ?, ?, ?, ?)
    `).run(id, title, content, authorId, published ? 1 : 0, createdAt);

    return db.prepare('SELECT * FROM posts WHERE id = ?').get(id);
  },

  updatePost: ({ id, title, content, published }: any) => {
    const post = db.prepare('SELECT * FROM posts WHERE id = ?').get(id);
    if (!post) throw new Error('Post not found');

    const updates: string[] = [];
    const values: any[] = [];

    if (title !== undefined) {
      updates.push('title = ?');
      values.push(title);
    }
    if (content !== undefined) {
      updates.push('content = ?');
      values.push(content);
    }
    if (published !== undefined) {
      updates.push('published = ?');
      values.push(published ? 1 : 0);
    }

    if (updates.length > 0) {
      values.push(id);
      db.prepare(`UPDATE posts SET ${updates.join(', ')} WHERE id = ?`).run(...values);
    }

    return db.prepare('SELECT * FROM posts WHERE id = ?').get(id);
  },

  deletePost: ({ id }: { id: string }) => {
    const result = db.prepare('DELETE FROM posts WHERE id = ?').run(id);
    return result.changes > 0;
  },

  publishPost: ({ id }: { id: string }) => {
    db.prepare('UPDATE posts SET published = 1 WHERE id = ?').run(id);
    return db.prepare('SELECT * FROM posts WHERE id = ?').get(id);
  },
};

// Field resolvers for nested data
const fieldResolvers = {
  User: {
    posts: (user: any) => {
      return db.prepare('SELECT * FROM posts WHERE author_id = ?').all(user.id);
    },
  },
  Post: {
    author: (post: any) => {
      return db.prepare('SELECT * FROM users WHERE id = ?').get(post.author_id);
    },
    published: (post: any) => {
      return post.published === 1;
    },
  },
};

// Create app
const app = createApp({
  protocols: {
    graphql: {
      path: '/graphql',
      schema,
      rootValue: { ...root, ...fieldResolvers },
      playground: true,
      introspection: true,
    },
  },
});

// REST endpoints (optional - for comparison)
app.get('/api/users', async (ctx) => {
  const users = db.prepare('SELECT * FROM users').all();
  ctx.json({ users });
});

// Start server
await app.listen(3000);
console.log('🚀 GraphQL API running at http://localhost:3000/graphql');
console.log('📊 GraphQL Playground available at http://localhost:3000/graphql');
```

## GraphQL Queries

### Query All Users

```graphql
query {
  users {
    id
    name
    email
    age
    posts {
      id
      title
      published
    }
  }
}
```

**Response:**

```json
{
  "data": {
    "users": [
      {
        "id": "user-1",
        "name": "Alice Smith",
        "email": "alice@example.com",
        "age": 30,
        "posts": [
          {
            "id": "post-1",
            "title": "My First Post",
            "published": true
          }
        ]
      }
    ]
  }
}
```

### Query Single User

```graphql
query {
  user(id: "user-1") {
    id
    name
    email
    posts {
      title
      content
      createdAt
    }
  }
}
```

### Query Posts

```graphql
query {
  posts(published: true) {
    id
    title
    content
    author {
      name
      email
    }
    createdAt
  }
}
```

## GraphQL Mutations

### Create User

```graphql
mutation {
  createUser(
    name: "Alice Smith"
    email: "alice@example.com"
    age: 30
  ) {
    id
    name
    email
    age
    createdAt
  }
}
```

**Response:**

```json
{
  "data": {
    "createUser": {
      "id": "user-1705320000000-abc123",
      "name": "Alice Smith",
      "email": "alice@example.com",
      "age": 30,
      "createdAt": "2024-01-15T10:00:00.000Z"
    }
  }
}
```

### Update User

```graphql
mutation {
  updateUser(
    id: "user-1"
    name: "Alice Johnson"
  ) {
    id
    name
    email
  }
}
```

### Create Post

```graphql
mutation {
  createPost(
    title: "GraphQL with RamAPI"
    content: "Learn how to build GraphQL APIs with RamAPI"
    authorId: "user-1"
    published: true
  ) {
    id
    title
    content
    author {
      name
    }
    published
    createdAt
  }
}
```

### Publish Post

```graphql
mutation {
  publishPost(id: "post-1") {
    id
    title
    published
  }
}
```

### Delete Post

```graphql
mutation {
  deletePost(id: "post-1")
}
```

## With Variables

```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    id
    name
    email
    posts {
      title
      published
    }
  }
}
```

**Variables:**

```json
{
  "userId": "user-1"
}
```

## Frontend Integration (Apollo Client)

```typescript
// apollo-client.ts
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

export const client = new ApolloClient({
  uri: 'http://localhost:3000/graphql',
  cache: new InMemoryCache(),
});

// Queries
export const GET_USERS = gql`
  query GetUsers {
    users {
      id
      name
      email
      posts {
        id
        title
      }
    }
  }
`;

export const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
      age
      posts {
        id
        title
        content
        published
      }
    }
  }
`;

// Mutations
export const CREATE_USER = gql`
  mutation CreateUser($name: String!, $email: String!, $age: Int) {
    createUser(name: $name, email: $email, age: $age) {
      id
      name
      email
    }
  }
`;

export const CREATE_POST = gql`
  mutation CreatePost(
    $title: String!
    $content: String!
    $authorId: ID!
    $published: Boolean
  ) {
    createPost(
      title: $title
      content: $content
      authorId: $authorId
      published: $published
    ) {
      id
      title
      content
      published
    }
  }
`;
```

```tsx
// UserList.tsx
import { useQuery, useMutation } from '@apollo/client';
import { GET_USERS, CREATE_USER } from './apollo-client';

export function UserList() {
  const { loading, error, data } = useQuery(GET_USERS);
  const [createUser] = useMutation(CREATE_USER, {
    refetchQueries: [{ query: GET_USERS }],
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  const handleCreate = async () => {
    await createUser({
      variables: {
        name: 'New User',
        email: 'new@example.com',
        age: 25,
      },
    });
  };

  return (
    <div>
      <button onClick={handleCreate}>Create User</button>
      <ul>
        {data.users.map((user: any) => (
          <li key={user.id}>
            {user.name} - {user.email}
            <ul>
              {user.posts.map((post: any) => (
                <li key={post.id}>{post.title}</li>
              ))}
            </ul>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## cURL Examples

### Query

```bash
curl -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ users { id name email } }"
  }'
```

### Mutation

```bash
curl -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation { createUser(name: \"Alice\", email: \"alice@example.com\") { id name } }"
  }'
```

### With Variables

```bash
curl -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
    "variables": { "id": "user-1" }
  }'
```

## Error Handling

GraphQL errors are returned in the `errors` array:

```json
{
  "errors": [
    {
      "message": "User not found",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["user"]
    }
  ],
  "data": {
    "user": null
  }
}
```

## See Also

- [GraphQL Guide](../multi-protocol/graphql.md)
- [Multi-Protocol Example](multi-protocol.md)
- [GraphQL Documentation](https://graphql.org)
