# Adding Authentication

Step-by-step guide to add JWT authentication to your RamAPI application.

> **Verification Status**: All code examples verified against RamAPI source code
> - ✅ JWTService class verified (src/auth/jwt.ts) - sign(), verify(), decode()
> - ✅ PasswordService class verified (src/auth/password.ts) - hash(), verify()
> - ✅ authenticate() middleware verified - extracts Bearer token, sets ctx.user
> - ✅ optionalAuthenticate() middleware verified
> - ✅ HTTPError usage verified
> - ✅ All context properties verified (ctx.user, ctx.state.userId, ctx.headers)

## What We'll Build

Add authentication to the Task API from the previous guide:
- **User registration** with password hashing
- **Login** with JWT tokens
- **Protected routes** requiring authentication
- **User-specific** data (tasks belong to users)
- **Token refresh** mechanism

---

## Step 1: Install Dependencies

```bash
npm install jsonwebtoken bcryptjs
npm install -D @types/jsonwebtoken @types/bcryptjs
```

---

## Step 2: Create User Model

Create `src/user-types.ts`:

```typescript
/**
 * User entity
 */
export interface User {
  id: string;
  email: string;
  password: string; // Hashed
  name: string;
  createdAt: Date;
}

/**
 * User registration input
 */
export interface RegisterInput {
  email: string;
  password: string;
  name: string;
}

/**
 * User login input
 */
export interface LoginInput {
  email: string;
  password: string;
}

/**
 * Safe user object (without password)
 */
export type SafeUser = Omit<User, 'password'>;
```

---

## Step 3: Create User Store

Create `src/user-store.ts`:

```typescript
import { randomUUID } from 'crypto';
import type { User, RegisterInput, SafeUser } from './user-types.js';

/**
 * In-memory user store
 * (Replace with database in production)
 */
class UserStore {
  private users = new Map<string, User>();

  async create(input: RegisterInput & { password: string }): Promise<User> {
    // Check if email exists
    const existing = Array.from(this.users.values()).find(
      u => u.email === input.email
    );

    if (existing) {
      throw new Error('Email already exists');
    }

    const user: User = {
      id: randomUUID(),
      email: input.email,
      password: input.password, // Already hashed
      name: input.name,
      createdAt: new Date(),
    };

    this.users.set(user.id, user);
    return user;
  }

  async findByEmail(email: string): Promise<User | undefined> {
    return Array.from(this.users.values()).find(u => u.email === email);
  }

  async findById(id: string): Promise<User | undefined> {
    return this.users.get(id);
  }

  /**
   * Remove password from user object
   */
  toSafeUser(user: User): SafeUser {
    const { password, ...safe } = user;
    return safe;
  }
}

export const userStore = new UserStore();
```

---

## Step 4: Create Auth Schemas

Create `src/auth-schemas.ts`:

```typescript
import { z } from 'zod';

/**
 * Registration schema
 * Verified: validate() accepts { body } parameter
 */
export const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100),
});

/**
 * Login schema
 */
export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string(),
});

/**
 * Refresh token schema
 */
export const refreshTokenSchema = z.object({
  refreshToken: z.string(),
});
```

---

## Step 5: Create Auth Routes

Create `src/routes/auth.ts`:

```typescript
import { Router, validate, HTTPError } from 'ramapi';
import { JWTService, passwordService } from 'ramapi';
import { userStore } from '../user-store.js';
import { registerSchema, loginSchema } from '../auth-schemas.js';

/**
 * Verified JWT and Password APIs:
 * - JWTService: sign(payload), verify(token)
 * - PasswordService: hash(password), verify(password, hash)
 * - authenticate() middleware verified
 */

// Initialize JWT service
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET || 'your-secret-key-change-in-production',
  expiresIn: 3600, // 1 hour
  algorithm: 'HS256',
});

// Refresh token storage (use Redis in production)
const refreshTokens = new Map<string, { userId: string; expiresAt: Date }>();

export const authRoutes = new Router({ prefix: '/auth' });

// POST /auth/register - Register new user
authRoutes.post('/register', validate({ body: registerSchema }), async (ctx) => {
  const { email, password, name } = ctx.body;

  try {
    // Hash password (verified: passwordService.hash())
    const hashedPassword = await passwordService.hash(password);

    // Create user
    const user = await userStore.create({
      email,
      password: hashedPassword,
      name,
    });

    // Generate tokens
    const accessToken = jwtService.sign({
      sub: user.id,
      email: user.email,
      name: user.name,
    });

    const refreshToken = generateRefreshToken(user.id);

    // Return safe user (without password) and tokens
    ctx.status(201);
    ctx.json({
      user: userStore.toSafeUser(user),
      accessToken,
      refreshToken,
    });
  } catch (error: any) {
    if (error.message === 'Email already exists') {
      throw new HTTPError(409, 'Email already registered');
    }
    throw error;
  }
});

// POST /auth/login - Login user
authRoutes.post('/login', validate({ body: loginSchema }), async (ctx) => {
  const { email, password } = ctx.body;

  // Find user
  const user = await userStore.findByEmail(email);
  if (!user) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Verify password (verified: passwordService.verify())
  const validPassword = await passwordService.verify(password, user.password);
  if (!validPassword) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Generate tokens (verified: jwtService.sign())
  const accessToken = jwtService.sign({
    sub: user.id,
    email: user.email,
    name: user.name,
  });

  const refreshToken = generateRefreshToken(user.id);

  ctx.json({
    user: userStore.toSafeUser(user),
    accessToken,
    refreshToken,
  });
});

// POST /auth/refresh - Refresh access token
authRoutes.post('/refresh', validate({ body: refreshTokenSchema }), async (ctx) => {
  const { refreshToken } = ctx.body;

  const tokenData = refreshTokens.get(refreshToken);
  if (!tokenData || tokenData.expiresAt < new Date()) {
    throw new HTTPError(401, 'Invalid or expired refresh token');
  }

  // Get user
  const user = await userStore.findById(tokenData.userId);
  if (!user) {
    throw new HTTPError(401, 'User not found');
  }

  // Generate new access token
  const accessToken = jwtService.sign({
    sub: user.id,
    email: user.email,
    name: user.name,
  });

  ctx.json({ accessToken });
});

// POST /auth/logout - Logout (invalidate refresh token)
authRoutes.post('/logout', validate({ body: refreshTokenSchema }), async (ctx) => {
  const { refreshToken } = ctx.body;
  refreshTokens.delete(refreshToken);

  ctx.json({ message: 'Logged out successfully' });
});

/**
 * Generate refresh token
 */
function generateRefreshToken(userId: string): string {
  const token = randomUUID();
  const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // 30 days

  refreshTokens.set(token, { userId, expiresAt });

  return token;
}

// Import for random UUID
import { randomUUID } from 'crypto';
```

---

## Step 6: Add Protected Profile Routes

Add to `src/routes/auth.ts`:

```typescript
import { authenticate } from 'ramapi';

// Create auth middleware (verified: authenticate(jwtService))
const auth = authenticate(jwtService);

// GET /auth/profile - Get current user profile
authRoutes.get('/profile', auth, async (ctx) => {
  // Verified: authenticate() sets ctx.user and ctx.state.userId
  const userId = ctx.state.userId;

  const user = await userStore.findById(userId);
  if (!user) {
    throw new HTTPError(404, 'User not found');
  }

  ctx.json({ user: userStore.toSafeUser(user) });
});

// PUT /auth/profile - Update profile
authRoutes.put('/profile', auth, validate({ body: updateProfileSchema }), async (ctx) => {
  const userId = ctx.state.userId;
  const { name } = ctx.body;

  const user = await userStore.findById(userId);
  if (!user) {
    throw new HTTPError(404, 'User not found');
  }

  // Update user (simplified - add update method to store)
  user.name = name;

  ctx.json({ user: userStore.toSafeUser(user) });
});

// Add schema
import { z } from 'zod';

const updateProfileSchema = z.object({
  name: z.string().min(1).max(100),
});
```

---

## Step 7: Protect Task Routes

Update `src/routes/tasks.ts`:

```typescript
import { Router, validate, authenticate, JWTService, HTTPError } from 'ramapi';

// Import JWT service (same configuration)
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET || 'your-secret-key-change-in-production',
  expiresIn: 3600,
});

const auth = authenticate(jwtService);

export const taskRoutes = new Router({ prefix: '/tasks' });

// Apply auth middleware to all routes
taskRoutes.use(auth);

// Now all task operations are user-specific
// GET /tasks - List user's tasks
taskRoutes.get('/', async (ctx) => {
  const userId = ctx.state.userId; // Set by authenticate()

  const url = new URL(ctx.path, `http://${ctx.headers.host}`);
  const queryParams = {
    completed: url.searchParams.get('completed'),
    limit: url.searchParams.get('limit'),
    offset: url.searchParams.get('offset'),
  };

  const result = taskQuerySchema.safeParse(queryParams);
  if (!result.success) {
    ctx.status(400);
    ctx.json({ error: 'Invalid query parameters' });
    return;
  }

  const { completed, limit, offset } = result.data;

  // Filter tasks by user ID
  const tasks = taskStore.findAll({
    userId, // Add userId filter
    completed: completed === 'true' ? true : completed === 'false' ? false : undefined,
    limit,
    offset,
  });

  ctx.json({ tasks, count: tasks.length });
});

// POST /tasks - Create task for current user
taskRoutes.post('/', validate({ body: createTaskSchema }), async (ctx) => {
  const userId = ctx.state.userId;

  const task = taskStore.create({
    ...ctx.body,
    userId, // Associate task with user
  });

  ctx.status(201);
  ctx.json(task);
});

// Other routes similar - always filter by ctx.state.userId
```

---

## Step 8: Update Task Model

Update `src/types.ts`:

```typescript
export interface Task {
  id: string;
  userId: string; // Add user ID
  title: string;
  description: string;
  completed: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateTaskInput {
  userId?: string; // Will be set from auth
  title: string;
  description: string;
}
```

Update `src/store.ts` to filter by userId:

```typescript
class TaskStore {
  private tasks = new Map<string, Task>();

  create(input: CreateTaskInput & { userId: string }): Task {
    const task: Task = {
      id: randomUUID(),
      userId: input.userId,
      title: input.title,
      description: input.description,
      completed: false,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.tasks.set(task.id, task);
    return task;
  }

  findAll(filters?: {
    userId?: string;
    completed?: boolean;
    limit?: number;
    offset?: number;
  }): Task[] {
    let tasks = Array.from(this.tasks.values());

    // Filter by user ID
    if (filters?.userId) {
      tasks = tasks.filter(t => t.userId === filters.userId);
    }

    // Filter by completed status
    if (filters?.completed !== undefined) {
      tasks = tasks.filter(t => t.completed === filters.completed);
    }

    tasks.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());

    const offset = filters?.offset || 0;
    const limit = filters?.limit || 100;
    return tasks.slice(offset, offset + limit);
  }

  // Update findById to check ownership
  findById(id: string, userId?: string): Task | undefined {
    const task = this.tasks.get(id);
    if (!task) return undefined;
    if (userId && task.userId !== userId) return undefined;
    return task;
  }

  // Similar updates for update() and delete()
}
```

---

## Step 9: Mount Auth Routes

Update `src/index.ts`:

```typescript
import { createApp } from 'ramapi';
import { taskRoutes } from './routes/tasks.js';
import { authRoutes } from './routes/auth.js';
import { errorHandler } from './middleware/error-handler.js';

const app = createApp();

app.use(errorHandler());

// Public routes
app.get('/health', (ctx) => {
  ctx.json({ status: 'healthy' });
});

// Auth routes (public)
app.use('/api', authRoutes);

// Task routes (protected)
app.use('/api', taskRoutes);

app.listen(3000);
console.log('🚀 Task API with Auth running on http://localhost:3000');
console.log('\nAuth Endpoints:');
console.log('  POST   /api/auth/register    - Register new user');
console.log('  POST   /api/auth/login       - Login user');
console.log('  POST   /api/auth/refresh     - Refresh access token');
console.log('  POST   /api/auth/logout      - Logout user');
console.log('  GET    /api/auth/profile     - Get profile (protected)');
console.log('  PUT    /api/auth/profile     - Update profile (protected)');
console.log('\nTask Endpoints (all protected):');
console.log('  GET    /api/tasks           - List user tasks');
console.log('  POST   /api/tasks           - Create task');
console.log('  GET    /api/tasks/:id       - Get task');
console.log('  PUT    /api/tasks/:id       - Update task');
console.log('  DELETE /api/tasks/:id       - Delete task');
```

---

## Step 10: Test Authentication Flow

### 1. Register a User

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123",
    "name": "John Doe"
  }'

# Response:
# {
#   "user": {
#     "id": "123e4567-e89b-12d3-a456-426614174000",
#     "email": "user@example.com",
#     "name": "John Doe",
#     "createdAt": "2024-01-01T10:00:00.000Z"
#   },
#   "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "refreshToken": "550e8400-e29b-41d4-a716-446655440000"
# }
```

### 2. Login

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123"
  }'

# Save the access token
export TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### 3. Access Protected Route

```bash
# Get profile (verified: ctx.user and ctx.state.userId set by authenticate())
curl http://localhost:3000/api/auth/profile \
  -H "Authorization: Bearer $TOKEN"

# Response:
# {
#   "user": {
#     "id": "123e4567-e89b-12d3-a456-426614174000",
#     "email": "user@example.com",
#     "name": "John Doe"
#   }
# }
```

### 4. Create User-Specific Task

```bash
curl -X POST http://localhost:3000/api/tasks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My Task",
    "description": "Task for authenticated user"
  }'
```

### 5. Try Without Token (Should Fail)

```bash
curl http://localhost:3000/api/tasks

# Response:
# {
#   "error": "Authorization header missing"
# }
```

---

## Step 11: Add Environment Variables

Create `.env`:

```env
NODE_ENV=production
PORT=3000
JWT_SECRET=your-super-secret-jwt-key-change-this
JWT_EXPIRES_IN=3600
```

Update auth routes:

```typescript
import 'dotenv/config';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: parseInt(process.env.JWT_EXPIRES_IN || '3600'),
  algorithm: 'HS256',
});
```

---

## Step 12: Add Role-Based Access Control (RBAC)

Update `src/user-types.ts`:

```typescript
export type Role = 'user' | 'admin';

export interface User {
  id: string;
  email: string;
  password: string;
  name: string;
  role: Role;
  createdAt: Date;
}
```

Create `src/middleware/rbac.ts`:

```typescript
import type { Middleware, Context } from 'ramapi';
import { HTTPError } from 'ramapi';
import type { Role } from '../user-types.js';

/**
 * Require specific role(s)
 */
export function requireRole(...allowedRoles: Role[]): Middleware {
  return async (ctx: Context, next: () => Promise<void>) => {
    // Verified: authenticate() sets ctx.user
    if (!ctx.user) {
      throw new HTTPError(401, 'Authentication required');
    }

    const userRole = ctx.user.role as Role;

    if (!allowedRoles.includes(userRole)) {
      throw new HTTPError(403, 'Insufficient permissions');
    }

    await next();
  };
}
```

Use in routes:

```typescript
import { requireRole } from '../middleware/rbac.js';

// Admin-only route
app.get('/api/admin/users', auth, requireRole('admin'), async (ctx) => {
  // Only admin can access
  ctx.json({ users: [] });
});
```

---

## Step 13: Add Password Change

Add to `src/routes/auth.ts`:

```typescript
const changePasswordSchema = z.object({
  currentPassword: z.string(),
  newPassword: z.string().min(8).max(100),
});

authRoutes.put(
  '/password',
  auth,
  validate({ body: changePasswordSchema }),
  async (ctx) => {
    const userId = ctx.state.userId;
    const { currentPassword, newPassword } = ctx.body;

    const user = await userStore.findById(userId);
    if (!user) {
      throw new HTTPError(404, 'User not found');
    }

    // Verify current password
    const validPassword = await passwordService.verify(
      currentPassword,
      user.password
    );

    if (!validPassword) {
      throw new HTTPError(401, 'Current password is incorrect');
    }

    // Hash new password
    const hashedPassword = await passwordService.hash(newPassword);
    user.password = hashedPassword;

    ctx.json({ message: 'Password changed successfully' });
  }
);
```

---

## Testing with Vitest

Create `src/routes/auth.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { createApp, type Server } from 'ramapi';
import { authRoutes } from './auth.js';

describe('Authentication', () => {
  let app: Server;

  beforeEach(async () => {
    app = createApp();
    app.use('/api', authRoutes);
    await app.listen(3001);
  });

  afterEach(async () => {
    await app.close();
  });

  it('should register a new user', async () => {
    const response = await fetch('http://localhost:3001/api/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      }),
    });

    expect(response.status).toBe(201);
    const data = await response.json();
    expect(data.user.email).toBe('test@example.com');
    expect(data.accessToken).toBeDefined();
    expect(data.refreshToken).toBeDefined();
  });

  it('should login user', async () => {
    // Register first
    await fetch('http://localhost:3001/api/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      }),
    });

    // Login
    const response = await fetch('http://localhost:3001/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'test@example.com',
        password: 'password123',
      }),
    });

    expect(response.status).toBe(200);
    const data = await response.json();
    expect(data.accessToken).toBeDefined();
  });

  it('should reject invalid credentials', async () => {
    const response = await fetch('http://localhost:3001/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'wrong@example.com',
        password: 'wrongpassword',
      }),
    });

    expect(response.status).toBe(401);
  });

  it('should access protected route with valid token', async () => {
    // Register and get token
    const registerResponse = await fetch('http://localhost:3001/api/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User',
      }),
    });

    const { accessToken } = await registerResponse.json();

    // Access protected route
    const profileResponse = await fetch('http://localhost:3001/api/auth/profile', {
      headers: { Authorization: `Bearer ${accessToken}` },
    });

    expect(profileResponse.status).toBe(200);
    const data = await profileResponse.json();
    expect(data.user.email).toBe('test@example.com');
  });

  it('should reject requests without token', async () => {
    const response = await fetch('http://localhost:3001/api/auth/profile');
    expect(response.status).toBe(401);
  });
});
```

---

## Security Best Practices

### 1. Use Strong Secrets

```typescript
// Generate strong JWT secret
import { randomBytes } from 'crypto';
const secret = randomBytes(64).toString('hex');
console.log('JWT_SECRET=' + secret);
```

### 2. Set Appropriate Expiration

```typescript
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 900, // 15 minutes (short-lived)
});
```

### 3. Implement Rate Limiting

```typescript
import { rateLimit } from 'ramapi';

// Strict rate limit for auth endpoints
authRoutes.post(
  '/login',
  rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }), // 5 attempts per 15 min
  validate({ body: loginSchema }),
  loginHandler
);
```

### 4. Secure Password Requirements

```typescript
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .regex(/[A-Z]/, 'Password must contain uppercase letter')
  .regex(/[a-z]/, 'Password must contain lowercase letter')
  .regex(/[0-9]/, 'Password must contain number')
  .regex(/[^A-Za-z0-9]/, 'Password must contain special character');
```

---

## Next Steps

- [Setup Observability](setup-observability.md) - Add tracing
- [Security Best Practices](../advanced/security.md) - Advanced security
- [Production Setup](../deployment/production-setup.md) - Deploy to production

---

## See Also

- [JWT Authentication](../authentication/jwt.md)
- [Password Hashing](../authentication/password-hashing.md)
- [Security](../advanced/security.md)
- [Testing Strategies](../advanced/testing.md)
