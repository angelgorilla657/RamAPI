# Production Setup

Complete guide for deploying RamAPI applications to production environments.

## Table of Contents

1. [Environment Variables](#environment-variables)
2. [Configuration](#configuration)
3. [Security Checklist](#security-checklist)
4. [Performance Optimization](#performance-optimization)
5. [Health Checks](#health-checks)
6. [Graceful Shutdown](#graceful-shutdown)

---

## Environment Variables

### Required Variables

```bash
# Application
NODE_ENV=production
PORT=3000
HOST=0.0.0.0

# Security
JWT_SECRET=your-super-secret-key-min-32-characters-long
SESSION_SECRET=another-secret-key-for-sessions

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# CORS
CORS_ORIGIN=https://yourdomain.com

# Logging
LOG_LEVEL=info
LOG_FORMAT=json
```

### Optional Variables

```bash
# HTTP Adapter
HTTP_ADAPTER=uwebsockets  # or node-http

# Observability
TRACING_ENABLED=true
TRACING_EXPORTER=otlp
TRACING_ENDPOINT=http://localhost:4318
SERVICE_NAME=my-api
SERVICE_VERSION=1.0.0

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100

# File Upload
MAX_FILE_SIZE=10485760  # 10MB in bytes

# External Services
REDIS_URL=redis://localhost:6379
ELASTICSEARCH_URL=http://localhost:9200
```

### .env.example

Create this file in your repository:

```bash
# .env.example
# Copy this file to .env and fill in your values

# Application
NODE_ENV=development
PORT=3000
HOST=localhost

# Security (CHANGE THESE IN PRODUCTION!)
JWT_SECRET=change-me-to-a-random-string-min-32-chars
SESSION_SECRET=change-me-to-another-random-string

# Database
DATABASE_URL=postgresql://localhost:5432/mydb

# CORS
CORS_ORIGIN=http://localhost:5173

# Observability
TRACING_ENABLED=false
LOG_LEVEL=debug
```

### Loading Environment Variables

```typescript
// config.ts
import dotenv from 'dotenv';

// Load .env file
dotenv.config();

export const config = {
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '3000'),
  host: process.env.HOST || '0.0.0.0',

  // Security
  jwtSecret: process.env.JWT_SECRET!,
  sessionSecret: process.env.SESSION_SECRET!,

  // Database
  databaseUrl: process.env.DATABASE_URL!,

  // CORS
  corsOrigin: process.env.CORS_ORIGIN || '*',

  // Observability
  tracing: {
    enabled: process.env.TRACING_ENABLED === 'true',
    exporter: process.env.TRACING_EXPORTER || 'console',
    endpoint: process.env.TRACING_ENDPOINT,
    serviceName: process.env.SERVICE_NAME || 'ramapi-app',
    serviceVersion: process.env.SERVICE_VERSION || '1.0.0',
  },

  // Logging
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    format: process.env.LOG_FORMAT || 'json',
  },
};

// Validate required variables
function validateConfig() {
  const required = [
    'JWT_SECRET',
    'DATABASE_URL',
  ];

  const missing = required.filter((key) => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }

  // Validate JWT secret length
  if (config.jwtSecret.length < 32) {
    throw new Error('JWT_SECRET must be at least 32 characters long');
  }
}

if (config.env === 'production') {
  validateConfig();
}
```

---

## Configuration

### Production Server Configuration

```typescript
// index.ts
import { createApp } from 'ramapi';
import { config } from './config';

const app = createApp({
  // Use uWebSockets in production for 2-3x performance
  adapter: {
    type: process.env.HTTP_ADAPTER === 'node-http' ? 'node-http' : 'uwebsockets',
    options: {
      idleTimeout: 120,
      maxBackpressure: 1024 * 1024,
      maxPayloadLength: 10 * 1024 * 1024, // 10MB
    },
  },

  // CORS configuration
  cors: {
    origin: config.corsOrigin,
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
  },

  // Observability (production settings)
  observability: {
    tracing: {
      enabled: config.tracing.enabled,
      serviceName: config.tracing.serviceName,
      serviceVersion: config.tracing.serviceVersion,
      exporter: config.tracing.exporter as any,
      endpoint: config.tracing.endpoint,
      sampleRate: 0.1, // Sample 10% in production
    },
    logging: {
      enabled: true,
      level: config.logging.level as any,
      format: config.logging.format as any,
      redactFields: ['password', 'token', 'apiKey', 'ssn', 'creditCard'],
    },
    metrics: {
      enabled: true,
      collectInterval: 60000,
    },
  },

  // Error handler
  onError: async (error, ctx) => {
    // Log error (don't expose details to client in production)
    console.error('Error:', {
      message: error.message,
      stack: error.stack,
      path: ctx.path,
      method: ctx.method,
    });

    // Send generic error to client
    ctx.json({
      error: 'Internal server error',
      ...(config.env !== 'production' && { details: error.message }),
    }, 500);
  },
});

// Start server
const server = await app.listen(config.port, config.host);
console.log(`Server running on ${config.host}:${config.port}`);
```

---

## Security Checklist

### ✅ Pre-Deployment Checklist

#### Environment & Secrets

- [ ] All secrets in environment variables (not hardcoded)
- [ ] `.env` file in `.gitignore`
- [ ] JWT secret is at least 32 characters
- [ ] Different secrets for dev/staging/production
- [ ] Secrets stored in secure vault (AWS Secrets Manager, etc.)

#### HTTPS/TLS

- [ ] HTTPS enabled (use reverse proxy like nginx)
- [ ] TLS 1.2 or higher
- [ ] Valid SSL certificate
- [ ] HTTP redirects to HTTPS
- [ ] HSTS header enabled

#### CORS

- [ ] CORS restricted to specific origins (no `*` in production)
- [ ] Credentials allowed only for trusted origins
- [ ] Proper preflight handling

#### Rate Limiting

- [ ] Rate limiting enabled on all routes
- [ ] Stricter limits on auth endpoints
- [ ] IP-based or user-based limiting
- [ ] Redis for distributed rate limiting

#### Authentication & Authorization

- [ ] JWT tokens with expiration
- [ ] Password hashing with bcrypt (10+ rounds)
- [ ] Protected routes require authentication
- [ ] Role-based access control implemented
- [ ] Session management secure

#### Input Validation

- [ ] All inputs validated with Zod
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (sanitize inputs)
- [ ] File upload restrictions (type, size)

#### Headers & Security

```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
  },
}));
```

#### Database

- [ ] Database credentials in environment variables
- [ ] Connection pooling enabled
- [ ] Prepared statements for queries
- [ ] Database backups automated
- [ ] Database firewall rules configured

#### Logging & Monitoring

- [ ] No sensitive data in logs
- [ ] Centralized logging (CloudWatch, etc.)
- [ ] Error tracking (Sentry, Rollbar)
- [ ] Uptime monitoring
- [ ] Performance monitoring

#### Dependencies

- [ ] Dependencies up to date
- [ ] No known vulnerabilities (`npm audit`)
- [ ] Production dependencies only
- [ ] Lock file committed (`package-lock.json`)

---

## Performance Optimization

### 1. Use uWebSockets Adapter

```typescript
const app = createApp({
  adapter: { type: 'uwebsockets' }, // 2-3x faster
});
```

### 2. Enable Compression

```typescript
import compression from 'compression';

app.use(compression());
```

### 3. Database Connection Pooling

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20, // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

### 4. Caching Strategy

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// Cache middleware
const cache = (ttl: number) => {
  return async (ctx, next) => {
    const key = `cache:${ctx.path}:${JSON.stringify(ctx.query)}`;

    const cached = await redis.get(key);
    if (cached) {
      ctx.json(JSON.parse(cached));
      return;
    }

    await next();

    // Cache successful responses
    if (ctx.res.statusCode === 200) {
      await redis.setex(key, ttl, JSON.stringify(ctx.body));
    }
  };
};

app.get('/api/expensive', cache(300), async (ctx) => {
  // Expensive operation
});
```

### 5. Static File Serving

Use nginx or CDN for static files instead of Node.js.

```nginx
# nginx.conf
location /static/ {
    alias /var/www/static/;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

---

## Health Checks

### Basic Health Check

```typescript
app.get('/health', async (ctx) => {
  ctx.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.SERVICE_VERSION || '1.0.0',
  });
});
```

### Detailed Health Check

```typescript
app.get('/health/detailed', async (ctx) => {
  const health = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.SERVICE_VERSION,
    checks: {
      database: 'ok',
      redis: 'ok',
      external_api: 'ok',
    },
  };

  // Check database
  try {
    await db.query('SELECT 1');
  } catch (error) {
    health.status = 'degraded';
    health.checks.database = 'error';
  }

  // Check Redis
  try {
    await redis.ping();
  } catch (error) {
    health.status = 'degraded';
    health.checks.redis = 'error';
  }

  const statusCode = health.status === 'ok' ? 200 : 503;
  ctx.json(health, statusCode);
});
```

### Kubernetes Probes

```yaml
# kubernetes deployment
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/detailed
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Graceful Shutdown

### Implementation

```typescript
import { createApp } from 'ramapi';

const app = createApp();

// Register routes
app.get('/users', getUsers);

// Start server
await app.listen(3000);

// Graceful shutdown
const shutdown = async (signal: string) => {
  console.log(`${signal} received, starting graceful shutdown...`);

  // Stop accepting new connections
  await app.close();

  // Close database connections
  await db.end();

  // Close Redis connections
  await redis.quit();

  // Close any other resources
  // await closeOtherConnections();

  console.log('Graceful shutdown complete');
  process.exit(0);
};

// Handle shutdown signals
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

// Handle uncaught errors
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Don't exit - log and continue
});

process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  // Exit on uncaught exceptions
  process.exit(1);
});
```

### With Active Connections Tracking

```typescript
let activeConnections = 0;

app.use(async (ctx, next) => {
  activeConnections++;
  try {
    await next();
  } finally {
    activeConnections--;
  }
});

const shutdown = async (signal: string) => {
  console.log(`${signal} received, waiting for ${activeConnections} active connections...`);

  // Stop accepting new connections
  await app.close();

  // Wait for active connections to finish (with timeout)
  const timeout = setTimeout(() => {
    console.log('Shutdown timeout reached, forcing exit');
    process.exit(1);
  }, 30000); // 30 second timeout

  while (activeConnections > 0) {
    await new Promise((resolve) => setTimeout(resolve, 100));
  }

  clearTimeout(timeout);

  // Close other resources
  await db.end();

  console.log('Graceful shutdown complete');
  process.exit(0);
};
```

---

## Complete Production Example

```typescript
// index.ts
import { createApp, logger, rateLimit, cors } from 'ramapi';
import { config } from './config';
import { setupDatabase } from './database';
import { setupRedis } from './redis';
import routes from './routes';

async function main() {
  // Initialize dependencies
  const db = await setupDatabase(config.databaseUrl);
  const redis = await setupRedis(config.redisUrl);

  // Create app
  const app = createApp({
    adapter: { type: 'uwebsockets' },
    cors: {
      origin: config.corsOrigin,
      credentials: true,
    },
    observability: {
      tracing: {
        enabled: config.tracing.enabled,
        serviceName: config.tracing.serviceName,
        exporter: 'otlp',
        endpoint: config.tracing.endpoint,
      },
      logging: {
        enabled: true,
        level: 'info',
        format: 'json',
      },
    },
  });

  // Global middleware
  app.use(logger());
  app.use(rateLimit({
    windowMs: 60000,
    maxRequests: 100,
  }));

  // Health check
  app.get('/health', async (ctx) => {
    ctx.json({ status: 'ok' });
  });

  // Register routes
  app.use('/api', routes);

  // Start server
  await app.listen(config.port, config.host);
  console.log(`Production server running on ${config.host}:${config.port}`);

  // Graceful shutdown
  const shutdown = async () => {
    await app.close();
    await db.end();
    await redis.quit();
    process.exit(0);
  };

  process.on('SIGTERM', shutdown);
  process.on('SIGINT', shutdown);
}

main().catch((error) => {
  console.error('Failed to start server:', error);
  process.exit(1);
});
```

---

## See Also

- [Docker Deployment](docker.md)
- [Cloud Deployment](cloud-deployment.md)
- [Production Observability](production-observability.md)
