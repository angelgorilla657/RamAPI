# Todo API Example

Complete REST API example with CRUD operations, validation, and error handling.

> **Note:** This is a documentation example showing how to use RamAPI in your own project. The code assumes you have RamAPI installed via npm. If you want to run examples from the RamAPI repository itself, see the `examples/` directory which uses relative imports.

## Prerequisites

```bash
# Install RamAPI
npm install ramapi

# Install dependencies
npm install zod better-sqlite3
npm install -D @types/better-sqlite3
```

## Overview

This example demonstrates:
- RESTful API design
- CRUD operations (Create, Read, Update, Delete)
- Request validation with Zod
- Error handling
- In-memory data storage
- TypeScript type safety

## Complete Code

### index.ts

```typescript
import { createApp, validate, logger, cors } from 'ramapi';
import { z } from 'zod';

// Types
interface Todo {
  id: string;
  title: string;
  description: string;
  completed: boolean;
  createdAt: string;
  updatedAt: string;
}

// In-memory storage
const todos: Map<string, Todo> = new Map();
let idCounter = 1;

// Helper functions
function generateId(): string {
  return `todo-${idCounter++}`;
}

function findTodo(id: string): Todo | undefined {
  return todos.get(id);
}

// Validation schemas
const createTodoSchema = z.object({
  title: z.string().min(1).max(100),
  description: z.string().max(500).optional().default(''),
  completed: z.boolean().optional().default(false),
});

const updateTodoSchema = z.object({
  title: z.string().min(1).max(100).optional(),
  description: z.string().max(500).optional(),
  completed: z.boolean().optional(),
});

const idParamSchema = z.object({
  id: z.string(),
});

const querySchema = z.object({
  completed: z.string().transform((val) => val === 'true').optional(),
  search: z.string().optional(),
});

// Create app
const app = createApp({
  cors: true,
});

// Middleware
app.use(logger());

// Routes

// GET /todos - List all todos
app.get('/todos',
  validate({ query: querySchema }),
  async (ctx) => {
    let todoList = Array.from(todos.values());

    // Filter by completed status
    if (ctx.query.completed !== undefined) {
      todoList = todoList.filter((todo) => todo.completed === ctx.query.completed);
    }

    // Search in title and description
    if (ctx.query.search) {
      const search = ctx.query.search.toLowerCase();
      todoList = todoList.filter((todo) =>
        todo.title.toLowerCase().includes(search) ||
        todo.description.toLowerCase().includes(search)
      );
    }

    ctx.json({
      todos: todoList,
      count: todoList.length,
    });
  }
);

// GET /todos/:id - Get single todo
app.get('/todos/:id',
  validate({ params: idParamSchema }),
  async (ctx) => {
    const todo = findTodo(ctx.params.id);

    if (!todo) {
      ctx.json({ error: 'Todo not found' }, 404);
      return;
    }

    ctx.json({ todo });
  }
);

// POST /todos - Create todo
app.post('/todos',
  validate({ body: createTodoSchema }),
  async (ctx) => {
    const id = generateId();
    const now = new Date().toISOString();

    const todo: Todo = {
      id,
      title: ctx.body.title,
      description: ctx.body.description,
      completed: ctx.body.completed,
      createdAt: now,
      updatedAt: now,
    };

    todos.set(id, todo);

    ctx.json({ todo }, 201);
  }
);

// PUT /todos/:id - Update todo
app.put('/todos/:id',
  validate({
    params: idParamSchema,
    body: updateTodoSchema,
  }),
  async (ctx) => {
    const todo = findTodo(ctx.params.id);

    if (!todo) {
      ctx.json({ error: 'Todo not found' }, 404);
      return;
    }

    // Update fields
    if (ctx.body.title !== undefined) {
      todo.title = ctx.body.title;
    }
    if (ctx.body.description !== undefined) {
      todo.description = ctx.body.description;
    }
    if (ctx.body.completed !== undefined) {
      todo.completed = ctx.body.completed;
    }

    todo.updatedAt = new Date().toISOString();

    todos.set(ctx.params.id, todo);

    ctx.json({ todo });
  }
);

// DELETE /todos/:id - Delete todo
app.delete('/todos/:id',
  validate({ params: idParamSchema }),
  async (ctx) => {
    const todo = findTodo(ctx.params.id);

    if (!todo) {
      ctx.json({ error: 'Todo not found' }, 404);
      return;
    }

    todos.delete(ctx.params.id);

    ctx.status(204);
  }
);

// GET /todos/stats - Get statistics
app.get('/todos/stats', async (ctx) => {
  const todoList = Array.from(todos.values());

  const stats = {
    total: todoList.length,
    completed: todoList.filter((t) => t.completed).length,
    pending: todoList.filter((t) => !t.completed).length,
  };

  ctx.json({ stats });
});

// Start server
await app.listen(3000);
console.log('📝 Todo API running at http://localhost:3000');
console.log('\nAvailable endpoints:');
console.log('  GET    /todos          - List all todos');
console.log('  GET    /todos/:id      - Get single todo');
console.log('  POST   /todos          - Create todo');
console.log('  PUT    /todos/:id      - Update todo');
console.log('  DELETE /todos/:id      - Delete todo');
console.log('  GET    /todos/stats    - Get statistics');
```

## Usage Examples

### Create Todo

```bash
curl -X POST http://localhost:3000/todos \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Learn RamAPI",
    "description": "Complete the getting started guide",
    "completed": false
  }'
```

**Response:**

```json
{
  "todo": {
    "id": "todo-1",
    "title": "Learn RamAPI",
    "description": "Complete the getting started guide",
    "completed": false,
    "createdAt": "2024-01-15T10:00:00.000Z",
    "updatedAt": "2024-01-15T10:00:00.000Z"
  }
}
```

### List All Todos

```bash
curl http://localhost:3000/todos
```

**Response:**

```json
{
  "todos": [
    {
      "id": "todo-1",
      "title": "Learn RamAPI",
      "description": "Complete the getting started guide",
      "completed": false,
      "createdAt": "2024-01-15T10:00:00.000Z",
      "updatedAt": "2024-01-15T10:00:00.000Z"
    }
  ],
  "count": 1
}
```

### Filter by Completion Status

```bash
curl http://localhost:3000/todos?completed=false
```

### Search Todos

```bash
curl "http://localhost:3000/todos?search=ramapi"
```

### Get Single Todo

```bash
curl http://localhost:3000/todos/todo-1
```

**Response:**

```json
{
  "todo": {
    "id": "todo-1",
    "title": "Learn RamAPI",
    "description": "Complete the getting started guide",
    "completed": false,
    "createdAt": "2024-01-15T10:00:00.000Z",
    "updatedAt": "2024-01-15T10:00:00.000Z"
  }
}
```

### Update Todo

```bash
curl -X PUT http://localhost:3000/todos/todo-1 \
  -H "Content-Type: application/json" \
  -d '{
    "completed": true
  }'
```

**Response:**

```json
{
  "todo": {
    "id": "todo-1",
    "title": "Learn RamAPI",
    "description": "Complete the getting started guide",
    "completed": true,
    "createdAt": "2024-01-15T10:00:00.000Z",
    "updatedAt": "2024-01-15T10:15:00.000Z"
  }
}
```

### Delete Todo

```bash
curl -X DELETE http://localhost:3000/todos/todo-1
```

**Response:** `204 No Content`

### Get Statistics

```bash
curl http://localhost:3000/todos/stats
```

**Response:**

```json
{
  "stats": {
    "total": 10,
    "completed": 7,
    "pending": 3
  }
}
```

## Error Handling

### Validation Error

```bash
curl -X POST http://localhost:3000/todos \
  -H "Content-Type: application/json" \
  -d '{"title": ""}'
```

**Response:** `400 Bad Request`

```json
{
  "error": true,
  "message": "Validation failed",
  "details": {
    "errors": [
      {
        "field": "body.title",
        "message": "String must contain at least 1 character(s)",
        "code": "too_small"
      }
    ]
  }
}
```

### Not Found Error

```bash
curl http://localhost:3000/todos/invalid-id
```

**Response:** `404 Not Found`

```json
{
  "error": "Todo not found"
}
```

## With Database (SQLite)

```typescript
import Database from 'better-sqlite3';

// Database setup
const db = new Database('todos.db');

db.exec(`
  CREATE TABLE IF NOT EXISTS todos (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    completed INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
  )
`);

// Prepared statements
const insertTodo = db.prepare(`
  INSERT INTO todos (id, title, description, completed, created_at, updated_at)
  VALUES (?, ?, ?, ?, ?, ?)
`);

const selectAllTodos = db.prepare('SELECT * FROM todos');
const selectTodoById = db.prepare('SELECT * FROM todos WHERE id = ?');
const updateTodo = db.prepare(`
  UPDATE todos
  SET title = ?, description = ?, completed = ?, updated_at = ?
  WHERE id = ?
`);
const deleteTodo = db.prepare('DELETE FROM todos WHERE id = ?');

// Update routes to use database
app.post('/todos',
  validate({ body: createTodoSchema }),
  async (ctx) => {
    const id = generateId();
    const now = new Date().toISOString();

    insertTodo.run(
      id,
      ctx.body.title,
      ctx.body.description,
      ctx.body.completed ? 1 : 0,
      now,
      now
    );

    const todo = selectTodoById.get(id);
    ctx.json({ todo }, 201);
  }
);
```

## Testing

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';

describe('Todo API', () => {
  beforeEach(() => {
    // Clear todos
    todos.clear();
    idCounter = 1;
  });

  it('should create a todo', async () => {
    const res = await request('http://localhost:3000')
      .post('/todos')
      .send({
        title: 'Test Todo',
        description: 'Test Description',
      })
      .expect(201);

    expect(res.body.todo).toMatchObject({
      id: 'todo-1',
      title: 'Test Todo',
      description: 'Test Description',
      completed: false,
    });
  });

  it('should list todos', async () => {
    // Create todos
    await request('http://localhost:3000')
      .post('/todos')
      .send({ title: 'Todo 1' });

    await request('http://localhost:3000')
      .post('/todos')
      .send({ title: 'Todo 2' });

    // List todos
    const res = await request('http://localhost:3000')
      .get('/todos')
      .expect(200);

    expect(res.body.count).toBe(2);
    expect(res.body.todos).toHaveLength(2);
  });

  it('should update a todo', async () => {
    // Create todo
    const createRes = await request('http://localhost:3000')
      .post('/todos')
      .send({ title: 'Original Title' });

    const todoId = createRes.body.todo.id;

    // Update todo
    const updateRes = await request('http://localhost:3000')
      .put(`/todos/${todoId}`)
      .send({ title: 'Updated Title', completed: true })
      .expect(200);

    expect(updateRes.body.todo.title).toBe('Updated Title');
    expect(updateRes.body.todo.completed).toBe(true);
  });

  it('should delete a todo', async () => {
    // Create todo
    const createRes = await request('http://localhost:3000')
      .post('/todos')
      .send({ title: 'To Delete' });

    const todoId = createRes.body.todo.id;

    // Delete todo
    await request('http://localhost:3000')
      .delete(`/todos/${todoId}`)
      .expect(204);

    // Verify deletion
    await request('http://localhost:3000')
      .get(`/todos/${todoId}`)
      .expect(404);
  });

  it('should filter by completion status', async () => {
    // Create todos
    await request('http://localhost:3000')
      .post('/todos')
      .send({ title: 'Completed', completed: true });

    await request('http://localhost:3000')
      .post('/todos')
      .send({ title: 'Pending', completed: false });

    // Filter completed
    const res = await request('http://localhost:3000')
      .get('/todos?completed=true')
      .expect(200);

    expect(res.body.count).toBe(1);
    expect(res.body.todos[0].completed).toBe(true);
  });
});
```

## Next Steps

- Add pagination for large datasets
- Implement user authentication
- Add due dates and priorities
- Create tags/categories system
- Add file attachments
- Implement real-time updates with WebSockets

## See Also

- [Validation Guide](../core-concepts/validation.md)
- [Error Handling](../guides/error-handling.md)
- [Testing Guide](../guides/testing.md)
