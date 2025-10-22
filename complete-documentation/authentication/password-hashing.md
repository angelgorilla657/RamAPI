# Password Hashing

Secure password storage using bcrypt. This guide covers the PasswordService class, hashing best practices, and security considerations.

## Table of Contents

1. [Overview](#overview)
2. [PasswordService Class](#passwordservice-class)
3. [Hashing Passwords](#hashing-passwords)
4. [Verifying Passwords](#verifying-passwords)
5. [Password Requirements](#password-requirements)
6. [Security Best Practices](#security-best-practices)
7. [Migration & Rehashing](#migration--rehashing)
8. [Complete Examples](#complete-examples)

---

## Overview

RamAPI uses bcrypt for password hashing, which is:
- **Slow by design** - Protects against brute-force attacks
- **Salted automatically** - Each hash is unique
- **Adaptive** - Can increase security over time

### Installation

Bcrypt is a required dependency:

```bash
npm install bcryptjs
```

### Quick Start

```typescript
import { passwordService } from 'ramapi';

// Hash a password
const hash = await passwordService.hash('mypassword123');

// Verify a password
const isValid = await passwordService.verify('mypassword123', hash);
console.log(isValid); // true
```

---

## PasswordService Class

### Constructor

```typescript
class PasswordService {
  constructor(saltRounds?: number);
}
```

- **saltRounds**: Number of bcrypt rounds (default: 10)
  - Higher = more secure but slower
  - Recommended: 10-12 for most applications

### Default Instance

RamAPI provides a default instance:

```typescript
import { passwordService } from 'ramapi';

// Uses default 10 salt rounds
await passwordService.hash('password');
```

### Custom Instance

Create a custom instance with different salt rounds:

```typescript
import { PasswordService } from 'ramapi';

// More secure but slower (good for sensitive apps)
const securePasswordService = new PasswordService(12);

// Faster but less secure (not recommended)
const fastPasswordService = new PasswordService(8);
```

### Methods

#### hash(password: string): Promise<string>

Hashes a password with bcrypt.

```typescript
const hash = await passwordService.hash('mypassword123');
// $2a$10$N9qo8uLOickgx2ZMRZoMye...
```

#### verify(password: string, hash: string): Promise<boolean>

Verifies a password against a hash.

```typescript
const isValid = await passwordService.verify('mypassword123', hash);
// true or false
```

#### needsRehash(hash: string): boolean

Checks if a hash needs to be rehashed (e.g., salt rounds changed).

```typescript
const needsUpdate = passwordService.needsRehash(oldHash);
if (needsUpdate) {
  const newHash = await passwordService.hash(password);
  // Update database
}
```

---

## Hashing Passwords

### Basic Hashing

```typescript
import { passwordService } from 'ramapi';

const password = 'userPassword123';
const hash = await passwordService.hash(password);

console.log(hash);
// $2a$10$N9qo8uLOickgx2ZMRZoMye...
```

### Registration Example

```typescript
import { createApp, passwordService, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

const registerSchema = {
  body: z.object({
    email: z.string().email(),
    password: z.string().min(8),
    name: z.string().min(1),
  }),
};

app.post('/auth/register',
  validate(registerSchema),
  async (ctx) => {
    const { email, password, name } = ctx.body;

    // Check if user exists
    const existing = await db.users.findByEmail(email);
    if (existing) {
      throw new HTTPError(409, 'Email already registered');
    }

    // Hash password
    const passwordHash = await passwordService.hash(password);

    // Create user
    const user = await db.users.create({
      email,
      passwordHash,
      name,
    });

    ctx.json({
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    }, 201);
  }
);
```

### Password Reset Example

```typescript
app.post('/auth/reset-password',
  validate({
    body: z.object({
      token: z.string(),
      newPassword: z.string().min(8),
    }),
  }),
  async (ctx) => {
    const { token, newPassword } = ctx.body;

    // Verify reset token
    const userId = await verifyResetToken(token);
    if (!userId) {
      throw new HTTPError(400, 'Invalid or expired reset token');
    }

    // Hash new password
    const passwordHash = await passwordService.hash(newPassword);

    // Update user password
    await db.users.update(userId, { passwordHash });

    ctx.json({ message: 'Password reset successful' });
  }
);
```

---

## Verifying Passwords

### Basic Verification

```typescript
import { passwordService } from 'ramapi';

const password = 'userPassword123';
const hash = '$2a$10$N9qo8uLOickgx2ZMRZoMye...';

const isValid = await passwordService.verify(password, hash);

if (isValid) {
  console.log('Password is correct');
} else {
  console.log('Password is incorrect');
}
```

### Login Example

```typescript
import { JWTService, passwordService } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400,
});

app.post('/auth/login',
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(1),
    }),
  }),
  async (ctx) => {
    const { email, password } = ctx.body;

    // Find user
    const user = await db.users.findByEmail(email);
    if (!user) {
      throw new HTTPError(401, 'Invalid credentials');
    }

    // Verify password
    const isValid = await passwordService.verify(password, user.passwordHash);
    if (!isValid) {
      throw new HTTPError(401, 'Invalid credentials');
    }

    // Generate JWT token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
    });

    ctx.json({ token });
  }
);
```

### Timing-Safe Verification

The example above is already timing-safe because bcrypt comparison is constant-time. However, always return the same error message for both "user not found" and "invalid password" to prevent user enumeration:

```typescript
// GOOD - Same error message
if (!user || !(await passwordService.verify(password, user.passwordHash))) {
  throw new HTTPError(401, 'Invalid credentials');
}

// BAD - Different error messages reveal if user exists
if (!user) {
  throw new HTTPError(401, 'User not found');
}
if (!(await passwordService.verify(password, user.passwordHash))) {
  throw new HTTPError(401, 'Invalid password');
}
```

---

## Password Requirements

### Validation with Zod

```typescript
import { z } from 'zod';

const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .max(128, 'Password must not exceed 128 characters')
  .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
  .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
  .regex(/[0-9]/, 'Password must contain at least one number')
  .regex(/[^a-zA-Z0-9]/, 'Password must contain at least one special character');

const registerSchema = {
  body: z.object({
    email: z.string().email(),
    password: passwordSchema,
  }),
};
```

### Password Strength Checker

```typescript
function checkPasswordStrength(password: string): 'weak' | 'medium' | 'strong' {
  let strength = 0;

  // Length
  if (password.length >= 8) strength++;
  if (password.length >= 12) strength++;

  // Character types
  if (/[a-z]/.test(password)) strength++;
  if (/[A-Z]/.test(password)) strength++;
  if (/[0-9]/.test(password)) strength++;
  if (/[^a-zA-Z0-9]/.test(password)) strength++;

  if (strength <= 2) return 'weak';
  if (strength <= 4) return 'medium';
  return 'strong';
}

app.post('/auth/check-password', async (ctx) => {
  const { password } = ctx.body;
  const strength = checkPasswordStrength(password);
  ctx.json({ strength });
});
```

### Common Password Check

```typescript
const commonPasswords = new Set([
  'password',
  '123456',
  '12345678',
  'qwerty',
  'abc123',
  // ... load from file
]);

function isCommonPassword(password: string): boolean {
  return commonPasswords.has(password.toLowerCase());
}

const passwordSchema = z.string()
  .min(8)
  .refine(
    (password) => !isCommonPassword(password),
    'Password is too common'
  );
```

---

## Security Best Practices

### 1. Use Strong Salt Rounds

```typescript
// Default - Good for most applications
const passwordService = new PasswordService(10);

// More secure - Good for sensitive data
const passwordService = new PasswordService(12);

// Too low - Not recommended
const passwordService = new PasswordService(6); // DON'T USE
```

### 2. Never Store Plain Passwords

```typescript
// BAD - Storing plain password
await db.users.create({
  email: user.email,
  password: user.password, // NEVER DO THIS
});

// GOOD - Storing hashed password
const passwordHash = await passwordService.hash(user.password);
await db.users.create({
  email: user.email,
  passwordHash,
});
```

### 3. Never Log Passwords

```typescript
// BAD
console.log('User password:', password);
console.log('Login attempt:', { email, password });

// GOOD
console.log('Login attempt:', { email });
```

### 4. Use HTTPS

Always transmit passwords over HTTPS to prevent interception.

### 5. Rate Limit Login Attempts

```typescript
import { rateLimit } from 'ramapi';

app.post('/auth/login',
  rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    maxRequests: 5, // 5 attempts
    message: 'Too many login attempts, please try again later',
  }),
  loginHandler
);
```

### 6. Implement Account Lockout

```typescript
async function checkLoginAttempts(userId: string) {
  const attempts = await redis.get(`login-attempts:${userId}`);
  if (attempts && parseInt(attempts) >= 5) {
    throw new HTTPError(429, 'Account temporarily locked. Try again in 15 minutes.');
  }
}

async function recordFailedLogin(userId: string) {
  const key = `login-attempts:${userId}`;
  await redis.incr(key);
  await redis.expire(key, 15 * 60); // 15 minutes
}
```

### 7. Hash on the Server

```typescript
// BAD - Client sends hashed password
// Client: hash(password) -> Server: store directly
// This makes the hash the password!

// GOOD - Client sends plain password over HTTPS
// Client: password (over HTTPS) -> Server: hash(password) -> store
```

### 8. Minimum Password Length

```typescript
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .max(128, 'Password is too long');
```

---

## Migration & Rehashing

### Checking for Rehash

```typescript
app.post('/auth/login', async (ctx) => {
  const { email, password } = ctx.body;

  const user = await db.users.findByEmail(email);
  if (!user || !(await passwordService.verify(password, user.passwordHash))) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Check if password needs rehashing
  if (passwordService.needsRehash(user.passwordHash)) {
    // Rehash with new salt rounds
    const newHash = await passwordService.hash(password);
    await db.users.update(user.id, { passwordHash: newHash });
  }

  // Generate token and respond
  const token = jwtService.sign({ sub: user.id });
  ctx.json({ token });
});
```

### Migrating from Another Hash Algorithm

```typescript
async function migratePassword(userId: string, plainPassword: string) {
  // Hash with bcrypt
  const bcryptHash = await passwordService.hash(plainPassword);

  // Update user
  await db.users.update(userId, {
    passwordHash: bcryptHash,
    hashAlgorithm: 'bcrypt',
  });
}

app.post('/auth/login', async (ctx) => {
  const { email, password } = ctx.body;
  const user = await db.users.findByEmail(email);

  if (!user) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Check if using old hash algorithm
  if (user.hashAlgorithm === 'md5') {
    // Verify with old algorithm
    const isValid = verifyMD5(password, user.passwordHash);
    if (isValid) {
      // Migrate to bcrypt
      await migratePassword(user.id, password);
    } else {
      throw new HTTPError(401, 'Invalid credentials');
    }
  } else {
    // Verify with bcrypt
    const isValid = await passwordService.verify(password, user.passwordHash);
    if (!isValid) {
      throw new HTTPError(401, 'Invalid credentials');
    }
  }

  // Continue with login...
});
```

### Increasing Salt Rounds Over Time

```typescript
// Old service with 10 rounds
const oldService = new PasswordService(10);

// New service with 12 rounds
const newService = new PasswordService(12);

app.post('/auth/login', async (ctx) => {
  const { email, password } = ctx.body;
  const user = await db.users.findByEmail(email);

  if (!user) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Verify with old service
  const isValid = await oldService.verify(password, user.passwordHash);
  if (!isValid) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  // Check if needs rehashing with new rounds
  if (newService.needsRehash(user.passwordHash)) {
    const newHash = await newService.hash(password);
    await db.users.update(user.id, { passwordHash: newHash });
  }

  // Continue...
});
```

---

## Complete Examples

### Complete Registration Flow

```typescript
import { createApp, passwordService, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .regex(/[a-z]/, 'Must contain lowercase letter')
  .regex(/[A-Z]/, 'Must contain uppercase letter')
  .regex(/[0-9]/, 'Must contain number');

const registerSchema = {
  body: z.object({
    email: z.string().email(),
    password: passwordSchema,
    name: z.string().min(1),
  }),
};

app.post('/auth/register',
  validate(registerSchema),
  async (ctx) => {
    const { email, password, name } = ctx.body;

    // Check if email exists
    const existing = await db.users.findByEmail(email);
    if (existing) {
      throw new HTTPError(409, 'Email already registered');
    }

    // Hash password
    const passwordHash = await passwordService.hash(password);

    // Create user
    const user = await db.users.create({
      email,
      passwordHash,
      name,
      createdAt: new Date(),
    });

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
    });

    ctx.json({
      token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    }, 201);
  }
);
```

### Complete Login Flow

```typescript
import { rateLimit, JWTService } from 'ramapi';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400,
});

app.post('/auth/login',
  rateLimit({
    windowMs: 15 * 60 * 1000,
    maxRequests: 5,
  }),
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(1),
    }),
  }),
  async (ctx) => {
    const { email, password } = ctx.body;

    // Find user
    const user = await db.users.findByEmail(email);

    // Verify password (timing-safe)
    if (!user || !(await passwordService.verify(password, user.passwordHash))) {
      throw new HTTPError(401, 'Invalid credentials');
    }

    // Check if password needs rehashing
    if (passwordService.needsRehash(user.passwordHash)) {
      const newHash = await passwordService.hash(password);
      await db.users.update(user.id, { passwordHash: newHash });
    }

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
      role: user.role,
    });

    ctx.json({
      token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    });
  }
);
```

---

## Next Steps

- [JWT Authentication](jwt.md)
- [Authentication Patterns](auth-patterns.md)
- [Security Best Practices](../guides/security.md)
- [Validation Guide](../core-concepts/validation.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
