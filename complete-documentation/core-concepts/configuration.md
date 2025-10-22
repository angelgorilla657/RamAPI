# Configuration

Proper configuration is essential for production applications. This guide covers server configuration, environment variables, and best practices for managing application settings.

## Table of Contents

1. [Server Configuration](#server-configuration)
2. [Environment Variables](#environment-variables)
3. [Configuration Files](#configuration-files)
4. [CORS Configuration](#cors-configuration)
5. [Observability Configuration](#observability-configuration)
6. [Adapter Configuration](#adapter-configuration)
7. [Environment-Specific Config](#environment-specific-config)
8. [Best Practices](#best-practices)

---

## Server Configuration

Configure your RamAPI server using the `ServerConfig` interface.

### Basic Configuration

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  port: 3000,
  host: '0.0.0.0',
});

app.listen();
```

### Full Configuration

```typescript
import { createApp, logger, cors } from 'ramapi';

const app = createApp({
  // Server settings
  port: 3000,
  host: '0.0.0.0',

  // Global middleware
  middleware: [
    logger(),
    cors(),
  ],

  // Error handler
  onError: async (error, ctx) => {
    console.error('Error:', error);
    ctx.json({
      error: true,
      message: error.message
    }, error instanceof HTTPError ? error.statusCode : 500);
  },

  // Not found handler
  onNotFound: async (ctx) => {
    ctx.json({
      error: true,
      message: 'Route not found'
    }, 404);
  },

  // CORS configuration
  cors: {
    origin: ['https://example.com'],
    credentials: true,
  },

  // Observability
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'console',
    },
  },

  // Adapter configuration (Phase 3.4)
  adapter: {
    type: 'uwebsockets', // or 'node-http'
  },
});
```

### ServerConfig Interface

```typescript
interface ServerConfig {
  port?: number;                    // Default: 3000
  host?: string;                    // Default: '0.0.0.0'
  cors?: boolean | CorsConfig;      // CORS configuration
  middleware?: Middleware[];        // Global middleware
  onError?: ErrorHandler;           // Custom error handler
  onNotFound?: Handler;             // Custom 404 handler
  observability?: ObservabilityConfig; // Tracing, metrics, profiling
  adapter?: AdapterConfig;          // HTTP adapter selection
}
```

---

## Environment Variables

Use environment variables for configuration that changes between environments.

### Basic .env File

```bash
# .env

# Server
NODE_ENV=development
PORT=3000
HOST=0.0.0.0

# Security
JWT_SECRET=your-super-secret-key-change-in-production
JWT_EXPIRES_IN=86400

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
DATABASE_MAX_CONNECTIONS=10

# CORS
CORS_ORIGIN=http://localhost:3001,https://example.com

# Rate Limiting
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=60000

# Observability
TRACING_ENABLED=true
TRACING_EXPORTER=console
SERVICE_NAME=my-api
```

### Loading Environment Variables

```bash
npm install dotenv
```

```typescript
import { config as loadEnv } from 'dotenv';

// Load .env file
loadEnv();

// Access environment variables
const port = parseInt(process.env.PORT || '3000', 10);
const jwtSecret = process.env.JWT_SECRET!;
const isDevelopment = process.env.NODE_ENV === 'development';
```

### Type-Safe Environment Variables

```typescript
// config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.string().regex(/^\d+$/).transform(Number).default('3000'),
  HOST: z.string().default('0.0.0.0'),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().regex(/^\d+$/).transform(Number).default('86400'),
  DATABASE_URL: z.string().url(),
  CORS_ORIGIN: z.string().transform((val) => val.split(',')),
  TRACING_ENABLED: z.string().transform((val) => val === 'true').default('false'),
});

export const env = envSchema.parse(process.env);

// Usage: env.PORT, env.JWT_SECRET, etc.
```

---

## Configuration Files

Organize configuration in dedicated files.

### config/index.ts

```typescript
import { config as loadEnv } from 'dotenv';

loadEnv();

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

### Usage

```typescript
import { createApp } from 'ramapi';
import { config } from './config/index.js';

const app = createApp({
  port: config.port,
  host: config.host,
  cors: config.cors,
  observability: config.observability,
});
```

### Modular Configuration

```typescript
// config/database.ts
export const databaseConfig = {
  url: process.env.DATABASE_URL!,
  maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS || '10', 10),
  ssl: process.env.DB_SSL === 'true',
};

// config/jwt.ts
export const jwtConfig = {
  secret: process.env.JWT_SECRET!,
  expiresIn: parseInt(process.env.JWT_EXPIRES_IN || '86400', 10),
  algorithm: 'HS256' as const,
};

// config/cors.ts
export const corsConfig = {
  origin: process.env.CORS_ORIGIN?.split(',') || ['*'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
};

// config/index.ts
export { databaseConfig } from './database.js';
export { jwtConfig } from './jwt.js';
export { corsConfig } from './cors.js';
```

---

## CORS Configuration

Configure Cross-Origin Resource Sharing.

### Simple CORS

```typescript
import { createApp, cors } from 'ramapi';

const app = createApp();

// Allow all origins
app.use(cors());
```

### Configured CORS

```typescript
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Request-ID'],
  credentials: true,
  maxAge: 86400, // 24 hours
}));
```

### Dynamic Origin

```typescript
app.use(cors({
  origin: (requestOrigin) => {
    // Allow localhost in development
    if (process.env.NODE_ENV === 'development') {
      return requestOrigin.includes('localhost');
    }

    // Production whitelist
    const allowedOrigins = [
      'https://example.com',
      'https://app.example.com',
    ];

    return allowedOrigins.includes(requestOrigin);
  },
  credentials: true,
}));
```

### Server-Level CORS

```typescript
const app = createApp({
  cors: {
    origin: ['https://example.com'],
    credentials: true,
  },
});
```

---

## Observability Configuration

Configure tracing, profiling, and metrics.

### Basic Observability

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'console', // or 'jaeger', 'zipkin', 'otlp'
    },
  },
});
```

### Full Observability

```typescript
const app = createApp({
  observability: {
    // Distributed tracing
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'jaeger',
      endpoint: 'http://localhost:14268/api/traces',
      sampleRate: 1.0, // 100% sampling
      maxSpanAttributes: 128,
      redactHeaders: ['authorization', 'cookie'],
    },

    // Performance profiling
    profiling: {
      enabled: true,
      captureStackTraces: true,
    },

    // Metrics collection
    metrics: {
      enabled: true,
    },

    // Structured logging
    logging: {
      level: 'info',
      format: 'json',
    },
  },
});
```

### Environment-Based Observability

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: process.env.NODE_ENV === 'production',
      serviceName: process.env.SERVICE_NAME || 'my-api',
      exporter: process.env.TRACING_EXPORTER as 'jaeger' | 'zipkin' | 'console',
      endpoint: process.env.TRACING_ENDPOINT,
      sampleRate: parseFloat(process.env.TRACING_SAMPLE_RATE || '0.1'),
    },
  },
});
```

---

## Adapter Configuration

Configure HTTP server adapters (Phase 3.4).

### Automatic Selection

```typescript
const app = createApp();
// Automatically selects uWebSockets if available, falls back to Node.js HTTP
```

### Explicit Adapter

```typescript
const app = createApp({
  adapter: {
    type: 'uwebsockets', // or 'node-http'
  },
});
```

### Adapter with Options

```typescript
const app = createApp({
  adapter: {
    type: 'uwebsockets',
    ssl: {
      key_file_name: './server.key',
      cert_file_name: './server.crt',
    },
  },
});
```

---

## Environment-Specific Config

Different configurations for different environments.

### Multiple .env Files

```
.env                 # Defaults
.env.development     # Development overrides
.env.production      # Production overrides
.env.test            # Test overrides
```

### Loading Environment-Specific Config

```typescript
import { config as loadEnv } from 'dotenv';
import path from 'path';

const env = process.env.NODE_ENV || 'development';

// Load base .env
loadEnv();

// Load environment-specific .env
loadEnv({
  path: path.resolve(`.env.${env}`),
  override: true,
});
```

### Configuration by Environment

```typescript
const baseConfig = {
  port: 3000,
  host: '0.0.0.0',
};

const developmentConfig = {
  ...baseConfig,
  observability: {
    tracing: {
      enabled: true,
      exporter: 'console',
      sampleRate: 1.0,
    },
  },
};

const productionConfig = {
  ...baseConfig,
  observability: {
    tracing: {
      enabled: true,
      exporter: 'jaeger',
      endpoint: process.env.JAEGER_ENDPOINT,
      sampleRate: 0.1, // 10% sampling
    },
  },
};

const env = process.env.NODE_ENV || 'development';
export const config = env === 'production'
  ? productionConfig
  : developmentConfig;
```

---

## Best Practices

### 1. Never Commit .env Files

```bash
# .gitignore
.env
.env.local
.env.*.local
```

### 2. Provide .env.example

```bash
# .env.example
NODE_ENV=development
PORT=3000
JWT_SECRET=change-this-to-a-secure-random-string
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
```

### 3. Validate Configuration on Startup

```typescript
function validateConfig() {
  const required = ['JWT_SECRET', 'DATABASE_URL'];

  for (const key of required) {
    if (!process.env[key]) {
      throw new Error(`Missing required environment variable: ${key}`);
    }
  }

  if (process.env.JWT_SECRET && process.env.JWT_SECRET.length < 32) {
    throw new Error('JWT_SECRET must be at least 32 characters');
  }
}

validateConfig();
```

### 4. Use Type-Safe Configuration

```typescript
import { z } from 'zod';

const configSchema = z.object({
  port: z.number().int().min(1).max(65535),
  jwtSecret: z.string().min(32),
  databaseUrl: z.string().url(),
});

export const config = configSchema.parse({
  port: parseInt(process.env.PORT!),
  jwtSecret: process.env.JWT_SECRET!,
  databaseUrl: process.env.DATABASE_URL!,
});
```

### 5. Centralize Configuration

```typescript
// Good - centralized
import { config } from './config/index.js';
const port = config.port;

// Not recommended - scattered
const port = parseInt(process.env.PORT || '3000');
```

### 6. Document Configuration Options

```typescript
/**
 * Application configuration
 *
 * Environment variables:
 * - PORT: Server port (default: 3000)
 * - HOST: Server host (default: 0.0.0.0)
 * - JWT_SECRET: Secret key for JWT signing (required)
 * - DATABASE_URL: PostgreSQL connection string (required)
 */
export const config = {
  // ...
};
```

### 7. Use Defaults Wisely

```typescript
// Good - sensible defaults
const port = parseInt(process.env.PORT || '3000', 10);

// Good - required values
const jwtSecret = process.env.JWT_SECRET!;
if (!jwtSecret) {
  throw new Error('JWT_SECRET is required');
}

// Not recommended - default for secrets
const jwtSecret = process.env.JWT_SECRET || 'default-secret'; // Insecure!
```

### 8. Separate Public and Secret Config

```typescript
// Public config (can be logged)
export const publicConfig = {
  port: config.port,
  host: config.host,
  environment: config.env,
};

// Secret config (never log)
export const secretConfig = {
  jwtSecret: config.jwt.secret,
  databaseUrl: config.database.url,
};

// Safe to log
console.log('Starting server:', publicConfig);

// Never log this!
// console.log(secretConfig.jwtSecret);
```

---

## Configuration Examples

### Minimal Production Config

```typescript
import { createApp, logger, cors } from 'ramapi';
import { config } from './config/index.js';

const app = createApp({
  port: config.port,
  host: config.host,
  middleware: [logger(), cors(config.cors)],
  observability: config.observability,
  onError: async (error, ctx) => {
    console.error('Error:', error);
    ctx.json({
      error: true,
      message: process.env.NODE_ENV === 'production'
        ? 'Internal server error'
        : error.message
    }, error instanceof HTTPError ? error.statusCode : 500);
  },
});
```

### Development Config

```typescript
const app = createApp({
  port: 3000,
  host: 'localhost',
  middleware: [
    logger(),
    cors({ origin: '*' }), // Allow all in development
  ],
  observability: {
    tracing: {
      enabled: true,
      exporter: 'console',
      sampleRate: 1.0, // Sample everything
    },
  },
  onError: async (error, ctx) => {
    console.error('Error:', error);
    ctx.json({
      error: true,
      message: error.message,
      stack: error.stack, // Include stack in development
    }, error instanceof HTTPError ? error.statusCode : 500);
  },
});
```

---

## Next Steps

- [Learn about Middleware](middleware.md)
- [Explore Observability](../observability/overview.md)
- [See Deployment Guide](../deployment/production-setup.md)
- [Check Complete Examples](../examples/todo-api.md)

---

**Need help?** See the [Troubleshooting Guide](../guides/troubleshooting.md) for common configuration issues.
