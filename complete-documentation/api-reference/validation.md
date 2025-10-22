# Validation API Reference

Complete API reference for RamAPI's request validation using Zod schemas.

## Table of Contents

1. [validate()](#validate)
2. [createHandler()](#createhandler)
3. [ValidationSchema](#validationschema)
4. [ValidationError](#validationerror)
5. [Type Safety](#type-safety)
6. [Examples](#examples)

---

## validate()

Create validation middleware from Zod schemas.

### Signature

```typescript
function validate(schema: ValidationSchema): Middleware
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schema` | `ValidationSchema` | Yes | Validation schema for body, query, and/or params |

### Returns

`Middleware` - Validation middleware

### ValidationSchema

```typescript
interface ValidationSchema {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}
```

### Behavior

- Validates `body`, `query`, and/or `params` against Zod schemas
- Replaces context properties with validated/transformed data
- Throws `HTTPError` (400) if validation fails
- Includes detailed error messages in response

### Examples

**Validate body:**

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  age: z.number().int().min(18).max(120),
});

app.post('/users',
  validate({ body: userSchema }),
  async (ctx) => {
    // ctx.body is validated and typed
    const { name, email, age } = ctx.body;
    ctx.json({ user: ctx.body }, 201);
  }
);
```

**Validate query:**

```typescript
const paginationSchema = z.object({
  page: z.string().transform(Number).pipe(z.number().int().min(1)),
  limit: z.string().transform(Number).pipe(z.number().int().min(1).max(100)),
  sort: z.enum(['asc', 'desc']).optional(),
});

app.get('/users',
  validate({ query: paginationSchema }),
  async (ctx) => {
    const { page, limit, sort } = ctx.query;
    // page and limit are numbers, sort is 'asc' | 'desc' | undefined
  }
);
```

**Validate params:**

```typescript
const idSchema = z.object({
  id: z.string().regex(/^\d+$/).transform(Number),
});

app.get('/users/:id',
  validate({ params: idSchema }),
  async (ctx) => {
    const userId: number = ctx.params.id; // Typed as number
  }
);
```

**Validate all:**

```typescript
app.put('/users/:id',
  validate({
    params: z.object({
      id: z.string().regex(/^\d+$/).transform(Number),
    }),
    body: z.object({
      name: z.string().min(2),
      email: z.string().email(),
    }),
    query: z.object({
      notify: z.string().transform(Boolean).optional(),
    }),
  }),
  async (ctx) => {
    const userId = ctx.params.id;
    const { name, email } = ctx.body;
    const notify = ctx.query.notify;
  }
);
```

### Validation Errors

When validation fails, a 400 error is thrown with details:

```json
{
  "error": true,
  "message": "Validation failed",
  "details": {
    "errors": [
      {
        "field": "body.email",
        "message": "Invalid email",
        "code": "invalid_string"
      },
      {
        "field": "query.page",
        "message": "Expected number, received nan",
        "code": "invalid_type"
      }
    ]
  }
}
```

---

## createHandler()

Create type-safe route handlers with validation.

### Signature

```typescript
function createHandler<TBody, TQuery, TParams>(
  schema: ValidationSchema,
  handler: (ctx: {
    body: TBody;
    query: TQuery;
    params: TParams;
    [key: string]: unknown;
  }) => void | Promise<void>
): Middleware[]
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `schema` | `ValidationSchema` | Validation schema |
| `handler` | `Function` | Route handler with typed context |

### Returns

`Middleware[]` - Array of middleware (validation + handler)

### Example

```typescript
import { createHandler } from 'ramapi';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

const createUser = createHandler(
  { body: userSchema },
  async (ctx) => {
    // ctx.body is fully typed
    const { name, email } = ctx.body;
    ctx.json({ user: { name, email } }, 201);
  }
);

app.post('/users', ...createUser);
```

**With all schemas:**

```typescript
const updateUser = createHandler(
  {
    params: z.object({
      id: z.string().transform(Number),
    }),
    body: z.object({
      name: z.string(),
      email: z.string().email(),
    }),
    query: z.object({
      notify: z.string().transform(Boolean).optional(),
    }),
  },
  async (ctx) => {
    const userId = ctx.params.id; // number
    const { name, email } = ctx.body; // { name: string, email: string }
    const notify = ctx.query.notify; // boolean | undefined
  }
);

app.put('/users/:id', ...updateUser);
```

---

## ValidationSchema

Interface for validation schemas.

### Type Definition

```typescript
interface ValidationSchema {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `body` | `ZodSchema` | Schema for request body |
| `query` | `ZodSchema` | Schema for query parameters |
| `params` | `ZodSchema` | Schema for route parameters |

### Example Schemas

**User registration:**

```typescript
const registrationSchema: ValidationSchema = {
  body: z.object({
    email: z.string().email(),
    password: z.string().min(8),
    name: z.string().min(2).max(100),
  }),
};
```

**Pagination:**

```typescript
const paginationSchema: ValidationSchema = {
  query: z.object({
    page: z.string().transform(Number).pipe(z.number().int().min(1)),
    limit: z.string().transform(Number).pipe(z.number().int().min(1).max(100)),
    search: z.string().optional(),
  }),
};
```

**ID parameter:**

```typescript
const idParamSchema: ValidationSchema = {
  params: z.object({
    id: z.string().uuid(),
  }),
};
```

---

## ValidationError

Error details when validation fails.

### Type Definition

```typescript
interface ValidationError {
  field: string;
  message: string;
  code: string;
}
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `field` | `string` | Field path (e.g., `body.email`, `query.page`) |
| `message` | `string` | Human-readable error message |
| `code` | `string` | Zod error code (e.g., `invalid_type`, `too_small`) |

### Example Response

```json
{
  "error": true,
  "message": "Validation failed",
  "details": {
    "errors": [
      {
        "field": "body.email",
        "message": "Invalid email",
        "code": "invalid_string"
      },
      {
        "field": "body.age",
        "message": "Number must be greater than or equal to 18",
        "code": "too_small"
      }
    ]
  }
}
```

---

## Type Safety

### Inferring Types from Schemas

```typescript
import { z } from 'zod';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int(),
});

type User = z.infer<typeof userSchema>;
// { name: string; email: string; age: number }

app.post('/users',
  validate({ body: userSchema }),
  async (ctx: Context<User>) => {
    const user: User = ctx.body; // Fully typed
  }
);
```

### Type-Safe Handlers

```typescript
import { Context } from 'ramapi';

interface UserBody {
  name: string;
  email: string;
}

interface UserParams {
  id: string;
}

app.put('/users/:id',
  validate({
    params: z.object({ id: z.string() }),
    body: z.object({
      name: z.string(),
      email: z.string().email(),
    }),
  }),
  async (ctx: Context<UserBody, unknown, UserParams>) => {
    const userId = ctx.params.id; // string
    const { name, email } = ctx.body; // { name: string, email: string }
  }
);
```

---

## Examples

### Complete CRUD API

```typescript
import { createApp, validate } from 'ramapi';
import { z } from 'zod';

const app = createApp();

// Schemas
const userSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  age: z.number().int().min(18).max(120),
});

const idSchema = z.object({
  id: z.string().regex(/^\d+$/).transform(Number),
});

const paginationSchema = z.object({
  page: z.string().transform(Number).pipe(z.number().int().min(1)).default('1'),
  limit: z.string().transform(Number).pipe(z.number().int().min(1).max(100)).default('10'),
});

// Routes
app.get('/users',
  validate({ query: paginationSchema }),
  async (ctx) => {
    const { page, limit } = ctx.query;
    ctx.json({ users: [], page, limit });
  }
);

app.get('/users/:id',
  validate({ params: idSchema }),
  async (ctx) => {
    const userId = ctx.params.id;
    ctx.json({ user: { id: userId } });
  }
);

app.post('/users',
  validate({ body: userSchema }),
  async (ctx) => {
    const user = ctx.body;
    ctx.json({ user }, 201);
  }
);

app.put('/users/:id',
  validate({
    params: idSchema,
    body: userSchema.partial(), // All fields optional
  }),
  async (ctx) => {
    const userId = ctx.params.id;
    const updates = ctx.body;
    ctx.json({ user: { id: userId, ...updates } });
  }
);

app.delete('/users/:id',
  validate({ params: idSchema }),
  async (ctx) => {
    const userId = ctx.params.id;
    ctx.status(204);
  }
);

await app.listen(3000);
```

### Complex Validation

```typescript
// Nested objects
const addressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zipCode: z.string().regex(/^\d{5}$/),
  country: z.string().length(2), // ISO country code
});

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int().min(18),
  address: addressSchema,
  tags: z.array(z.string()),
});

// Refinements
const passwordSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
});

// Transformations
const querySchema = z.object({
  tags: z.string().transform((s) => s.split(',')),
  active: z.string().transform((s) => s === 'true'),
  sort: z.enum(['asc', 'desc']).default('asc'),
});

// Conditional validation
const createUserSchema = z.object({
  type: z.enum(['individual', 'business']),
  name: z.string(),
  companyName: z.string().optional(),
}).refine((data) => {
  if (data.type === 'business') {
    return !!data.companyName;
  }
  return true;
}, {
  message: 'Company name required for business accounts',
  path: ['companyName'],
});
```

### Custom Error Messages

```typescript
const userSchema = z.object({
  name: z.string({
    required_error: 'Name is required',
    invalid_type_error: 'Name must be a string',
  }).min(2, 'Name must be at least 2 characters'),

  email: z.string().email('Please provide a valid email address'),

  age: z.number({
    required_error: 'Age is required',
    invalid_type_error: 'Age must be a number',
  }).int('Age must be an integer').min(18, 'Must be 18 or older'),
});
```

---

## See Also

- [Validation Guide](../core-concepts/validation.md) - Validation guide
- [Zod Documentation](https://zod.dev) - Zod schema library
- [Middleware API](middleware-api.md) - Built-in middleware

---

**Need help?** Check the [Validation Guide](../core-concepts/validation.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
