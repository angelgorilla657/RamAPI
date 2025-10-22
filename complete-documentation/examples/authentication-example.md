# Authentication Example

Complete authentication system with user registration, login, and protected routes using JWT and bcrypt.

> **Note:** This is a documentation example showing how to use RamAPI in your own project. The code assumes you have RamAPI installed via npm.

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
- User registration with password hashing
- Login with JWT token generation
- Protected routes with authentication middleware
- Password verification
- Token refresh
- Role-based access control (RBAC)

## Complete Code

### index.ts

```typescript
import { createApp, JWTService, passwordService, validate, logger, cors } from 'ramapi';
import { z } from 'zod';
import Database from 'better-sqlite3';

// Database setup
const db = new Database('auth.db');

db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT DEFAULT 'user',
    created_at TEXT NOT NULL
  )
`);

// Types
interface User {
  id: string;
  email: string;
  name: string;
  role: string;
  created_at: string;
}

// JWT Service
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET || 'your-secret-key-change-in-production',
  expiresIn: 86400, // 24 hours
  issuer: 'todo-api',
});

// Helper functions
function generateUserId(): string {
  return `user-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

// Validation schemas
const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(2).max(100),
});

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string(),
});

const updateProfileSchema = z.object({
  name: z.string().min(2).max(100).optional(),
});

// Create app
const app = createApp({
  cors: {
    origin: 'http://localhost:5173',
    credentials: true,
  },
});

app.use(logger());

// Authentication middleware
const authenticate = (jwtService: JWTService) => {
  return async (ctx, next) => {
    const authHeader = ctx.headers.authorization;

    if (!authHeader) {
      ctx.json({ error: 'Authorization header missing' }, 401);
      return;
    }

    const parts = authHeader.split(' ');
    if (parts.length !== 2 || parts[0] !== 'Bearer') {
      ctx.json({ error: 'Invalid authorization format' }, 401);
      return;
    }

    try {
      const payload = jwtService.verify(parts[1]);
      ctx.user = payload;
      ctx.state.userId = payload.sub;
      await next();
    } catch (error) {
      ctx.json({ error: 'Invalid or expired token' }, 401);
    }
  };
};

// Role check middleware
const requireRole = (role: string) => {
  return async (ctx, next) => {
    if (!ctx.user || ctx.user.role !== role) {
      ctx.json({ error: 'Forbidden' }, 403);
      return;
    }
    await next();
  };
};

const auth = authenticate(jwtService);

// Routes

// POST /auth/register - Register new user
app.post('/auth/register',
  validate({ body: registerSchema }),
  async (ctx) => {
    const { email, password, name } = ctx.body;

    // Check if user exists
    const existing = db.prepare('SELECT id FROM users WHERE email = ?').get(email);
    if (existing) {
      ctx.json({ error: 'Email already registered' }, 409);
      return;
    }

    // Hash password
    const hashedPassword = await passwordService.hash(password);

    // Create user
    const userId = generateUserId();
    const createdAt = new Date().toISOString();

    db.prepare(`
      INSERT INTO users (id, email, password, name, role, created_at)
      VALUES (?, ?, ?, ?, 'user', ?)
    `).run(userId, email, hashedPassword, name, createdAt);

    // Generate token
    const token = jwtService.sign({
      sub: userId,
      email,
      name,
      role: 'user',
    });

    ctx.json({
      user: {
        id: userId,
        email,
        name,
        role: 'user',
      },
      token,
    }, 201);
  }
);

// POST /auth/login - Login user
app.post('/auth/login',
  validate({ body: loginSchema }),
  async (ctx) => {
    const { email, password } = ctx.body;

    // Get user
    const user = db.prepare(`
      SELECT id, email, password, name, role
      FROM users
      WHERE email = ?
    `).get(email) as any;

    if (!user) {
      ctx.json({ error: 'Invalid credentials' }, 401);
      return;
    }

    // Verify password
    const isValid = await passwordService.verify(password, user.password);
    if (!isValid) {
      ctx.json({ error: 'Invalid credentials' }, 401);
      return;
    }

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
    });

    ctx.json({
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
      },
      token,
    });
  }
);

// GET /auth/me - Get current user
app.get('/auth/me', auth, async (ctx) => {
  const user = db.prepare(`
    SELECT id, email, name, role, created_at
    FROM users
    WHERE id = ?
  `).get(ctx.state.userId) as any;

  if (!user) {
    ctx.json({ error: 'User not found' }, 404);
    return;
  }

  ctx.json({ user });
});

// PUT /auth/profile - Update profile
app.put('/auth/profile',
  auth,
  validate({ body: updateProfileSchema }),
  async (ctx) => {
    if (ctx.body.name) {
      db.prepare('UPDATE users SET name = ? WHERE id = ?')
        .run(ctx.body.name, ctx.state.userId);
    }

    const user = db.prepare(`
      SELECT id, email, name, role
      FROM users
      WHERE id = ?
    `).get(ctx.state.userId);

    ctx.json({ user });
  }
);

// POST /auth/refresh - Refresh token
app.post('/auth/refresh', auth, async (ctx) => {
  // Generate new token with same payload
  const token = jwtService.sign({
    sub: ctx.user.sub,
    email: ctx.user.email,
    name: ctx.user.name,
    role: ctx.user.role,
  });

  ctx.json({ token });
});

// Admin routes (RBAC example)
app.get('/admin/users',
  auth,
  requireRole('admin'),
  async (ctx) => {
    const users = db.prepare(`
      SELECT id, email, name, role, created_at
      FROM users
    `).all();

    ctx.json({ users });
  }
);

app.delete('/admin/users/:id',
  auth,
  requireRole('admin'),
  async (ctx) => {
    // Don't allow deleting yourself
    if (ctx.params.id === ctx.state.userId) {
      ctx.json({ error: 'Cannot delete your own account' }, 400);
      return;
    }

    db.prepare('DELETE FROM users WHERE id = ?').run(ctx.params.id);
    ctx.status(204);
  }
);

// Start server
await app.listen(3000);
console.log('🔐 Auth API running at http://localhost:3000');
console.log('\nAvailable endpoints:');
console.log('  POST   /auth/register  - Register new user');
console.log('  POST   /auth/login     - Login user');
console.log('  GET    /auth/me        - Get current user (protected)');
console.log('  PUT    /auth/profile   - Update profile (protected)');
console.log('  POST   /auth/refresh   - Refresh token (protected)');
console.log('  GET    /admin/users    - List all users (admin only)');
console.log('  DELETE /admin/users/:id - Delete user (admin only)');
```

## Usage Examples

### Register User

```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "password": "securePassword123",
    "name": "Alice Smith"
  }'
```

**Response:**

```json
{
  "user": {
    "id": "user-1705320000000-abc123",
    "email": "alice@example.com",
    "name": "Alice Smith",
    "role": "user"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Login

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "password": "securePassword123"
  }'
```

**Response:**

```json
{
  "user": {
    "id": "user-1705320000000-abc123",
    "email": "alice@example.com",
    "name": "Alice Smith",
    "role": "user"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Get Current User (Protected)

```bash
curl http://localhost:3000/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response:**

```json
{
  "user": {
    "id": "user-1705320000000-abc123",
    "email": "alice@example.com",
    "name": "Alice Smith",
    "role": "user",
    "created_at": "2024-01-15T10:00:00.000Z"
  }
}
```

### Update Profile

```bash
curl -X PUT http://localhost:3000/auth/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Johnson"
  }'
```

### Refresh Token

```bash
curl -X POST http://localhost:3000/auth/refresh \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.NEW_TOKEN..."
}
```

### Admin: List All Users

```bash
curl http://localhost:3000/admin/users \
  -H "Authorization: Bearer ADMIN_TOKEN..."
```

**Response:**

```json
{
  "users": [
    {
      "id": "user-1",
      "email": "alice@example.com",
      "name": "Alice Smith",
      "role": "user",
      "created_at": "2024-01-15T10:00:00.000Z"
    },
    {
      "id": "user-2",
      "email": "admin@example.com",
      "name": "Admin User",
      "role": "admin",
      "created_at": "2024-01-15T09:00:00.000Z"
    }
  ]
}
```

## Frontend Integration (React)

```typescript
// auth.ts - Auth service
class AuthService {
  private baseUrl = 'http://localhost:3000';
  private token: string | null = null;

  constructor() {
    this.token = localStorage.getItem('token');
  }

  async register(email: string, password: string, name: string) {
    const response = await fetch(`${this.baseUrl}/auth/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password, name }),
    });

    const data = await response.json();

    if (response.ok) {
      this.token = data.token;
      localStorage.setItem('token', data.token);
      return data.user;
    }

    throw new Error(data.error);
  }

  async login(email: string, password: string) {
    const response = await fetch(`${this.baseUrl}/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });

    const data = await response.json();

    if (response.ok) {
      this.token = data.token;
      localStorage.setItem('token', data.token);
      return data.user;
    }

    throw new Error(data.error);
  }

  async getCurrentUser() {
    if (!this.token) return null;

    const response = await fetch(`${this.baseUrl}/auth/me`, {
      headers: { Authorization: `Bearer ${this.token}` },
    });

    if (response.ok) {
      const data = await response.json();
      return data.user;
    }

    return null;
  }

  logout() {
    this.token = null;
    localStorage.removeItem('token');
  }

  getToken() {
    return this.token;
  }
}

export const authService = new AuthService();
```

```tsx
// LoginForm.tsx
import { useState } from 'react';
import { authService } from './auth';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');

    try {
      const user = await authService.login(email, password);
      console.log('Logged in:', user);
      // Redirect to dashboard
    } catch (err) {
      setError(err.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        required
      />
      {error && <div className="error">{error}</div>}
      <button type="submit">Login</button>
    </form>
  );
}
```

## Testing

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';

describe('Authentication', () => {
  let token: string;

  beforeEach(() => {
    // Clear database
    db.exec('DELETE FROM users');
  });

  it('should register a new user', async () => {
    const res = await request('http://localhost:3000')
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      })
      .expect(201);

    expect(res.body.user.email).toBe('test@example.com');
    expect(res.body.token).toBeDefined();
  });

  it('should not allow duplicate email', async () => {
    // Register first user
    await request('http://localhost:3000')
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'password123',
        name: 'User 1',
      });

    // Try to register with same email
    const res = await request('http://localhost:3000')
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'password456',
        name: 'User 2',
      })
      .expect(409);

    expect(res.body.error).toBe('Email already registered');
  });

  it('should login with valid credentials', async () => {
    // Register user
    await request('http://localhost:3000')
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      });

    // Login
    const res = await request('http://localhost:3000')
      .post('/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123',
      })
      .expect(200);

    expect(res.body.token).toBeDefined();
    token = res.body.token;
  });

  it('should reject invalid credentials', async () => {
    await request('http://localhost:3000')
      .post('/auth/login')
      .send({
        email: 'invalid@example.com',
        password: 'wrongpassword',
      })
      .expect(401);
  });

  it('should access protected route with valid token', async () => {
    // Register and get token
    const regRes = await request('http://localhost:3000')
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      });

    token = regRes.body.token;

    // Access protected route
    const res = await request('http://localhost:3000')
      .get('/auth/me')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    expect(res.body.user.email).toBe('test@example.com');
  });

  it('should reject access without token', async () => {
    await request('http://localhost:3000')
      .get('/auth/me')
      .expect(401);
  });
});
```

## Security Best Practices

1. **Use HTTPS in production**
2. **Store JWT secret in environment variables**
3. **Use strong password requirements**
4. **Implement rate limiting on auth endpoints**
5. **Add email verification**
6. **Implement password reset flow**
7. **Use refresh tokens for long sessions**
8. **Hash passwords with bcrypt (10+ rounds)**

## See Also

- [Authentication Guide](../authentication/jwt.md)
- [Password Hashing](../authentication/password-hashing.md)
- [Security Best Practices](../guides/security.md)
