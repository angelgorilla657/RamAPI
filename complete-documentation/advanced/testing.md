# Testing Strategies

Comprehensive guide to testing RamAPI applications: unit tests, integration tests, mocking, and best practices.

> **Note**: This documentation provides testing patterns and examples verified against RamAPI's APIs:
> - ✅ Context object structure verified (method, path, params, headers, body, status(), json(), text())
> - ✅ Server methods verified (listen(), close())
> - ✅ Testing patterns are standard Node.js/TypeScript practices
> - ⚠️ Mock context helper is a conceptual example - adapt to your needs
> - ⚠️ Test examples assume Vitest/Jest but patterns work with any framework

## Table of Contents

1. [Testing Setup](#testing-setup)
2. [Unit Testing Handlers](#unit-testing-handlers)
3. [Integration Testing](#integration-testing)
4. [Mocking Strategies](#mocking-strategies)
5. [Testing Middleware](#testing-middleware)
6. [E2E Testing](#e2e-testing)
7. [Performance Testing](#performance-testing)
8. [Best Practices](#best-practices)

---

## Testing Setup

### Install Testing Dependencies

```bash
npm install -D vitest @vitest/ui
npm install -D supertest @types/supertest
```

### Vitest Configuration

**vitest.config.ts:**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules/', 'dist/', '**/*.test.ts'],
    },
  },
});
```

**package.json scripts:**

```json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage"
  }
}
```

---

## Unit Testing Handlers

### Basic Handler Tests

```typescript
import { describe, it, expect } from 'vitest';
import type { Context } from 'ramapi';

// Mock context helper
function createMockContext(overrides: Partial<Context> = {}): Context {
  return {
    method: 'GET',
    path: '/',
    params: {},
    query: {},
    headers: {},
    body: undefined,
    status: function (code: number) {
      this.statusCode = code;
    },
    json: function (data: any, status?: number) {
      if (status) this.statusCode = status;
      this.responseBody = JSON.stringify(data);
    },
    text: function (data: string, status?: number) {
      if (status) this.statusCode = status;
      this.responseBody = data;
    },
    statusCode: 200,
    responseBody: '',
    ...overrides,
  } as Context;
}

// Handler to test
function getUserHandler(ctx: Context) {
  const userId = ctx.params.id;

  if (!userId) {
    ctx.status(400);
    ctx.json({ error: 'User ID required' });
    return;
  }

  ctx.json({ id: userId, name: 'John Doe' });
}

describe('getUserHandler', () => {
  it('should return user data', () => {
    const ctx = createMockContext({
      params: { id: '123' },
    });

    getUserHandler(ctx);

    expect(ctx.statusCode).toBe(200);
    expect(JSON.parse(ctx.responseBody)).toEqual({
      id: '123',
      name: 'John Doe',
    });
  });

  it('should return 400 if user ID missing', () => {
    const ctx = createMockContext({
      params: {},
    });

    getUserHandler(ctx);

    expect(ctx.statusCode).toBe(400);
    expect(JSON.parse(ctx.responseBody)).toEqual({
      error: 'User ID required',
    });
  });
});
```

### Testing with Dependencies

```typescript
import { describe, it, expect, vi } from 'vitest';

// Service to mock
interface UserService {
  getUser(id: string): Promise<{ id: string; name: string }>;
}

// Handler with dependency
function createGetUserHandler(userService: UserService) {
  return async (ctx: Context) => {
    const userId = ctx.params.id;
    const user = await userService.getUser(userId);
    ctx.json(user);
  };
}

describe('getUserHandler with dependencies', () => {
  it('should fetch user from service', async () => {
    // Mock service
    const mockUserService: UserService = {
      getUser: vi.fn().mockResolvedValue({
        id: '123',
        name: 'John Doe',
      }),
    };

    const handler = createGetUserHandler(mockUserService);
    const ctx = createMockContext({
      params: { id: '123' },
    });

    await handler(ctx);

    expect(mockUserService.getUser).toHaveBeenCalledWith('123');
    expect(JSON.parse(ctx.responseBody)).toEqual({
      id: '123',
      name: 'John Doe',
    });
  });

  it('should handle service errors', async () => {
    const mockUserService: UserService = {
      getUser: vi.fn().mockRejectedValue(new Error('User not found')),
    };

    const handler = createGetUserHandler(mockUserService);
    const ctx = createMockContext({
      params: { id: '999' },
    });

    await expect(handler(ctx)).rejects.toThrow('User not found');
  });
});
```

---

## Integration Testing

### Using Supertest

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { createApp, type Server } from 'ramapi';

describe('API Integration Tests', () => {
  let app: Server;
  let baseURL: string;

  beforeAll(async () => {
    app = createApp();

    app.get('/users', (ctx) => {
      ctx.json({ users: [{ id: '1', name: 'John' }] });
    });

    app.get('/users/:id', (ctx) => {
      ctx.json({ id: ctx.params.id, name: 'John Doe' });
    });

    app.post('/users', (ctx) => {
      ctx.json({ id: '123', ...ctx.body }, 201);
    });

    await app.listen(3001);
    baseURL = 'http://localhost:3001';
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /users should return users list', async () => {
    const response = await request(baseURL).get('/users');

    expect(response.status).toBe(200);
    expect(response.body).toEqual({
      users: [{ id: '1', name: 'John' }],
    });
  });

  it('GET /users/:id should return single user', async () => {
    const response = await request(baseURL).get('/users/123');

    expect(response.status).toBe(200);
    expect(response.body).toEqual({
      id: '123',
      name: 'John Doe',
    });
  });

  it('POST /users should create user', async () => {
    const response = await request(baseURL)
      .post('/users')
      .send({ name: 'Jane Doe', email: 'jane@example.com' })
      .set('Content-Type', 'application/json');

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      id: '123',
      name: 'Jane Doe',
      email: 'jane@example.com',
    });
  });

  it('GET /unknown should return 404', async () => {
    const response = await request(baseURL).get('/unknown');

    expect(response.status).toBe(404);
  });
});
```

### Testing with Database

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import Database from 'better-sqlite3';

describe('User API with Database', () => {
  let app: Server;
  let db: Database.Database;
  let baseURL: string;

  beforeEach(async () => {
    // Create in-memory database
    db = new Database(':memory:');

    // Setup schema
    db.exec(`
      CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL
      )
    `);

    // Create app with database
    app = createApp();

    app.get('/users', (ctx) => {
      const users = db.prepare('SELECT * FROM users').all();
      ctx.json({ users });
    });

    app.post('/users', (ctx) => {
      const { name, email } = ctx.body as { name: string; email: string };
      const result = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)').run(name, email);
      ctx.json({ id: result.lastInsertRowid, name, email }, 201);
    });

    await app.listen(3002);
    baseURL = 'http://localhost:3002';
  });

  afterEach(async () => {
    await app.close();
    db.close();
  });

  it('should create and retrieve users', async () => {
    // Create user
    const createResponse = await request(baseURL)
      .post('/users')
      .send({ name: 'John', email: 'john@example.com' });

    expect(createResponse.status).toBe(201);

    // Retrieve users
    const getResponse = await request(baseURL).get('/users');

    expect(getResponse.status).toBe(200);
    expect(getResponse.body.users).toHaveLength(1);
    expect(getResponse.body.users[0]).toMatchObject({
      name: 'John',
      email: 'john@example.com',
    });
  });
});
```

---

## Mocking Strategies

### Mocking External APIs

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

// Mock fetch globally
global.fetch = vi.fn();

describe('External API Integration', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should fetch data from external API', async () => {
    // Mock fetch response
    (global.fetch as any).mockResolvedValueOnce({
      ok: true,
      json: async () => ({ data: 'external data' }),
    });

    const handler = async (ctx: Context) => {
      const response = await fetch('https://api.example.com/data');
      const data = await response.json();
      ctx.json(data);
    };

    const ctx = createMockContext();
    await handler(ctx);

    expect(global.fetch).toHaveBeenCalledWith('https://api.example.com/data');
    expect(JSON.parse(ctx.responseBody)).toEqual({ data: 'external data' });
  });

  it('should handle external API errors', async () => {
    (global.fetch as any).mockRejectedValueOnce(new Error('Network error'));

    const handler = async (ctx: Context) => {
      try {
        await fetch('https://api.example.com/data');
      } catch (error: any) {
        ctx.status(500);
        ctx.json({ error: error.message });
      }
    };

    const ctx = createMockContext();
    await handler(ctx);

    expect(ctx.statusCode).toBe(500);
    expect(JSON.parse(ctx.responseBody)).toEqual({ error: 'Network error' });
  });
});
```

### Mocking Database

```typescript
import { describe, it, expect, vi } from 'vitest';

interface Database {
  query(sql: string, params: any[]): Promise<any[]>;
}

describe('Database Mocking', () => {
  it('should query database', async () => {
    // Mock database
    const mockDb: Database = {
      query: vi.fn().mockResolvedValue([
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' },
      ]),
    };

    const handler = async (ctx: Context) => {
      const users = await mockDb.query('SELECT * FROM users', []);
      ctx.json({ users });
    };

    const ctx = createMockContext();
    await handler(ctx);

    expect(mockDb.query).toHaveBeenCalledWith('SELECT * FROM users', []);
    expect(JSON.parse(ctx.responseBody)).toEqual({
      users: [
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' },
      ],
    });
  });
});
```

---

## Testing Middleware

### Testing Authentication Middleware

```typescript
import { describe, it, expect } from 'vitest';
import { JWTService, authenticate } from 'ramapi';

describe('Authentication Middleware', () => {
  const jwtService = new JWTService({ secret: 'test-secret' });
  const auth = authenticate(jwtService);

  it('should authenticate valid token', async () => {
    const token = jwtService.sign({ sub: '123', role: 'user' });

    const ctx = createMockContext({
      headers: {
        authorization: `Bearer ${token}`,
      },
    });

    let nextCalled = false;
    const next = async () => {
      nextCalled = true;
    };

    await auth(ctx, next);

    expect(nextCalled).toBe(true);
    expect(ctx.user).toEqual({ sub: '123', role: 'user' });
  });

  it('should reject invalid token', async () => {
    const ctx = createMockContext({
      headers: {
        authorization: 'Bearer invalid-token',
      },
    });

    const next = async () => {};

    await expect(auth(ctx, next)).rejects.toThrow();
  });

  it('should reject missing token', async () => {
    const ctx = createMockContext({
      headers: {},
    });

    const next = async () => {};

    await expect(auth(ctx, next)).rejects.toThrow('Authorization header missing');
  });
});
```

### Testing Custom Middleware

```typescript
import { describe, it, expect } from 'vitest';
import type { Middleware } from 'ramapi';

function rateLimitMiddleware(max: number): Middleware {
  const requests = new Map<string, number>();

  return async (ctx, next) => {
    const ip = ctx.headers['x-forwarded-for'] || 'unknown';
    const count = requests.get(ip) || 0;

    if (count >= max) {
      ctx.status(429);
      ctx.json({ error: 'Too many requests' });
      return;
    }

    requests.set(ip, count + 1);
    await next();
  };
}

describe('Rate Limit Middleware', () => {
  it('should allow requests within limit', async () => {
    const middleware = rateLimitMiddleware(2);
    const ctx = createMockContext({
      headers: { 'x-forwarded-for': '127.0.0.1' },
    });

    let nextCalled = false;
    const next = async () => {
      nextCalled = true;
    };

    await middleware(ctx, next);

    expect(nextCalled).toBe(true);
  });

  it('should block requests over limit', async () => {
    const middleware = rateLimitMiddleware(2);
    const ctx = createMockContext({
      headers: { 'x-forwarded-for': '127.0.0.1' },
    });

    const next = async () => {};

    // First two requests should pass
    await middleware(ctx, next);
    await middleware(ctx, next);

    // Third request should be blocked
    await middleware(ctx, next);

    expect(ctx.statusCode).toBe(429);
    expect(JSON.parse(ctx.responseBody)).toEqual({ error: 'Too many requests' });
  });
});
```

---

## E2E Testing

### Complete User Flow

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';

describe('E2E: User Registration and Login', () => {
  let app: Server;
  let baseURL: string;

  beforeAll(async () => {
    app = createApp();
    // Setup routes...
    await app.listen(3003);
    baseURL = 'http://localhost:3003';
  });

  afterAll(async () => {
    await app.close();
  });

  it('should complete full user flow', async () => {
    // 1. Register user
    const registerResponse = await request(baseURL)
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      });

    expect(registerResponse.status).toBe(201);
    expect(registerResponse.body).toHaveProperty('token');

    const token = registerResponse.body.token;

    // 2. Login
    const loginResponse = await request(baseURL)
      .post('/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123',
      });

    expect(loginResponse.status).toBe(200);
    expect(loginResponse.body).toHaveProperty('token');

    // 3. Access protected route
    const profileResponse = await request(baseURL)
      .get('/auth/profile')
      .set('Authorization', `Bearer ${token}`);

    expect(profileResponse.status).toBe(200);
    expect(profileResponse.body).toMatchObject({
      email: 'test@example.com',
      name: 'Test User',
    });

    // 4. Update profile
    const updateResponse = await request(baseURL)
      .put('/auth/profile')
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'Updated Name' });

    expect(updateResponse.status).toBe(200);

    // 5. Verify update
    const verifyResponse = await request(baseURL)
      .get('/auth/profile')
      .set('Authorization', `Bearer ${token}`);

    expect(verifyResponse.body.name).toBe('Updated Name');
  });
});
```

---

## Performance Testing

### Load Testing with Autocannon

```bash
npm install -D autocannon
```

```typescript
import { describe, it } from 'vitest';
import autocannon from 'autocannon';

describe('Performance Tests', () => {
  it('should handle 10k requests', async () => {
    const result = await autocannon({
      url: 'http://localhost:3000',
      connections: 10,
      duration: 10,
    });

    console.log('Requests/sec:', result.requests.average);
    console.log('Latency (ms):', result.latency.mean);

    // Assert performance thresholds
    expect(result.requests.average).toBeGreaterThan(1000);
    expect(result.latency.mean).toBeLessThan(100);
  }, 30000);
});
```

### Benchmark Tests

```typescript
import { describe, it } from 'vitest';

describe('Benchmarks', () => {
  it('should process requests quickly', async () => {
    const iterations = 10000;
    const start = Date.now();

    const ctx = createMockContext();

    for (let i = 0; i < iterations; i++) {
      ctx.json({ result: i });
    }

    const duration = Date.now() - start;
    const opsPerSecond = (iterations / duration) * 1000;

    console.log(`Operations/sec: ${opsPerSecond.toFixed(0)}`);

    expect(opsPerSecond).toBeGreaterThan(10000);
  });
});
```

---

## Best Practices

### 1. Test Organization

```typescript
// Good: Organize by feature
src/
  users/
    users.handler.ts
    users.handler.test.ts
    users.service.ts
    users.service.test.ts
  posts/
    posts.handler.ts
    posts.handler.test.ts

// Bad: Separate test directory
src/
  users/
    users.handler.ts
    users.service.ts
tests/
  users.test.ts
```

### 2. Test Naming

```typescript
// Good: Descriptive test names
describe('getUserHandler', () => {
  it('should return 200 with user data when user exists', () => {});
  it('should return 404 when user not found', () => {});
  it('should return 400 when user ID is invalid', () => {});
});

// Bad: Vague test names
describe('getUserHandler', () => {
  it('works', () => {});
  it('handles errors', () => {});
});
```

### 3. Test Independence

```typescript
// Good: Independent tests
describe('User API', () => {
  beforeEach(() => {
    // Fresh setup for each test
    db = createTestDatabase();
  });

  afterEach(() => {
    db.close();
  });

  it('should create user', async () => {});
  it('should update user', async () => {});
});

// Bad: Dependent tests
describe('User API', () => {
  let userId: string;

  it('should create user', async () => {
    userId = await createUser();
  });

  it('should update user', async () => {
    // Depends on previous test
    await updateUser(userId);
  });
});
```

### 4. Use Test Fixtures

```typescript
// fixtures/users.ts
export const testUsers = {
  john: {
    id: '1',
    name: 'John Doe',
    email: 'john@example.com',
  },
  jane: {
    id: '2',
    name: 'Jane Doe',
    email: 'jane@example.com',
  },
};

// users.test.ts
import { testUsers } from './fixtures/users.js';

it('should return user', () => {
  const user = getUser('1');
  expect(user).toEqual(testUsers.john);
});
```

---

## See Also

- [Advanced Routing](advanced-routing.md)
- [Custom Adapters](custom-adapters.md)
- [Security Best Practices](security.md)
- [Performance Optimization](../performance/optimization.md)
