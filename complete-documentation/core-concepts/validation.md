# Validation

RamAPI integrates seamlessly with Zod for type-safe, runtime validation. This guide covers everything you need to validate requests, transform data, and maintain type safety throughout your application.

## Table of Contents

1. [Why Validation?](#why-validation)
2. [Zod Basics](#zod-basics)
3. [Validation Middleware](#validation-middleware)
4. [Validation Schemas](#validation-schemas)
5. [Type Inference](#type-inference)
6. [Error Handling](#error-handling)
7. [Advanced Patterns](#advanced-patterns)
8. [Best Practices](#best-practices)

---

## Why Validation?

Input validation is crucial for:

- **Security** - Prevent injection attacks and malicious input
- **Data Integrity** - Ensure data matches expected format
- **Type Safety** - Runtime validation complements TypeScript
- **User Experience** - Provide clear, actionable error messages
- **Documentation** - Schemas serve as self-documenting contracts

### Without Validation

```typescript
app.post('/users', async (ctx) => {
  // What if email is missing? Invalid format? Wrong type?
  const { email, age } = ctx.body;

  // Runtime errors waiting to happen
  await database.createUser({ email, age });
});
```

### With Validation

```typescript
import { z } from 'zod';
import { validate } from 'ramapi';

const schema = z.object({
  email: z.string().email('Invalid email'),
  age: z.number().int().min(18, 'Must be 18+'),
});

app.post('/users',
  validate({ body: schema }),
  async (ctx) => {
    // Guaranteed to be valid
    const { email, age } = ctx.body as z.infer<typeof schema>;
    await database.createUser({ email, age });
  }
);
```

---

## Zod Basics

Zod is a TypeScript-first schema validation library.

### Installing Zod

```bash
npm install zod
```

### Basic Schemas

```typescript
import { z } from 'zod';

// String
const name = z.string();

// Number
const age = z.number();

// Boolean
const active = z.boolean();

// Object
const user = z.object({
  name: z.string(),
  age: z.number(),
});

// Array
const tags = z.array(z.string());

// Optional
const optional = z.string().optional();

// Nullable
const nullable = z.string().nullable();

// Default value
const withDefault = z.string().default('default');
```

### String Validation

```typescript
// Email
z.string().email()

// URL
z.string().url()

// UUID
z.string().uuid()

// Min/Max length
z.string().min(3).max(100)

// Regex
z.string().regex(/^[a-z]+$/)

// Trim whitespace
z.string().trim()

// Transform to lowercase
z.string().toLowerCase()

// Custom validation
z.string().refine(
  (val) => val.startsWith('http'),
  { message: 'Must start with http' }
)
```

### Number Validation

```typescript
// Integer
z.number().int()

// Positive
z.number().positive()

// Min/Max
z.number().min(0).max(100)

// Multiple of
z.number().multipleOf(5)

// Transform string to number
z.string().regex(/^\d+$/).transform(Number)
```

### Object Validation

```typescript
const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().int().min(18),
  role: z.enum(['user', 'admin']),
  tags: z.array(z.string()),
  metadata: z.record(z.string()), // Key-value pairs
});
```

---

## Validation Middleware

The `validate()` middleware validates request data.

### Basic Usage

```typescript
import { validate } from 'ramapi';
import { z } from 'zod';

const bodySchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

app.post('/register',
  validate({ body: bodySchema }),
  async (ctx) => {
    // ctx.body is now validated
    const { email, password } = ctx.body as z.infer<typeof bodySchema>;
    ctx.json({ message: 'User registered' }, 201);
  }
);
```

### Validate Body

```typescript
const createUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().int().min(18),
});

app.post('/users',
  validate({ body: createUserSchema }),
  async (ctx) => {
    const user = ctx.body as z.infer<typeof createUserSchema>;
    ctx.json({ user }, 201);
  }
);
```

### Validate Query Parameters

```typescript
const querySchema = z.object({
  page: z.string().regex(/^\d+$/).transform(Number).default('1'),
  limit: z.string().regex(/^\d+$/).transform(Number).default('10'),
  sort: z.enum(['asc', 'desc']).default('asc'),
});

app.get('/users',
  validate({ query: querySchema }),
  async (ctx) => {
    const { page, limit, sort } = ctx.query as z.infer<typeof querySchema>;
    // page and limit are numbers now (transformed)
    ctx.json({ page, limit, sort });
  }
);
```

### Validate Route Parameters

```typescript
const paramsSchema = z.object({
  id: z.string().uuid('Invalid user ID format'),
});

app.get('/users/:id',
  validate({ params: paramsSchema }),
  async (ctx) => {
    const { id } = ctx.params as z.infer<typeof paramsSchema>;
    ctx.json({ id });
  }
);
```

### Validate Multiple

```typescript
app.patch('/users/:id',
  validate({
    params: z.object({
      id: z.string().uuid(),
    }),
    body: z.object({
      name: z.string().min(2).optional(),
      email: z.string().email().optional(),
    }),
  }),
  async (ctx) => {
    const { id } = ctx.params as z.infer<typeof paramsSchema>;
    const updates = ctx.body;
    ctx.json({ message: 'Updated', id, updates });
  }
);
```

---

## Validation Schemas

Organize and reuse validation schemas.

### Schema File Organization

**schemas/user.schema.ts**
```typescript
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  age: z.number().int().min(18, 'Must be 18 or older'),
});

export const updateUserSchema = z.object({
  name: z.string().min(2).optional(),
  email: z.string().email().optional(),
  age: z.number().int().min(18).optional(),
});

export const userParamsSchema = z.object({
  id: z.string().uuid('Invalid user ID'),
});

// Type exports
export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

**routes/users.ts**
```typescript
import { validate } from 'ramapi';
import {
  createUserSchema,
  updateUserSchema,
  userParamsSchema,
} from '../schemas/user.schema.js';

app.post('/users',
  validate({ body: createUserSchema }),
  createUserHandler
);

app.patch('/users/:id',
  validate({
    params: userParamsSchema,
    body: updateUserSchema,
  }),
  updateUserHandler
);
```

### Nested Objects

```typescript
const addressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zipCode: z.string().regex(/^\d{5}$/),
  country: z.string().length(2), // ISO country code
});

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  address: addressSchema, // Nested schema
});
```

### Partial Schemas

```typescript
const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number(),
});

// All fields optional
const partialUserSchema = userSchema.partial();

// Specific fields optional
const updateUserSchema = userSchema.pick({
  name: true,
  email: true,
}).partial();
```

### Schema Composition

```typescript
const baseUserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

// Extend schema
const registrationSchema = baseUserSchema.extend({
  password: z.string().min(8),
  confirmPassword: z.string().min(8),
}).refine(
  (data) => data.password === data.confirmPassword,
  { message: 'Passwords must match', path: ['confirmPassword'] }
);

// Merge schemas
const schema1 = z.object({ a: z.string() });
const schema2 = z.object({ b: z.number() });
const merged = schema1.merge(schema2);
```

---

## Type Inference

Zod provides automatic TypeScript type inference.

### Basic Inference

```typescript
const userSchema = z.object({
  name: z.string(),
  age: z.number(),
});

// Infer TypeScript type from schema
type User = z.infer<typeof userSchema>;
// type User = { name: string; age: number }
```

### Using Inferred Types

```typescript
const createUserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number(),
});

type CreateUserInput = z.infer<typeof createUserSchema>;

app.post('/users',
  validate({ body: createUserSchema }),
  async (ctx) => {
    // Cast to inferred type
    const user = ctx.body as CreateUserInput;

    // Full type safety
    const name: string = user.name;
    const email: string = user.email;
    const age: number = user.age;

    ctx.json({ user }, 201);
  }
);
```

### Type-Safe Handlers

```typescript
import { Context } from 'ramapi';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

type UserInput = z.infer<typeof userSchema>;

async function createUser(ctx: Context<UserInput>) {
  // ctx.body is typed as UserInput
  const { name, email } = ctx.body;

  const user = await database.createUser({ name, email });
  ctx.json({ user }, 201);
}

app.post('/users',
  validate({ body: userSchema }),
  createUser
);
```

### Exported Types

```typescript
// schemas/user.schema.ts
export const userSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string().email(),
  createdAt: z.date(),
});

export type User = z.infer<typeof userSchema>;

// services/user.service.ts
import { User } from '../schemas/user.schema.js';

export class UserService {
  async createUser(data: Partial<User>): Promise<User> {
    // Implementation
  }
}
```

---

## Error Handling

Validation errors are automatically formatted.

### Validation Error Response

When validation fails, RamAPI returns a 400 error:

```typescript
// Request
POST /users
{
  "name": "J",
  "email": "invalid",
  "age": 15
}

// Response (400 Bad Request)
{
  "error": true,
  "message": "Validation failed",
  "details": {
    "errors": [
      {
        "field": "body.name",
        "message": "Name must be at least 2 characters",
        "code": "too_small"
      },
      {
        "field": "body.email",
        "message": "Invalid email address",
        "code": "invalid_string"
      },
      {
        "field": "body.age",
        "message": "Must be 18 or older",
        "code": "too_small"
      }
    ]
  }
}
```

### Custom Error Messages

```typescript
const schema = z.object({
  name: z.string().min(2, 'Name is too short'),
  email: z.string().email('Please provide a valid email'),
  age: z.number({
    required_error: 'Age is required',
    invalid_type_error: 'Age must be a number',
  }).min(18, 'You must be at least 18 years old'),
});
```

### Custom Validation Logic

```typescript
const passwordSchema = z.object({
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain uppercase letter')
    .regex(/[a-z]/, 'Password must contain lowercase letter')
    .regex(/[0-9]/, 'Password must contain number')
    .regex(/[^A-Za-z0-9]/, 'Password must contain special character'),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: 'Passwords do not match',
    path: ['confirmPassword'], // Error attached to confirmPassword field
  }
);
```

---

## Advanced Patterns

### Conditional Validation

```typescript
const schema = z.object({
  type: z.enum(['individual', 'company']),
  firstName: z.string().optional(),
  lastName: z.string().optional(),
  companyName: z.string().optional(),
}).refine(
  (data) => {
    if (data.type === 'individual') {
      return !!data.firstName && !!data.lastName;
    }
    if (data.type === 'company') {
      return !!data.companyName;
    }
    return false;
  },
  {
    message: 'Invalid fields for selected type',
  }
);
```

### Transform Data

```typescript
const querySchema = z.object({
  // String to number
  page: z.string().regex(/^\d+$/).transform(Number),

  // String to boolean
  active: z.string()
    .transform((val) => val === 'true')
    .pipe(z.boolean()),

  // Parse JSON string
  filters: z.string()
    .transform((val) => JSON.parse(val))
    .pipe(z.array(z.string())),

  // Trim and lowercase
  search: z.string().trim().toLowerCase(),
});
```

### Discriminated Unions

```typescript
const eventSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('click'),
    elementId: z.string(),
    x: z.number(),
    y: z.number(),
  }),
  z.object({
    type: z.literal('submit'),
    formId: z.string(),
    data: z.record(z.string()),
  }),
  z.object({
    type: z.literal('navigate'),
    url: z.string().url(),
  }),
]);
```

### Recursive Schemas

```typescript
type Category = {
  id: string;
  name: string;
  children?: Category[];
};

const categorySchema: z.ZodType<Category> = z.lazy(() =>
  z.object({
    id: z.string(),
    name: z.string(),
    children: z.array(categorySchema).optional(),
  })
);
```

---

## Best Practices

### 1. Validate Early

```typescript
// Good - validate at route level
app.post('/users',
  validate({ body: userSchema }),
  createUserHandler
);

// Not recommended - validate in handler
app.post('/users', async (ctx) => {
  const result = userSchema.safeParse(ctx.body);
  if (!result.success) {
    // Manual error handling
  }
});
```

### 2. Use Descriptive Error Messages

```typescript
// Good
z.string().min(8, 'Password must be at least 8 characters long')

// Not clear
z.string().min(8)
```

### 3. Organize Schemas in Separate Files

```typescript
// Good
// schemas/user.schema.ts
export const createUserSchema = z.object({ /* ... */ });
export const updateUserSchema = z.object({ /* ... */ });

// routes/users.ts
import { createUserSchema } from '../schemas/user.schema.js';
```

### 4. Export Types from Schemas

```typescript
// Good
export const userSchema = z.object({ /* ... */ });
export type User = z.infer<typeof userSchema>;

// Use everywhere
import { User } from './schemas/user.schema.js';
```

### 5. Use Transformations

```typescript
// Convert string query params to numbers
const querySchema = z.object({
  page: z.string().regex(/^\d+$/).transform(Number),
  limit: z.string().regex(/^\d+$/).transform(Number),
});
```

### 6. Reuse Common Patterns

```typescript
// Common patterns
const uuidSchema = z.string().uuid();
const emailSchema = z.string().email();
const timestampSchema = z.string().datetime();

// Reuse
const userSchema = z.object({
  id: uuidSchema,
  email: emailSchema,
  createdAt: timestampSchema,
});
```

### 7. Document Complex Validations

```typescript
/**
 * Password validation requirements:
 * - Minimum 8 characters
 * - At least one uppercase letter
 * - At least one lowercase letter
 * - At least one number
 * - At least one special character
 */
const passwordSchema = z.string()
  .min(8)
  .regex(/[A-Z]/)
  .regex(/[a-z]/)
  .regex(/[0-9]/)
  .regex(/[^A-Za-z0-9]/);
```

---

## Next Steps

- [Learn about Error Handling](error-handling.md)
- [Explore Middleware](middleware.md)
- [See Validation API Reference](../api-reference/validation.md)
- [Check Complete Examples](../examples/todo-api.md)

---

**Need help?** Check [Zod Documentation](https://zod.dev) for more validation patterns.
