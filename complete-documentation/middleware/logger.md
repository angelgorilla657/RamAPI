# Logger Middleware

The logger middleware provides request logging with response times, status codes, and error tracking. This guide covers basic usage, customization, and best practices.

## Table of Contents

1. [Basic Usage](#basic-usage)
2. [Log Format](#log-format)
3. [Custom Logging](#custom-logging)
4. [Production Logging](#production-logging)
5. [Integration with Observability](#integration-with-observability)
6. [Best Practices](#best-practices)

---

## Basic Usage

### Simple Logger

```typescript
import { createApp, logger } from 'ramapi';

const app = createApp();

// Add logger middleware
app.use(logger());

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello, World!' });
});

app.listen(3000);
```

### Output

```
[200] GET / - 5ms
[404] GET /not-found - 2ms
[500] POST /error - 15ms
[ERROR] POST /crash - 8ms Error: Something went wrong
```

---

## Log Format

### Default Format

The built-in logger outputs:
- **Status code** (color-coded)
- **HTTP method**
- **Request path**
- **Response time** (in milliseconds)

### Color Coding

- **Green** (32): Success (2xx)
- **Yellow** (33): Client error (4xx)
- **Red** (31): Server error (5xx)
- **Red ERROR**: Unhandled exceptions

### Example Logs

```typescript
app.use(logger());

app.get('/success', async (ctx) => {
  ctx.json({ status: 'ok' });
});
// Output: [200] GET /success - 3ms

app.get('/not-found', async (ctx) => {
  throw new HTTPError(404, 'Not found');
});
// Output: [404] GET /not-found - 2ms

app.get('/error', async (ctx) => {
  throw new Error('Internal error');
});
// Output: [ERROR] GET /error - 5ms Error: Internal error
```

---

## Custom Logging

### Custom Logger Function

Create a custom logger with your preferred format:

```typescript
import type { Middleware } from 'ramapi';

function customLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    const { method, path } = ctx;

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      console.log(`${new Date().toISOString()} ${status} ${method} ${path} ${duration}ms`);
    } catch (error) {
      const duration = Date.now() - start;
      console.error(`${new Date().toISOString()} ERROR ${method} ${path} ${duration}ms`, error);
      throw error;
    }
  };
}

app.use(customLogger());
```

### JSON Structured Logging

For production, use structured JSON logs:

```typescript
function jsonLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    const { method, path } = ctx;

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        level: 'info',
        method,
        path,
        status,
        duration,
        ip: ctx.req.socket.remoteAddress,
        userAgent: ctx.headers['user-agent'],
      }));
    } catch (error) {
      const duration = Date.now() - start;
      const err = error as Error;

      console.error(JSON.stringify({
        timestamp: new Date().toISOString(),
        level: 'error',
        method,
        path,
        duration,
        error: err.message,
        stack: err.stack,
      }));
      throw error;
    }
  };
}

app.use(jsonLogger());
```

### Output:

```json
{"timestamp":"2025-01-15T10:30:45.123Z","level":"info","method":"GET","path":"/api/users","status":200,"duration":15,"ip":"127.0.0.1","userAgent":"Mozilla/5.0..."}
{"timestamp":"2025-01-15T10:30:46.456Z","level":"error","method":"POST","path":"/api/users","duration":8,"error":"Validation failed","stack":"Error: Validation failed\n    at..."}
```

---

## Production Logging

### Winston Integration

Use Winston for advanced logging features:

```bash
npm install winston
```

```typescript
import winston from 'winston';
import type { Middleware } from 'ramapi';

const winstonLogger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// In development, also log to console
if (process.env.NODE_ENV !== 'production') {
  winstonLogger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}

function winstonMiddleware(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    const { method, path } = ctx;

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      winstonLogger.info({
        method,
        path,
        status,
        duration,
        ip: ctx.req.socket.remoteAddress,
      });
    } catch (error) {
      const duration = Date.now() - start;
      const err = error as Error;

      winstonLogger.error({
        method,
        path,
        duration,
        error: err.message,
        stack: err.stack,
      });
      throw error;
    }
  };
}

app.use(winstonMiddleware());
```

### Pino Integration

Pino is a fast, low-overhead logger:

```bash
npm install pino
```

```typescript
import pino from 'pino';
import type { Middleware } from 'ramapi';

const pinoLogger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV !== 'production' ? {
    target: 'pino-pretty',
  } : undefined,
});

function pinoMiddleware(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    const { method, path } = ctx;

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      pinoLogger.info({
        method,
        path,
        status,
        duration,
      });
    } catch (error) {
      const duration = Date.now() - start;
      const err = error as Error;

      pinoLogger.error({
        method,
        path,
        duration,
        err,
      });
      throw error;
    }
  };
}

app.use(pinoMiddleware());
```

---

## Integration with Observability

### Correlation IDs

Add correlation IDs to track requests across services:

```typescript
function correlationIdLogger(): Middleware {
  return async (ctx, next) => {
    const correlationId = ctx.headers['x-correlation-id'] as string ||
                         crypto.randomUUID();

    // Add to response headers
    ctx.setHeader('X-Correlation-ID', correlationId);

    const start = Date.now();
    const { method, path } = ctx;

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      console.log(JSON.stringify({
        correlationId,
        method,
        path,
        status,
        duration,
      }));
    } catch (error) {
      const duration = Date.now() - start;
      console.error(JSON.stringify({
        correlationId,
        method,
        path,
        duration,
        error: (error as Error).message,
      }));
      throw error;
    }
  };
}
```

### Integration with OpenTelemetry

RamAPI has built-in OpenTelemetry support:

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'console',
    },
  },
});

// Traces automatically include request details
app.get('/api/users', async (ctx) => {
  // Request is automatically traced
  ctx.json({ users: [] });
});
```

See [Observability Guide](../observability/overview.md) for more details.

---

## Best Practices

### 1. Log at the Right Level

```typescript
function smartLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      // Only log slow requests and errors
      if (duration > 1000 || status >= 400) {
        console.log(`[${status}] ${ctx.method} ${ctx.path} - ${duration}ms`);
      }
    } catch (error) {
      console.error(`ERROR ${ctx.method} ${ctx.path}`, error);
      throw error;
    }
  };
}
```

### 2. Redact Sensitive Data

```typescript
function secureLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    const { method, path } = ctx;

    // Redact sensitive headers
    const headers = { ...ctx.headers };
    if (headers.authorization) headers.authorization = '[REDACTED]';
    if (headers.cookie) headers.cookie = '[REDACTED]';

    try {
      await next();
      const duration = Date.now() - start;
      const status = ctx.res.statusCode;

      console.log(JSON.stringify({
        method,
        path,
        status,
        duration,
        headers, // Redacted headers
      }));
    } catch (error) {
      console.error('Request error:', { method, path, error: (error as Error).message });
      throw error;
    }
  };
}
```

### 3. Conditional Logging

```typescript
// Only log in development
if (process.env.NODE_ENV !== 'production') {
  app.use(logger());
}
```

### 4. Performance Monitoring

```typescript
function performanceLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    try {
      await next();
      const duration = Date.now() - start;

      // Alert on slow requests
      if (duration > 5000) {
        console.warn(`SLOW REQUEST: ${ctx.method} ${ctx.path} - ${duration}ms`);
      }
    } catch (error) {
      throw error;
    }
  };
}
```

### 5. Request/Response Logging

```typescript
function detailedLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    console.log('Request:', {
      method: ctx.method,
      path: ctx.path,
      query: ctx.query,
      body: ctx.body,
      headers: ctx.headers,
    });

    try {
      await next();
      const duration = Date.now() - start;

      console.log('Response:', {
        status: ctx.res.statusCode,
        duration,
      });
    } catch (error) {
      console.error('Error:', error);
      throw error;
    }
  };
}
```

**Warning:** Be careful logging request/response bodies in production - they may contain sensitive data!

---

## Complete Examples

### Development Setup

```typescript
import { createApp, logger } from 'ramapi';

const app = createApp();

// Simple colored logs for development
app.use(logger());

app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});

app.listen(3000);
```

### Production Setup

```typescript
import { createApp } from 'ramapi';
import pino from 'pino';

const pinoLogger = pino({
  level: 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
});

function productionLogger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();

    try {
      await next();
      const duration = Date.now() - start;

      pinoLogger.info({
        method: ctx.method,
        path: ctx.path,
        status: ctx.res.statusCode,
        duration,
      });
    } catch (error) {
      pinoLogger.error({
        method: ctx.method,
        path: ctx.path,
        error: (error as Error).message,
      });
      throw error;
    }
  };
}

const app = createApp();
app.use(productionLogger());

app.listen(3000);
```

---

## Next Steps

- [Rate Limiting Middleware](rate-limiting.md)
- [CORS Middleware](cors.md)
- [Custom Middleware Guide](custom-middleware.md)
- [Observability Guide](../observability/overview.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
