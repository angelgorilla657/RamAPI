# Error Handling

Robust error handling is essential for production applications. RamAPI provides a comprehensive error handling system with the HTTPError class, custom error handlers, and automatic error formatting.

## Table of Contents

1. [HTTPError Class](#httperror-class)
2. [Throwing Errors](#throwing-errors)
3. [Custom Error Handlers](#custom-error-handlers)
4. [Error Middleware](#error-middleware)
5. [Validation Errors](#validation-errors)
6. [Error Response Format](#error-response-format)
7. [Error Handling Patterns](#error-handling-patterns)
8. [Best Practices](#best-practices)

---

## HTTPError Class

The `HTTPError` class extends the standard Error class with HTTP status codes and additional details.

### HTTPError Structure

```typescript
class HTTPError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public details?: unknown
  )
}
```

### Creating HTTPErrors

```typescript
import { HTTPError } from 'ramapi';

// Basic error
throw new HTTPError(404, 'Resource not found');

// With details
throw new HTTPError(400, 'Invalid input', {
  field: 'email',
  reason: 'Email already exists'
});

// Common status codes
throw new HTTPError(400, 'Bad Request');
throw new HTTPError(401, 'Unauthorized');
throw new HTTPError(403, 'Forbidden');
throw new HTTPError(404, 'Not Found');
throw new HTTPError(409, 'Conflict');
throw new HTTPError(422, 'Unprocessable Entity');
throw new HTTPError(500, 'Internal Server Error');
```

---

## Throwing Errors

Throw errors in handlers and middleware to trigger error handling.

### In Route Handlers

```typescript
app.get('/users/:id', async (ctx) => {
  const user = await database.findUser(ctx.params.id);

  if (!user) {
    throw new HTTPError(404, 'User not found');
  }

  ctx.json({ user });
});
```

### With Details

```typescript
app.post('/users', async (ctx) => {
  const { email } = ctx.body;

  const existing = await database.findByEmail(email);

  if (existing) {
    throw new HTTPError(409, 'Email already registered', {
      field: 'email',
      value: email
    });
  }

  const user = await database.createUser(ctx.body);
  ctx.json({ user }, 201);
});
```

### In Middleware

```typescript
const requireAdmin: Middleware = async (ctx, next) => {
  const user = ctx.state.user;

  if (!user || user.role !== 'admin') {
    throw new HTTPError(403, 'Admin access required');
  }

  await next();
};
```

### Async Errors

```typescript
app.get('/data', async (ctx) => {
  try {
    const data = await externalAPI.fetch();
    ctx.json({ data });
  } catch (error) {
    // Wrap external errors
    throw new HTTPError(502, 'External service unavailable', {
      originalError: error.message
    });
  }
});
```

---

## Custom Error Handlers

Define custom error handling logic for your application.

### Basic Error Handler

```typescript
import { createApp, HTTPError } from 'ramapi';

const app = createApp({
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
```

### Development vs Production

```typescript
const isDevelopment = process.env.NODE_ENV === 'development';

const app = createApp({
  onError: async (error, ctx) => {
    console.error('Error:', error);

    const statusCode = error instanceof HTTPError
      ? error.statusCode
      : 500;

    const response: any = {
      error: true,
      message: error.message,
    };

    // Include details in HTTPError
    if (error instanceof HTTPError && error.details) {
      response.details = error.details;
    }

    // Include stack trace in development
    if (isDevelopment) {
      response.stack = error.stack;
    }

    ctx.json(response, statusCode);
  }
});
```

### Logging Errors

```typescript
const app = createApp({
  onError: async (error, ctx) => {
    // Log to your logging service
    logger.error('Request error', {
      error: error.message,
      stack: error.stack,
      method: ctx.method,
      path: ctx.path,
      statusCode: error instanceof HTTPError ? error.statusCode : 500,
      userId: ctx.state.userId,
    });

    // Send response
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
```

### Error Tracking

```typescript
import * as Sentry from '@sentry/node';

const app = createApp({
  onError: async (error, ctx) => {
    // Report to Sentry
    Sentry.captureException(error, {
      extra: {
        method: ctx.method,
        path: ctx.path,
        params: ctx.params,
        query: ctx.query,
        userId: ctx.state.userId,
      }
    });

    // Send response
    const statusCode = error instanceof HTTPError ? error.statusCode : 500;

    ctx.json({
      error: true,
      message: error.message,
      ...(error instanceof HTTPError && { details: error.details })
    }, statusCode);
  }
});
```

---

## Error Middleware

Create middleware specifically for error handling.

### Global Error Catcher

```typescript
const errorCatcher: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    console.error('Caught error:', error);

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
};

// Apply early in middleware chain
app.use(errorCatcher);
```

### Error Recovery

```typescript
const errorRecovery: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    // Try to recover
    if (error.code === 'ECONNREFUSED') {
      // Retry logic
      try {
        await next();
        return;
      } catch (retryError) {
        throw new HTTPError(503, 'Service temporarily unavailable');
      }
    }

    // Re-throw if can't recover
    throw error;
  }
};
```

### Type-Specific Error Handling

```typescript
const databaseErrorHandler: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    // Handle database errors
    if (error.code === '23505') { // Unique constraint violation
      throw new HTTPError(409, 'Resource already exists');
    }

    if (error.code === '23503') { // Foreign key violation
      throw new HTTPError(400, 'Referenced resource does not exist');
    }

    // Re-throw other errors
    throw error;
  }
};
```

---

## Validation Errors

Validation errors are handled automatically by the `validate()` middleware.

### Validation Error Format

```typescript
// Request
POST /users
{
  "name": "J",
  "email": "invalid"
}

// Response (400 Bad Request)
{
  "error": true,
  "message": "Validation failed",
  "details": {
    "errors": [
      {
        "field": "body.name",
        "message": "String must contain at least 2 character(s)",
        "code": "too_small"
      },
      {
        "field": "body.email",
        "message": "Invalid email",
        "code": "invalid_string"
      }
    ]
  }
}
```

### Custom Validation Error Handling

```typescript
const app = createApp({
  onError: async (error, ctx) => {
    if (error instanceof HTTPError && error.message === 'Validation failed') {
      // Custom validation error format
      const errors = error.details as { errors: any[] };

      ctx.json({
        success: false,
        message: 'Please fix the following errors',
        fields: errors.errors.reduce((acc, err) => {
          const field = err.field.replace('body.', '');
          acc[field] = err.message;
          return acc;
        }, {} as Record<string, string>)
      }, 400);
    } else if (error instanceof HTTPError) {
      ctx.json({
        success: false,
        message: error.message,
        details: error.details
      }, error.statusCode);
    } else {
      ctx.json({
        success: false,
        message: 'Internal server error'
      }, 500);
    }
  }
});
```

---

## Error Response Format

Standardize error responses across your API.

### Standard Format

```typescript
interface ErrorResponse {
  error: boolean;
  message: string;
  details?: unknown;
  stack?: string; // Development only
}
```

### Consistent Error Responses

```typescript
const app = createApp({
  onError: async (error, ctx) => {
    const statusCode = error instanceof HTTPError
      ? error.statusCode
      : 500;

    const response: ErrorResponse = {
      error: true,
      message: error.message,
    };

    if (error instanceof HTTPError && error.details) {
      response.details = error.details;
    }

    if (process.env.NODE_ENV === 'development') {
      response.stack = error.stack;
    }

    ctx.json(response, statusCode);
  }
});
```

### API-Specific Formats

```typescript
// REST API format
ctx.json({
  success: false,
  error: {
    code: 'USER_NOT_FOUND',
    message: 'User not found',
    statusCode: 404
  }
}, 404);

// JSON:API format
ctx.json({
  errors: [{
    status: '404',
    title: 'Not Found',
    detail: 'User not found',
    source: { pointer: '/users/123' }
  }]
}, 404);
```

---

## Error Handling Patterns

Common patterns for handling specific errors.

### Not Found

```typescript
app.get('/users/:id', async (ctx) => {
  const user = await database.findUser(ctx.params.id);

  if (!user) {
    throw new HTTPError(404, 'User not found', {
      userId: ctx.params.id
    });
  }

  ctx.json({ user });
});
```

### Unauthorized

```typescript
const requireAuth: Middleware = async (ctx, next) => {
  const token = ctx.headers['authorization'];

  if (!token) {
    throw new HTTPError(401, 'Authentication required', {
      message: 'Please provide an authorization token'
    });
  }

  await next();
};
```

### Forbidden

```typescript
app.delete('/users/:id', async (ctx) => {
  const currentUser = ctx.state.user;
  const targetUserId = ctx.params.id;

  if (currentUser.id !== targetUserId && currentUser.role !== 'admin') {
    throw new HTTPError(403, 'Forbidden', {
      message: 'You do not have permission to delete this user'
    });
  }

  await database.deleteUser(targetUserId);
  ctx.json({ message: 'User deleted' });
});
```

### Conflict

```typescript
app.post('/users', async (ctx) => {
  const { email } = ctx.body;

  const existing = await database.findByEmail(email);

  if (existing) {
    throw new HTTPError(409, 'Email already exists', {
      field: 'email',
      existingUserId: existing.id
    });
  }

  const user = await database.createUser(ctx.body);
  ctx.json({ user }, 201);
});
```

### Rate Limit

```typescript
const rateLimit: Middleware = async (ctx, next) => {
  const remaining = checkRateLimit(ctx);

  if (remaining <= 0) {
    throw new HTTPError(429, 'Too many requests', {
      message: 'Rate limit exceeded. Please try again later.',
      retryAfter: 60
    });
  }

  await next();
};
```

### Service Unavailable

```typescript
app.get('/external-data', async (ctx) => {
  try {
    const data = await externalAPI.fetch();
    ctx.json({ data });
  } catch (error) {
    if (error.code === 'ETIMEDOUT' || error.code === 'ECONNREFUSED') {
      throw new HTTPError(503, 'Service temporarily unavailable', {
        message: 'External service is not responding',
        retryAfter: 30
      });
    }

    throw error;
  }
});
```

---

## Best Practices

### 1. Use Appropriate Status Codes

```typescript
// Good - specific status codes
throw new HTTPError(404, 'Not found');
throw new HTTPError(409, 'Conflict');
throw new HTTPError(422, 'Unprocessable entity');

// Not recommended - generic 400
throw new HTTPError(400, 'Error');
```

### 2. Provide Helpful Error Messages

```typescript
// Good - actionable message
throw new HTTPError(400, 'Email is required and must be a valid email address');

// Not helpful
throw new HTTPError(400, 'Invalid input');
```

### 3. Include Relevant Details

```typescript
// Good
throw new HTTPError(404, 'User not found', {
  userId: ctx.params.id,
  searchedBy: 'id'
});

// Minimal
throw new HTTPError(404, 'Not found');
```

### 4. Don't Expose Sensitive Information

```typescript
// Good
throw new HTTPError(401, 'Invalid credentials');

// Bad - exposes implementation details
throw new HTTPError(401, 'Password hash mismatch for user@example.com');
```

### 5. Log Errors Appropriately

```typescript
const app = createApp({
  onError: async (error, ctx) => {
    // Log all errors
    logger.error('Request error', {
      error: error.message,
      stack: error.stack,
      path: ctx.path,
    });

    // Don't expose internal errors to client
    const isClientError = error instanceof HTTPError && error.statusCode < 500;

    ctx.json({
      error: true,
      message: isClientError ? error.message : 'Internal server error'
    }, error instanceof HTTPError ? error.statusCode : 500);
  }
});
```

### 6. Handle Async Errors

```typescript
// Good - errors propagate automatically
app.get('/data', async (ctx) => {
  const data = await fetchData(); // Throws error
  ctx.json({ data });
});

// Unnecessary try-catch
app.get('/data', async (ctx) => {
  try {
    const data = await fetchData();
    ctx.json({ data });
  } catch (error) {
    throw error; // Redundant
  }
});
```

### 7. Create Error Constants

```typescript
// errors/constants.ts
export const ERRORS = {
  USER_NOT_FOUND: {
    statusCode: 404,
    message: 'User not found',
    code: 'USER_NOT_FOUND'
  },
  EMAIL_EXISTS: {
    statusCode: 409,
    message: 'Email already registered',
    code: 'EMAIL_EXISTS'
  },
  UNAUTHORIZED: {
    statusCode: 401,
    message: 'Authentication required',
    code: 'UNAUTHORIZED'
  }
} as const;

// Usage
import { ERRORS } from './errors/constants.js';

throw new HTTPError(
  ERRORS.USER_NOT_FOUND.statusCode,
  ERRORS.USER_NOT_FOUND.message,
  { code: ERRORS.USER_NOT_FOUND.code }
);
```

### 8. Test Error Paths

```typescript
// Test that errors are thrown correctly
describe('GET /users/:id', () => {
  it('should return 404 when user not found', async () => {
    const response = await request(app)
      .get('/users/nonexistent')
      .expect(404);

    expect(response.body).toEqual({
      error: true,
      message: 'User not found'
    });
  });
});
```

---

## Common HTTP Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 400 | Bad Request | Invalid request format |
| 401 | Unauthorized | Authentication required or failed |
| 403 | Forbidden | User doesn't have permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource already exists |
| 422 | Unprocessable Entity | Validation failed (semantic errors) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server error |
| 502 | Bad Gateway | Upstream service error |
| 503 | Service Unavailable | Temporary unavailability |

---

## Next Steps

- [Learn about Configuration](configuration.md)
- [Explore Validation](validation.md)
- [See Error Handling Examples](../examples/todo-api.md)
- [Check API Reference](../api-reference/server.md)

---

**Need help?** See the [Troubleshooting Guide](../guides/troubleshooting.md) for common issues.
