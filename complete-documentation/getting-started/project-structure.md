# Project Structure

Learn how to organize your RamAPI project for scalability and maintainability. This guide covers recommended folder structures, configuration files, and best practices.

## Basic Project Structure

For small to medium applications:

```
my-ramapi-app/
├── src/
│   ├── index.ts                 # Application entry point
│   ├── routes/                  # Route handlers
│   │   ├── users.ts
│   │   ├── posts.ts
│   │   └── auth.ts
│   ├── middleware/              # Custom middleware
│   │   ├── auth.ts
│   │   └── validation.ts
│   ├── models/                  # Data models/types
│   │   ├── user.ts
│   │   └── post.ts
│   ├── services/                # Business logic
│   │   ├── user.service.ts
│   │   └── email.service.ts
│   └── utils/                   # Utility functions
│       ├── db.ts
│       └── logger.ts
├── tests/                       # Test files
│   ├── routes/
│   └── services/
├── dist/                        # Compiled JavaScript (generated)
├── .env                         # Environment variables
├── .env.example                 # Environment template
├── .gitignore
├── package.json
├── tsconfig.json
└── README.md
```

## Enterprise Project Structure

For large-scale applications with microservices:

```
my-ramapi-enterprise/
├── src/
│   ├── index.ts                 # Entry point
│   ├── app.ts                   # App setup
│   ├── server.ts                # Server configuration
│   │
│   ├── config/                  # Configuration
│   │   ├── index.ts
│   │   ├── database.ts
│   │   ├── observability.ts
│   │   └── security.ts
│   │
│   ├── api/                     # API layer
│   │   ├── v1/                  # Versioned API
│   │   │   ├── index.ts
│   │   │   ├── users/
│   │   │   │   ├── routes.ts
│   │   │   │   ├── controller.ts
│   │   │   │   ├── service.ts
│   │   │   │   └── validation.ts
│   │   │   ├── posts/
│   │   │   └── auth/
│   │   └── v2/
│   │
│   ├── middleware/              # Custom middleware
│   │   ├── authentication.ts
│   │   ├── authorization.ts
│   │   ├── error-handler.ts
│   │   └── request-logger.ts
│   │
│   ├── models/                  # Data models
│   │   ├── user.model.ts
│   │   ├── post.model.ts
│   │   └── index.ts
│   │
│   ├── services/                # Business logic
│   │   ├── user.service.ts
│   │   ├── auth.service.ts
│   │   ├── email.service.ts
│   │   └── cache.service.ts
│   │
│   ├── repositories/            # Data access layer
│   │   ├── user.repository.ts
│   │   └── post.repository.ts
│   │
│   ├── database/                # Database
│   │   ├── connection.ts
│   │   ├── migrations/
│   │   └── seeds/
│   │
│   ├── types/                   # TypeScript types
│   │   ├── api.types.ts
│   │   ├── auth.types.ts
│   │   └── index.ts
│   │
│   ├── utils/                   # Utilities
│   │   ├── logger.ts
│   │   ├── validators.ts
│   │   └── helpers.ts
│   │
│   └── protocols/               # Multi-protocol support
│       ├── graphql/
│       │   ├── schema.ts
│       │   └── resolvers.ts
│       └── grpc/
│           ├── protos/
│           └── services.ts
│
├── tests/                       # Tests
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── scripts/                     # Build/deploy scripts
│   ├── build.sh
│   └── deploy.sh
│
├── docs/                        # Documentation
├── dist/                        # Compiled output
├── .env
├── .env.example
├── .env.test
├── .env.production
├── .gitignore
├── .dockerignore
├── Dockerfile
├── docker-compose.yml
├── package.json
├── tsconfig.json
└── README.md
```

## Entry Point Setup

### src/index.ts

Main entry point that starts the server:

```typescript
import { app } from './app.js';
import { config } from './config/index.js';

const PORT = config.port || 3000;
const HOST = config.host || '0.0.0.0';

app.listen(PORT, HOST, () => {
  console.log(`🚀 Server running on http://${HOST}:${PORT}`);
  console.log(`📊 Environment: ${process.env.NODE_ENV || 'development'}`);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully...');
  await app.close();
  process.exit(0);
});
```

### src/app.ts

Application setup and configuration:

```typescript
import { createApp } from 'ramapi';
import { config } from './config/index.js';
import { registerMiddleware } from './middleware/index.js';
import { registerRoutes } from './api/v1/index.js';

export const app = createApp({
  observability: config.observability,
  onError: async (error, ctx) => {
    console.error('Error:', error);

    const statusCode = error instanceof HTTPError ? error.statusCode : 500;
    ctx.json({ error: true, message: error.message }, statusCode);
  },
});

// Register global middleware
registerMiddleware(app);

// Register routes
registerRoutes(app);

export default app;
```

## Configuration Files

### config/index.ts

Centralized configuration management:

```typescript
import { config as dotenvConfig } from 'dotenv';

dotenvConfig();

export const config = {
  // Server
  port: parseInt(process.env.PORT || '3000', 10),
  host: process.env.HOST || '0.0.0.0',
  env: process.env.NODE_ENV || 'development',

  // Database
  database: {
    url: process.env.DATABASE_URL!,
    maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS || '10', 10),
  },

  // JWT
  jwt: {
    secret: process.env.JWT_SECRET!,
    expiresIn: parseInt(process.env.JWT_EXPIRES_IN || '86400', 10),
  },

  // CORS
  cors: {
    origin: process.env.CORS_ORIGIN?.split(',') || ['*'],
    credentials: true,
  },

  // Rate Limiting
  rateLimit: {
    maxRequests: parseInt(process.env.RATE_LIMIT_MAX || '100', 10),
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW || '60000', 10),
  },

  // Observability
  observability: {
    tracing: {
      enabled: process.env.TRACING_ENABLED === 'true',
      exporter: process.env.TRACING_EXPORTER || 'console',
      serviceName: process.env.SERVICE_NAME || 'ramapi-service',
    },
  },
};
```

### .env.example

Environment variables template:

```bash
# Server Configuration
NODE_ENV=development
PORT=3000
HOST=0.0.0.0

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
DB_MAX_CONNECTIONS=10

# JWT Authentication
JWT_SECRET=your-super-secret-key-change-this-in-production
JWT_EXPIRES_IN=86400

# CORS
CORS_ORIGIN=http://localhost:3001,https://myapp.com

# Rate Limiting
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=60000

# Observability
TRACING_ENABLED=true
TRACING_EXPORTER=console
SERVICE_NAME=my-api

# External Services
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=user@example.com
SMTP_PASS=password

# Redis Cache
REDIS_URL=redis://localhost:6379
```

## Route Organization

### Flat Route Structure (Small Apps)

```typescript
// src/routes/users.ts
import { Router } from 'ramapi';
import { validate } from 'ramapi/middleware';
import { createUserSchema } from '../validation/user.js';

export const userRoutes = (router: Router) => {
  router.get('/users', listUsers);
  router.get('/users/:id', getUser);
  router.post('/users', validate({ body: createUserSchema }), createUser);
  router.put('/users/:id', updateUser);
  router.delete('/users/:id', deleteUser);
};
```

### Modular Route Structure (Large Apps)

```typescript
// src/api/v1/users/routes.ts
import { Router } from 'ramapi';
import { UserController } from './controller.js';
import { userValidation } from './validation.js';
import { authenticate } from '../../../middleware/authentication.js';

export const registerUserRoutes = (router: Router) => {
  const controller = new UserController();

  router.group('/users', (users) => {
    users.use(authenticate);

    users.get('/', controller.list);
    users.get('/:id', controller.get);
    users.post('/', userValidation.create, controller.create);
    users.put('/:id', userValidation.update, controller.update);
    users.delete('/:id', controller.delete);
  });
};
```

### Controller Pattern

```typescript
// src/api/v1/users/controller.ts
import { Context } from 'ramapi';
import { UserService } from './service.js';

export class UserController {
  private service: UserService;

  constructor() {
    this.service = new UserService();
  }

  list = async (ctx: Context) => {
    const users = await this.service.findAll();
    ctx.json({ users });
  };

  get = async (ctx: Context) => {
    const user = await this.service.findById(ctx.params.id);
    ctx.json({ user });
  };

  create = async (ctx: Context) => {
    const user = await this.service.create(ctx.body);
    ctx.json({ user }, 201);
  };

  update = async (ctx: Context) => {
    const user = await this.service.update(ctx.params.id, ctx.body);
    ctx.json({ user });
  };

  delete = async (ctx: Context) => {
    await this.service.delete(ctx.params.id);
    ctx.status(204);
  };
}
```

## Service Layer

Separate business logic from route handlers:

```typescript
// src/services/user.service.ts
import { UserRepository } from '../repositories/user.repository.js';
import { User, CreateUserDTO } from '../types/user.types.js';
import { HTTPError } from 'ramapi';

export class UserService {
  private repository: UserRepository;

  constructor() {
    this.repository = new UserRepository();
  }

  async findAll(): Promise<User[]> {
    return this.repository.findAll();
  }

  async findById(id: string): Promise<User> {
    const user = await this.repository.findById(id);

    if (!user) {
      throw new HTTPError(404, 'User not found');
    }

    return user;
  }

  async create(data: CreateUserDTO): Promise<User> {
    // Business logic
    const existingUser = await this.repository.findByEmail(data.email);

    if (existingUser) {
      throw new HTTPError(400, 'Email already exists');
    }

    return this.repository.create(data);
  }

  async update(id: string, data: Partial<User>): Promise<User> {
    const user = await this.findById(id);
    return this.repository.update(id, data);
  }

  async delete(id: string): Promise<void> {
    await this.findById(id);
    await this.repository.delete(id);
  }
}
```

## Middleware Organization

```typescript
// src/middleware/index.ts
import { Server } from 'ramapi';
import { logger, cors, rateLimit } from 'ramapi/middleware';
import { config } from '../config/index.js';
import { errorHandler } from './error-handler.js';
import { requestId } from './request-id.js';

export const registerMiddleware = (app: Server) => {
  // Request ID
  app.use(requestId());

  // Logging
  app.use(logger());

  // CORS
  app.use(cors(config.cors));

  // Rate limiting
  app.use(rateLimit(config.rateLimit));

  // Error handling (should be last)
  app.use(errorHandler);
};
```

## TypeScript Configuration

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["./src/*"],
      "@/config/*": ["./src/config/*"],
      "@/api/*": ["./src/api/*"],
      "@/services/*": ["./src/services/*"],
      "@/utils/*": ["./src/utils/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

## Package.json Scripts

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc && tsc-alias",
    "start": "node dist/index.js",
    "test": "vitest",
    "test:coverage": "vitest --coverage",
    "test:e2e": "vitest run tests/e2e",
    "type-check": "tsc --noEmit",
    "lint": "eslint src/**/*.ts",
    "format": "prettier --write \"src/**/*.ts\"",
    "db:migrate": "tsx scripts/migrate.ts",
    "db:seed": "tsx scripts/seed.ts"
  }
}
```

## Docker Configuration

### Dockerfile

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY --from=builder /app/dist ./dist

EXPOSE 3000

USER node

CMD ["node", "dist/index.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## Best Practices

### 1. Separation of Concerns

Keep routes, controllers, services, and repositories separate.

### 2. Environment-based Configuration

Use different `.env` files for different environments:
- `.env.development`
- `.env.test`
- `.env.production`

### 3. Error Handling

Centralize error handling logic.

### 4. Validation

Keep validation schemas close to routes or in dedicated files.

### 5. Type Safety

Define TypeScript types/interfaces for all data structures.

### 6. Testing

Organize tests to mirror your source structure.

### 7. Documentation

Document complex logic and API endpoints.

## Anti-Patterns to Avoid

- Don't put business logic in route handlers
- Don't hardcode configuration values
- Don't mix concerns (routes, logic, data access)
- Don't ignore error handling
- Don't skip input validation
- Don't commit `.env` files

## Next Steps

- [Learn about Context & Handlers](../core-concepts/context-and-handlers.md)
- [Understand Routing](../core-concepts/routing.md)
- [Explore Middleware](../core-concepts/middleware.md)
- [See Complete Examples](../examples/todo-api.md)

---

**Ready to dive deeper?** Explore [Core Concepts](../core-concepts/context-and-handlers.md)
