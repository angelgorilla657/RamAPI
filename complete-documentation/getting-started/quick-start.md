# Quick Start Guide

Get up and running with RamAPI in minutes. This guide walks you through creating a fully functional API with routing, validation, and authentication.

## Prerequisites

Make sure you've completed the [Installation](installation.md) guide.

## Hello World

Let's start with the simplest possible RamAPI application.

### Create Your Server

Create `src/index.ts`:

```typescript
import { createApp } from 'ramapi';

const app = createApp();

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello, RamAPI!' });
});

app.listen(3000, () => {
  console.log('🚀 Server running on http://localhost:3000');
});
```

### Run It

```bash
npm run dev
```

### Test It

```bash
curl http://localhost:3000
```

Response:

```json
{
  "message": "Hello, RamAPI!"
}
```

## Basic Routing

Let's add more routes with different HTTP methods.

```typescript
import { createApp } from 'ramapi';

const app = createApp();

// GET request
app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello, RamAPI!' });
});

// GET with route parameters
app.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  ctx.json({ userId, name: 'John Doe' });
});

// POST request
app.post('/users', async (ctx) => {
  const userData = ctx.body;
  ctx.json({ message: 'User created', data: userData }, 201);
});

// PUT request
app.put('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  const updates = ctx.body;
  ctx.json({ message: 'User updated', userId, updates });
});

// DELETE request
app.delete('/users/:id', async (ctx) => {
  const userId = ctx.params.id;
  ctx.json({ message: 'User deleted', userId });
});

app.listen(3000);
```

### Test the Routes

```bash
# GET with parameter
curl http://localhost:3000/users/123

# POST with body
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John", "email": "john@example.com"}'

# PUT with parameter and body
curl -X PUT http://localhost:3000/users/123 \
  -H "Content-Type: application/json" \
  -d '{"name": "Jane"}'

# DELETE
curl -X DELETE http://localhost:3000/users/123
```

## Query Parameters

Access query string parameters:

```typescript
app.get('/search', async (ctx) => {
  const { q, page = '1', limit = '10' } = ctx.query;

  ctx.json({
    query: q,
    page: parseInt(page),
    limit: parseInt(limit),
    results: []
  });
});
```

Test it:

```bash
curl "http://localhost:3000/search?q=typescript&page=2&limit=20"
```

## Request Validation

Add type-safe validation with Zod schemas.

```typescript
import { createApp, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

// Define validation schema
const createUserSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  age: z.number().int().min(18, 'Must be at least 18 years old'),
});

// Apply validation middleware
app.post('/users',
  validate({ body: createUserSchema }),
  async (ctx) => {
    // ctx.body is now validated and typed!
    const user = ctx.body as z.infer<typeof createUserSchema>;

    ctx.json({
      message: 'User created successfully',
      user
    }, 201);
  }
);

app.listen(3000);
```

### Test Validation

Valid request:

```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John", "email": "john@example.com", "age": 25}'
```

Invalid request (triggers validation error):

```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "J", "email": "invalid", "age": 15}'
```

Response:

```json
{
  "error": true,
  "message": "Validation failed",
  "details": {
    "errors": [
      {
        "field": "body.name",
        "message": "Name must be at least 2 characters"
      },
      {
        "field": "body.email",
        "message": "Invalid email address"
      },
      {
        "field": "body.age",
        "message": "Must be at least 18 years old"
      }
    ]
  }
}
```

## Middleware

Add request logging middleware:

```typescript
import { createApp, logger } from 'ramapi';

const app = createApp();

// Add logger middleware globally
app.use(logger());

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello!' });
});

app.listen(3000);
```

Output:

```
[200] GET / - 2ms
```

## CORS Configuration

Enable Cross-Origin Resource Sharing:

```typescript
import { createApp, cors } from 'ramapi';

const app = createApp();

// Enable CORS for all routes
app.use(cors({
  origin: ['http://localhost:3001', 'https://myapp.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,
}));

app.get('/api/data', async (ctx) => {
  ctx.json({ data: 'Available via CORS' });
});

app.listen(3000);
```

## Rate Limiting

Protect your API from abuse:

```typescript
import { createApp, rateLimit } from 'ramapi';

const app = createApp();

// Global rate limit: 100 requests per minute
app.use(rateLimit({
  maxRequests: 100,
  windowMs: 60000, // 1 minute
}));

// Stricter rate limit for specific route
app.post('/api/expensive',
  rateLimit({ maxRequests: 5, windowMs: 60000 }),
  async (ctx) => {
    ctx.json({ message: 'Expensive operation completed' });
  }
);

app.listen(3000);
```

## Authentication

Add JWT-based authentication:

```typescript
import { createApp, JWTService, authenticate, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

// Initialize JWT service
const jwt = new JWTService({
  secret: process.env.JWT_SECRET || 'your-secret-key',
  expiresIn: 86400, // 24 hours
});

// Login schema
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});

// Public route: Login
app.post('/login',
  validate({ body: loginSchema }),
  async (ctx) => {
    const { email, password } = ctx.body as z.infer<typeof loginSchema>;

    // TODO: Verify credentials (simplified for example)
    if (email === 'user@example.com' && password === 'password123') {
      const token = jwt.sign({ sub: '123', email });

      ctx.json({ token });
    } else {
      ctx.status(401).json({ error: 'Invalid credentials' });
    }
  }
);

// Protected route: Profile
app.get('/profile',
  authenticate(jwt),
  async (ctx) => {
    // ctx.user contains decoded JWT payload
    ctx.json({
      user: ctx.user,
      message: 'This is a protected route'
    });
  }
);

app.listen(3000);
```

### Test Authentication

Login:

```bash
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "password123"}'
```

Response:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

Access protected route:

```bash
curl http://localhost:3000/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

Response:

```json
{
  "user": {
    "sub": "123",
    "email": "user@example.com"
  },
  "message": "This is a protected route"
}
```

## Route Groups

Organize routes with shared prefixes and middleware:

```typescript
import { createApp, authenticate, logger } from 'ramapi';

const app = createApp();
const jwt = new JWTService({ secret: 'secret' });

// Global middleware
app.use(logger());

// API v1 group
app.group('/api/v1', (api) => {

  // Public routes
  api.post('/register', registerHandler);
  api.post('/login', loginHandler);

  // Protected user routes
  api.group('/users', (users) => {
    users.use(authenticate(jwt)); // Auth required for all user routes

    users.get('/', listUsers);           // GET /api/v1/users
    users.get('/:id', getUser);          // GET /api/v1/users/:id
    users.post('/', createUser);         // POST /api/v1/users
    users.put('/:id', updateUser);       // PUT /api/v1/users/:id
    users.delete('/:id', deleteUser);    // DELETE /api/v1/users/:id
  });

  // Protected posts routes
  api.group('/posts', (posts) => {
    posts.use(authenticate(jwt));

    posts.get('/', listPosts);           // GET /api/v1/posts
    posts.post('/', createPost);         // POST /api/v1/posts
  });
});

app.listen(3000);
```

## Error Handling

Handle errors gracefully:

```typescript
import { createApp, HTTPError } from 'ramapi';

const app = createApp({
  // Custom error handler
  onError: async (error, ctx) => {
    console.error('Error:', error);

    if (error instanceof HTTPError) {
      ctx.json({
        error: true,
        message: error.message,
        details: error.details
      }, error.statusCode);
    } else {
      ctx.json({
        error: true,
        message: 'Internal server error'
      }, 500);
    }
  }
});

app.get('/users/:id', async (ctx) => {
  const userId = ctx.params.id;

  // Simulate database lookup
  const user = await findUser(userId);

  if (!user) {
    throw new HTTPError(404, 'User not found');
  }

  ctx.json(user);
});

app.listen(3000);
```

## Complete Example

Here's a complete example combining everything:

```typescript
import { createApp, validate, authenticate, JWTService, logger, cors, rateLimit } from 'ramapi';
import { z } from 'zod';

// Initialize app with configuration
const app = createApp({
  port: 3000,
  host: '0.0.0.0',
});

// JWT service
const jwt = new JWTService({
  secret: process.env.JWT_SECRET || 'your-secret-key',
  expiresIn: 86400,
});

// Global middleware
app.use(logger());
app.use(cors({ origin: '*' }));
app.use(rateLimit({ maxRequests: 100, windowMs: 60000 }));

// Schemas
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});

const createPostSchema = z.object({
  title: z.string().min(5),
  content: z.string().min(10),
});

// Public routes
app.get('/', async (ctx) => {
  ctx.json({ message: 'Welcome to RamAPI!' });
});

app.post('/login',
  validate({ body: loginSchema }),
  async (ctx) => {
    const { email, password } = ctx.body as z.infer<typeof loginSchema>;

    // Verify credentials (simplified)
    if (email && password) {
      const token = jwt.sign({ sub: '123', email });
      ctx.json({ token });
    } else {
      ctx.status(401).json({ error: 'Invalid credentials' });
    }
  }
);

// Protected routes
app.group('/api', (api) => {
  api.use(authenticate(jwt));

  api.get('/profile', async (ctx) => {
    ctx.json({ user: ctx.user });
  });

  api.post('/posts',
    validate({ body: createPostSchema }),
    async (ctx) => {
      const post = ctx.body as z.infer<typeof createPostSchema>;
      ctx.json({ message: 'Post created', post }, 201);
    }
  );
});

// Start server
app.listen(3000, () => {
  console.log('🚀 Server running on http://localhost:3000');
});
```

## Next Steps

Now that you understand the basics:

1. [Learn about project structure](project-structure.md) for organizing larger applications
2. [Explore core concepts](../core-concepts/context-and-handlers.md) in depth
3. [Check out complete examples](../examples/todo-api.md)
4. [Add observability features](../observability/overview.md)
5. [Learn about multi-protocol support](../protocols/overview.md)

## Useful Commands

```bash
# Development with hot reload
npm run dev

# Build for production
npm run build

# Run production build
npm start

# Type checking
npm run type-check
```

## Tips

- Use `validate()` middleware for all input validation
- Always use `authenticate()` for protected routes
- Leverage route groups for organization
- Add logging middleware early in development
- Use TypeScript for better type safety
- Handle errors with custom error handlers

## Common Patterns

### JSON Response Helper

```typescript
app.get('/data', async (ctx) => {
  ctx.json({ key: 'value' }, 200); // status code optional
});
```

### Text Response

```typescript
app.get('/health', async (ctx) => {
  ctx.text('OK');
});
```

### Status Code

```typescript
app.post('/resource', async (ctx) => {
  ctx.status(201).json({ created: true });
});
```

### Custom Headers

```typescript
app.get('/download', async (ctx) => {
  ctx.setHeader('Content-Disposition', 'attachment; filename="file.txt"');
  ctx.text('File content');
});
```

---

**Ready to structure your project?** Continue to [Project Structure](project-structure.md)
