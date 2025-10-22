# Authentication Patterns & Examples

Complete authentication patterns for common use cases. This guide covers login/register flows, protected routes, role-based access control, and real-world examples.

## Table of Contents

1. [Complete Authentication System](#complete-authentication-system)
2. [Login & Registration](#login--registration)
3. [Protected Routes](#protected-routes)
4. [Role-Based Access Control](#role-based-access-control)
5. [Refresh Tokens](#refresh-tokens)
6. [Password Reset Flow](#password-reset-flow)
7. [Email Verification](#email-verification)
8. [Multi-Factor Authentication](#multi-factor-authentication)
9. [Session Management](#session-management)
10. [Complete Examples](#complete-examples)

---

## Complete Authentication System

### Basic Setup

```typescript
import {
  createApp,
  JWTService,
  passwordService,
  authenticate,
  validate,
  rateLimit,
} from 'ramapi';
import { z } from 'zod';

// JWT Service
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400, // 24 hours
  issuer: 'my-api',
});

const app = createApp();

// Global middleware
app.use(cors());
app.use(logger());
```

---

## Login & Registration

### Registration Endpoint

```typescript
const registerSchema = {
  body: z.object({
    email: z.string().email(),
    password: z.string()
      .min(8, 'Password must be at least 8 characters')
      .regex(/[a-z]/, 'Must contain lowercase letter')
      .regex(/[A-Z]/, 'Must contain uppercase letter')
      .regex(/[0-9]/, 'Must contain number'),
    name: z.string().min(1),
  }),
};

app.post('/auth/register',
  rateLimit({ maxRequests: 5 }),
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
      role: 'user',
      emailVerified: false,
      createdAt: new Date(),
    });

    // Generate token
    const token = jwtService.sign({
      sub: user.id,
      email: user.email,
      role: user.role,
    });

    // Send verification email (async)
    sendVerificationEmail(user.email, user.id).catch(console.error);

    ctx.json({
      token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
      },
    }, 201);
  }
);
```

### Login Endpoint

```typescript
const loginSchema = {
  body: z.object({
    email: z.string().email(),
    password: z.string().min(1),
  }),
};

app.post('/auth/login',
  rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    maxRequests: 5,
    message: 'Too many login attempts',
  }),
  validate(loginSchema),
  async (ctx) => {
    const { email, password } = ctx.body;

    // Find user
    const user = await db.users.findByEmail(email);

    // Verify password (timing-safe)
    if (!user || !(await passwordService.verify(password, user.passwordHash))) {
      throw new HTTPError(401, 'Invalid credentials');
    }

    // Check if account is locked
    if (user.lockedUntil && user.lockedUntil > new Date()) {
      throw new HTTPError(423, 'Account is temporarily locked');
    }

    // Reset failed login attempts
    await db.users.update(user.id, { failedLoginAttempts: 0 });

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
        role: user.role,
      },
    });
  }
);
```

### Logout Endpoint

```typescript
// Simple logout (client-side token removal)
app.post('/auth/logout',
  authenticate(jwtService),
  async (ctx) => {
    // Optional: Add token to blacklist
    await redis.set(`blacklist:${ctx.state.token}`, '1', 'EX', 86400);

    ctx.json({ message: 'Logged out successfully' });
  }
);
```

---

## Protected Routes

### Basic Protection

```typescript
// Public route
app.get('/api/public', async (ctx) => {
  ctx.json({ message: 'Public data' });
});

// Protected route
app.get('/api/profile',
  authenticate(jwtService),
  async (ctx) => {
    const user = await db.users.findById(ctx.user.sub);
    ctx.json({ user });
  }
);
```

### Optional Authentication

```typescript
import { optionalAuthenticate } from 'ramapi';

app.get('/api/posts',
  optionalAuthenticate(jwtService),
  async (ctx) => {
    if (ctx.user) {
      // Authenticated - show all posts including drafts
      const posts = await db.posts.findByUserId(ctx.user.sub);
      ctx.json({ posts, authenticated: true });
    } else {
      // Not authenticated - show only published posts
      const posts = await db.posts.findPublished();
      ctx.json({ posts, authenticated: false });
    }
  }
);
```

### Protected Route Groups

```typescript
app.group('/api/private', (router) => {
  // Apply authentication to all routes in this group
  router.use(authenticate(jwtService));

  router.get('/profile', async (ctx) => {
    const user = await db.users.findById(ctx.user.sub);
    ctx.json({ user });
  });

  router.put('/profile', async (ctx) => {
    await db.users.update(ctx.user.sub, ctx.body);
    ctx.json({ message: 'Profile updated' });
  });

  router.get('/posts', async (ctx) => {
    const posts = await db.posts.findByUserId(ctx.user.sub);
    ctx.json({ posts });
  });
});
```

---

## Role-Based Access Control

### Role Middleware

```typescript
function requireRole(...roles: string[]): Middleware {
  return async (ctx, next) => {
    if (!ctx.user) {
      throw new HTTPError(401, 'Authentication required');
    }

    if (!roles.includes(ctx.user.role)) {
      throw new HTTPError(403, 'Insufficient permissions');
    }

    await next();
  };
}
```

### Admin-Only Routes

```typescript
app.get('/api/admin/users',
  authenticate(jwtService),
  requireRole('admin'),
  async (ctx) => {
    const users = await db.users.findAll();
    ctx.json({ users });
  }
);

app.delete('/api/admin/users/:id',
  authenticate(jwtService),
  requireRole('admin'),
  async (ctx) => {
    await db.users.delete(ctx.params.id);
    ctx.json({ message: 'User deleted' });
  }
);
```

### Multiple Roles

```typescript
// Allow admin or moderator
app.post('/api/posts/:id/approve',
  authenticate(jwtService),
  requireRole('admin', 'moderator'),
  async (ctx) => {
    await db.posts.approve(ctx.params.id);
    ctx.json({ message: 'Post approved' });
  }
);
```

### Permission-Based Access

```typescript
function requirePermission(...permissions: string[]): Middleware {
  return async (ctx, next) => {
    if (!ctx.user) {
      throw new HTTPError(401, 'Authentication required');
    }

    // Get user permissions from database
    const userPermissions = await db.users.getPermissions(ctx.user.sub);

    // Check if user has all required permissions
    const hasPermissions = permissions.every(p => userPermissions.includes(p));

    if (!hasPermissions) {
      throw new HTTPError(403, 'Insufficient permissions');
    }

    await next();
  };
}

app.delete('/api/posts/:id',
  authenticate(jwtService),
  requirePermission('posts:delete'),
  async (ctx) => {
    await db.posts.delete(ctx.params.id);
    ctx.json({ message: 'Post deleted' });
  }
);
```

### Resource Ownership

```typescript
async function requireOwnership(resourceType: string): Promise<Middleware> {
  return async (ctx, next) => {
    if (!ctx.user) {
      throw new HTTPError(401, 'Authentication required');
    }

    const resourceId = ctx.params.id;
    const userId = ctx.user.sub;

    // Check ownership based on resource type
    let isOwner = false;
    if (resourceType === 'post') {
      const post = await db.posts.findById(resourceId);
      isOwner = post?.userId === userId;
    } else if (resourceType === 'comment') {
      const comment = await db.comments.findById(resourceId);
      isOwner = comment?.userId === userId;
    }

    // Admins can access any resource
    if (!isOwner && ctx.user.role !== 'admin') {
      throw new HTTPError(403, 'Access denied');
    }

    await next();
  };
}

app.put('/api/posts/:id',
  authenticate(jwtService),
  requireOwnership('post'),
  async (ctx) => {
    await db.posts.update(ctx.params.id, ctx.body);
    ctx.json({ message: 'Post updated' });
  }
);
```

---

## Refresh Tokens

### Generate Access & Refresh Tokens

```typescript
interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

async function generateTokens(userId: string): Promise<TokenPair> {
  // Short-lived access token
  const accessToken = jwtService.sign({
    sub: userId,
    type: 'access',
  });

  // Long-lived refresh token
  const refreshToken = crypto.randomBytes(32).toString('hex');

  // Store refresh token in database
  await db.refreshTokens.create({
    token: refreshToken,
    userId,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
  });

  return { accessToken, refreshToken };
}
```

### Login with Refresh Token

```typescript
app.post('/auth/login', async (ctx) => {
  const { email, password } = ctx.body;

  const user = await db.users.findByEmail(email);
  if (!user || !(await passwordService.verify(password, user.passwordHash))) {
    throw new HTTPError(401, 'Invalid credentials');
  }

  const { accessToken, refreshToken } = await generateTokens(user.id);

  ctx.json({
    accessToken,
    refreshToken,
    user: {
      id: user.id,
      email: user.email,
      name: user.name,
    },
  });
});
```

### Refresh Token Endpoint

```typescript
app.post('/auth/refresh',
  validate({
    body: z.object({
      refreshToken: z.string(),
    }),
  }),
  async (ctx) => {
    const { refreshToken } = ctx.body;

    // Verify refresh token
    const tokenRecord = await db.refreshTokens.findByToken(refreshToken);

    if (!tokenRecord) {
      throw new HTTPError(401, 'Invalid refresh token');
    }

    if (tokenRecord.expiresAt < new Date()) {
      await db.refreshTokens.delete(tokenRecord.id);
      throw new HTTPError(401, 'Refresh token expired');
    }

    // Generate new access token
    const accessToken = jwtService.sign({
      sub: tokenRecord.userId,
      type: 'access',
    });

    ctx.json({ accessToken });
  }
);
```

### Revoke Refresh Token

```typescript
app.post('/auth/revoke',
  authenticate(jwtService),
  validate({
    body: z.object({
      refreshToken: z.string(),
    }),
  }),
  async (ctx) => {
    const { refreshToken } = ctx.body;

    await db.refreshTokens.deleteByToken(refreshToken);

    ctx.json({ message: 'Token revoked' });
  }
);
```

---

## Password Reset Flow

### Request Password Reset

```typescript
app.post('/auth/forgot-password',
  rateLimit({ maxRequests: 3 }),
  validate({
    body: z.object({
      email: z.string().email(),
    }),
  }),
  async (ctx) => {
    const { email } = ctx.body;

    const user = await db.users.findByEmail(email);

    // Always return success to prevent email enumeration
    if (!user) {
      ctx.json({ message: 'If the email exists, a reset link has been sent' });
      return;
    }

    // Generate reset token
    const resetToken = crypto.randomBytes(32).toString('hex');
    const resetTokenHash = crypto
      .createHash('sha256')
      .update(resetToken)
      .digest('hex');

    // Store reset token
    await db.users.update(user.id, {
      resetTokenHash,
      resetTokenExpiresAt: new Date(Date.now() + 60 * 60 * 1000), // 1 hour
    });

    // Send email with reset link
    await sendPasswordResetEmail(user.email, resetToken);

    ctx.json({ message: 'If the email exists, a reset link has been sent' });
  }
);
```

### Reset Password

```typescript
app.post('/auth/reset-password',
  rateLimit({ maxRequests: 5 }),
  validate({
    body: z.object({
      token: z.string(),
      newPassword: z.string().min(8),
    }),
  }),
  async (ctx) => {
    const { token, newPassword } = ctx.body;

    // Hash the token to compare with stored hash
    const tokenHash = crypto
      .createHash('sha256')
      .update(token)
      .digest('hex');

    // Find user with this token
    const user = await db.users.findByResetToken(tokenHash);

    if (!user || !user.resetTokenExpiresAt || user.resetTokenExpiresAt < new Date()) {
      throw new HTTPError(400, 'Invalid or expired reset token');
    }

    // Hash new password
    const passwordHash = await passwordService.hash(newPassword);

    // Update password and clear reset token
    await db.users.update(user.id, {
      passwordHash,
      resetTokenHash: null,
      resetTokenExpiresAt: null,
    });

    ctx.json({ message: 'Password reset successful' });
  }
);
```

---

## Email Verification

### Send Verification Email

```typescript
async function sendVerificationEmail(email: string, userId: string) {
  const token = crypto.randomBytes(32).toString('hex');
  const tokenHash = crypto.createHash('sha256').update(token).digest('hex');

  await db.users.update(userId, {
    verificationTokenHash: tokenHash,
    verificationTokenExpiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
  });

  const verificationUrl = `${process.env.APP_URL}/verify-email?token=${token}`;

  // Send email with verification link
  await emailService.send({
    to: email,
    subject: 'Verify your email',
    html: `Click here to verify: <a href="${verificationUrl}">${verificationUrl}</a>`,
  });
}
```

### Verify Email Endpoint

```typescript
app.post('/auth/verify-email',
  validate({
    body: z.object({
      token: z.string(),
    }),
  }),
  async (ctx) => {
    const { token } = ctx.body;

    const tokenHash = crypto
      .createHash('sha256')
      .update(token)
      .digest('hex');

    const user = await db.users.findByVerificationToken(tokenHash);

    if (!user || !user.verificationTokenExpiresAt || user.verificationTokenExpiresAt < new Date()) {
      throw new HTTPError(400, 'Invalid or expired verification token');
    }

    await db.users.update(user.id, {
      emailVerified: true,
      verificationTokenHash: null,
      verificationTokenExpiresAt: null,
    });

    ctx.json({ message: 'Email verified successfully' });
  }
);
```

### Resend Verification Email

```typescript
app.post('/auth/resend-verification',
  authenticate(jwtService),
  rateLimit({ maxRequests: 3 }),
  async (ctx) => {
    const user = await db.users.findById(ctx.user.sub);

    if (user.emailVerified) {
      throw new HTTPError(400, 'Email already verified');
    }

    await sendVerificationEmail(user.email, user.id);

    ctx.json({ message: 'Verification email sent' });
  }
);
```

---

## Multi-Factor Authentication

### Enable 2FA

```typescript
import speakeasy from 'speakeasy';
import qrcode from 'qrcode';

app.post('/auth/2fa/enable',
  authenticate(jwtService),
  async (ctx) => {
    const user = await db.users.findById(ctx.user.sub);

    if (user.twoFactorEnabled) {
      throw new HTTPError(400, '2FA already enabled');
    }

    // Generate secret
    const secret = speakeasy.generateSecret({
      name: `MyApp (${user.email})`,
    });

    // Store secret temporarily
    await db.users.update(user.id, {
      twoFactorSecretTemp: secret.base32,
    });

    // Generate QR code
    const qrCodeUrl = await qrcode.toDataURL(secret.otpauth_url!);

    ctx.json({
      secret: secret.base32,
      qrCode: qrCodeUrl,
    });
  }
);
```

### Verify and Confirm 2FA

```typescript
app.post('/auth/2fa/verify',
  authenticate(jwtService),
  validate({
    body: z.object({
      token: z.string().length(6),
    }),
  }),
  async (ctx) => {
    const { token } = ctx.body;
    const user = await db.users.findById(ctx.user.sub);

    if (!user.twoFactorSecretTemp) {
      throw new HTTPError(400, '2FA setup not initiated');
    }

    // Verify token
    const verified = speakeasy.totp.verify({
      secret: user.twoFactorSecretTemp,
      encoding: 'base32',
      token,
    });

    if (!verified) {
      throw new HTTPError(400, 'Invalid 2FA token');
    }

    // Enable 2FA
    await db.users.update(user.id, {
      twoFactorEnabled: true,
      twoFactorSecret: user.twoFactorSecretTemp,
      twoFactorSecretTemp: null,
    });

    ctx.json({ message: '2FA enabled successfully' });
  }
);
```

### Login with 2FA

```typescript
app.post('/auth/login-2fa',
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string(),
      token: z.string().length(6),
    }),
  }),
  async (ctx) => {
    const { email, password, token } = ctx.body;

    const user = await db.users.findByEmail(email);
    if (!user || !(await passwordService.verify(password, user.passwordHash))) {
      throw new HTTPError(401, 'Invalid credentials');
    }

    if (user.twoFactorEnabled) {
      const verified = speakeasy.totp.verify({
        secret: user.twoFactorSecret,
        encoding: 'base32',
        token,
      });

      if (!verified) {
        throw new HTTPError(401, 'Invalid 2FA token');
      }
    }

    const authToken = jwtService.sign({
      sub: user.id,
      email: user.email,
    });

    ctx.json({ token: authToken });
  }
);
```

---

## Session Management

### Get Current User

```typescript
app.get('/auth/me',
  authenticate(jwtService),
  async (ctx) => {
    const user = await db.users.findById(ctx.user.sub);

    if (!user) {
      throw new HTTPError(404, 'User not found');
    }

    ctx.json({
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
        emailVerified: user.emailVerified,
      },
    });
  }
);
```

### Update Profile

```typescript
app.put('/auth/profile',
  authenticate(jwtService),
  validate({
    body: z.object({
      name: z.string().min(1).optional(),
      bio: z.string().max(500).optional(),
    }),
  }),
  async (ctx) => {
    await db.users.update(ctx.user.sub, ctx.body);

    const user = await db.users.findById(ctx.user.sub);
    ctx.json({ user });
  }
);
```

### Change Password

```typescript
app.post('/auth/change-password',
  authenticate(jwtService),
  validate({
    body: z.object({
      currentPassword: z.string(),
      newPassword: z.string().min(8),
    }),
  }),
  async (ctx) => {
    const { currentPassword, newPassword } = ctx.body;

    const user = await db.users.findById(ctx.user.sub);

    // Verify current password
    if (!(await passwordService.verify(currentPassword, user.passwordHash))) {
      throw new HTTPError(401, 'Current password is incorrect');
    }

    // Hash new password
    const passwordHash = await passwordService.hash(newPassword);

    // Update password
    await db.users.update(user.id, { passwordHash });

    ctx.json({ message: 'Password changed successfully' });
  }
);
```

---

## Complete Examples

### Complete Auth API

```typescript
import {
  createApp,
  JWTService,
  passwordService,
  authenticate,
  optionalAuthenticate,
  validate,
  rateLimit,
  cors,
  logger,
} from 'ramapi';
import { z } from 'zod';

const jwtService = new JWTService({
  secret: process.env.JWT_SECRET!,
  expiresIn: 86400,
});

const app = createApp();

app.use(logger());
app.use(cors());

// Registration
app.post('/auth/register',
  rateLimit({ maxRequests: 5 }),
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string().min(8),
      name: z.string().min(1),
    }),
  }),
  async (ctx) => {
    // Implementation from above
  }
);

// Login
app.post('/auth/login',
  rateLimit({ maxRequests: 5 }),
  validate({
    body: z.object({
      email: z.string().email(),
      password: z.string(),
    }),
  }),
  async (ctx) => {
    // Implementation from above
  }
);

// Get current user
app.get('/auth/me',
  authenticate(jwtService),
  async (ctx) => {
    // Implementation from above
  }
);

// Protected resource
app.get('/api/posts',
  optionalAuthenticate(jwtService),
  async (ctx) => {
    // Implementation from above
  }
);

// Admin route
app.get('/api/admin/users',
  authenticate(jwtService),
  requireRole('admin'),
  async (ctx) => {
    // Implementation from above
  }
);

app.listen(3000);
```

---

## Next Steps

- [JWT Authentication](jwt.md)
- [Password Hashing](password-hashing.md)
- [Security Best Practices](../guides/security.md)
- [Middleware Guide](../middleware/custom-middleware.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
