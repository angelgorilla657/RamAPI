# Distributed Tracing

Distributed tracing tracks requests across your entire system, providing visibility into performance bottlenecks and dependencies. RamAPI uses OpenTelemetry for standards-based tracing.

## Table of Contents

1. [Overview](#overview)
2. [Configuration](#configuration)
3. [Exporters](#exporters)
4. [Trace Context](#trace-context)
5. [Custom Spans](#custom-spans)
6. [Best Practices](#best-practices)
7. [Integration Examples](#integration-examples)

---

## Overview

### What is Distributed Tracing?

Distributed tracing follows a request as it flows through multiple services:

```
Client → API Gateway → Auth Service → Database
  |          |              |            |
  5ms       10ms          15ms         50ms
```

Each step creates a **span**, and all spans together form a **trace**.

### Automatic Tracing

RamAPI automatically creates traces for all requests:

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
    },
  },
});

// Every request is automatically traced
app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});
```

---

## Configuration

### TracingConfig Interface

```typescript
interface TracingConfig {
  enabled: boolean;
  serviceName: string;
  serviceVersion?: string;
  exporter?: 'console' | 'otlp' | 'memory';
  endpoint?: string;
  sampleRate?: number;

  // Advanced
  captureStackTraces?: boolean;
  maxSpanAttributes?: number;
  redactHeaders?: string[];
  captureRequestBody?: boolean;
  captureResponseBody?: boolean;
  spanNaming?: 'default' | 'http.route' | 'operation';
  defaultAttributes?: Record<string, any>;
}
```

### Basic Configuration

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
```

### Production Configuration

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      serviceVersion: '1.0.0',
      exporter: 'otlp',
      endpoint: 'http://jaeger:4318/v1/traces',
      sampleRate: 0.1, // Sample 10%

      // Security
      redactHeaders: ['authorization', 'cookie', 'x-api-key'],
      captureRequestBody: false,
      captureResponseBody: false,

      // Performance
      maxSpanAttributes: 128,
      captureStackTraces: true,

      // Metadata
      defaultAttributes: {
        'deployment.environment': 'production',
        'service.namespace': 'api',
      },
    },
  },
});
```

---

## Exporters

### Console Exporter (Development)

Logs traces to console:

```typescript
tracing: {
  enabled: true,
  serviceName: 'my-api',
  exporter: 'console',
}
```

**Output:**
```json
{
  "traceId": "3f2504e04f8911edb13900505634b5f1",
  "spanId": "8a3c60f7d188f8fa",
  "name": "GET /api/users",
  "kind": "SERVER",
  "startTime": "2025-01-15T10:30:45.123Z",
  "endTime": "2025-01-15T10:30:45.178Z",
  "duration": 55,
  "attributes": {
    "http.method": "GET",
    "http.url": "/api/users",
    "http.status_code": 200
  }
}
```

### OTLP Exporter (Production)

Universal exporter compatible with Jaeger, Tempo, etc.:

```typescript
tracing: {
  enabled: true,
  serviceName: 'my-api',
  exporter: 'otlp',
  endpoint: 'http://jaeger:4318/v1/traces',
}
```

### Jaeger Setup

```bash
# Run Jaeger with Docker
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318/v1/traces',
    },
  },
});
```

Access Jaeger UI: http://localhost:16686

### Zipkin Setup

```bash
# Run Zipkin with Docker
docker run -d -p 9411:9411 openzipkin/zipkin
```

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://localhost:9411/api/v2/spans',
    },
  },
});
```

---

## Trace Context

### Accessing Trace Context

```typescript
app.get('/api/users', async (ctx) => {
  // Access trace context
  const { traceId, spanId } = ctx.trace;

  console.log('Trace ID:', traceId);
  console.log('Span ID:', spanId);

  ctx.json({ users: [], traceId });
});
```

### Response Headers

RamAPI automatically adds trace headers to responses:

```http
HTTP/1.1 200 OK
X-Trace-Id: 3f2504e04f8911edb13900505634b5f1
X-Span-Id: 8a3c60f7d188f8fa
```

### W3C Trace Context

RamAPI uses W3C Trace Context for propagation:

```http
GET /api/users HTTP/1.1
traceparent: 00-3f2504e04f8911edb13900505634b5f1-8a3c60f7d188f8fa-01
```

---

## Custom Spans

### Creating Child Spans

```typescript
import { startSpan, endSpan } from 'ramapi';

app.get('/api/users', async (ctx) => {
  // Create child span for database query
  const dbSpan = startSpan('database.query', {
    'db.system': 'postgresql',
    'db.statement': 'SELECT * FROM users',
  });

  try {
    const users = await db.query('SELECT * FROM users');

    dbSpan.setAttribute('db.rows_returned', users.length);
    ctx.json({ users });
  } catch (error) {
    dbSpan.recordException(error as Error);
    throw error;
  } finally {
    endSpan(dbSpan);
  }
});
```

### Using Context Helper Methods

```typescript
app.get('/api/users', async (ctx) => {
  // Start span using context helper
  const span = ctx.startSpan?.('fetch-users', {
    'user.id': ctx.user?.id,
  });

  const users = await db.query('SELECT * FROM users');

  // Add event
  ctx.addEvent?.('users-fetched', {
    count: users.length,
  });

  // End span
  ctx.endSpan?.(span);

  ctx.json({ users });
});
```

### Nested Operations

```typescript
app.get('/api/dashboard', async (ctx) => {
  const dashboardSpan = startSpan('build-dashboard');

  // Fetch users
  const usersSpan = startSpan('fetch-users');
  const users = await db.users.findAll();
  endSpan(usersSpan);

  // Fetch posts
  const postsSpan = startSpan('fetch-posts');
  const posts = await db.posts.findAll();
  endSpan(postsSpan);

  // Fetch analytics
  const analyticsSpan = startSpan('fetch-analytics');
  const analytics = await analyticsService.getData();
  endSpan(analyticsSpan);

  endSpan(dashboardSpan);

  ctx.json({ users, posts, analytics });
});
```

---

## Best Practices

### 1. Sample in Production

```typescript
tracing: {
  enabled: true,
  serviceName: 'my-api',
  exporter: 'otlp',
  sampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
}
```

### 2. Redact Sensitive Data

```typescript
tracing: {
  enabled: true,
  serviceName: 'my-api',
  redactHeaders: ['authorization', 'cookie', 'x-api-key', 'x-session-token'],
  captureRequestBody: false, // Don't capture bodies
}
```

### 3. Add Meaningful Attributes

```typescript
const span = startSpan('process-payment', {
  'payment.method': 'credit_card',
  'payment.amount': amount,
  'payment.currency': 'USD',
  'user.id': userId,
});
```

### 4. Use Semantic Conventions

Follow OpenTelemetry semantic conventions:

```typescript
// Good - Standard attributes
const span = startSpan('http.request', {
  'http.method': 'POST',
  'http.url': '/api/users',
  'http.status_code': 201,
});

// Avoid - Custom attributes for standard things
const span = startSpan('request', {
  'method': 'POST',
  'path': '/api/users',
  'code': 201,
});
```

### 5. Handle Errors Properly

```typescript
const span = startSpan('risky-operation');

try {
  await riskyOperation();
  span.setStatus({ code: SpanStatusCode.OK });
} catch (error) {
  span.recordException(error as Error);
  span.setStatus({
    code: SpanStatusCode.ERROR,
    message: (error as Error).message,
  });
  throw error;
} finally {
  endSpan(span);
}
```

### 6. Avoid Span Explosion

```typescript
// BAD - Creating too many spans
for (let i = 0; i < 10000; i++) {
  const span = startSpan(`iteration-${i}`);
  // ...
  endSpan(span);
}

// GOOD - Single span with count
const span = startSpan('process-items');
span.setAttribute('items.count', 10000);
// ... process items
endSpan(span);
```

---

## Integration Examples

### Microservices Communication

```typescript
// Service A
app.get('/api/data', async (ctx) => {
  // Make request to Service B with trace propagation
  const response = await fetch('http://service-b/api/external', {
    headers: {
      // Trace context automatically propagated
      'traceparent': ctx.req.headers['traceparent'],
    },
  });

  const data = await response.json();
  ctx.json({ data });
});
```

### Database Queries

```typescript
import { startSpan, endSpan } from 'ramapi';

app.get('/api/users/:id', async (ctx) => {
  const span = startSpan('db.query.user', {
    'db.system': 'postgresql',
    'db.name': 'mydb',
    'db.operation': 'SELECT',
  });

  try {
    const user = await db.users.findById(ctx.params.id);

    span.setAttributes({
      'db.rows_returned': user ? 1 : 0,
      'user.found': !!user,
    });

    ctx.json({ user });
  } finally {
    endSpan(span);
  }
});
```

### External API Calls

```typescript
app.get('/api/weather', async (ctx) => {
  const span = startSpan('http.client.openweather', {
    'http.method': 'GET',
    'http.url': 'https://api.openweathermap.org/data/2.5/weather',
    'peer.service': 'openweather',
  });

  try {
    const response = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?q=London`
    );

    span.setAttributes({
      'http.status_code': response.status,
      'http.response_content_length': response.headers.get('content-length'),
    });

    const data = await response.json();
    ctx.json(data);
  } finally {
    endSpan(span);
  }
});
```

### Complete Example

```typescript
import { createApp, startSpan, endSpan } from 'ramapi';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://jaeger:4318/v1/traces',
      sampleRate: 0.1,
    },
  },
});

app.get('/api/dashboard', async (ctx) => {
  // Parent span for entire operation
  const dashboardSpan = startSpan('build.dashboard', {
    'user.id': ctx.user?.id,
  });

  try {
    // Fetch user data
    const userSpan = startSpan('db.query.user');
    const user = await db.users.findById(ctx.user.id);
    endSpan(userSpan);

    // Fetch posts
    const postsSpan = startSpan('db.query.posts');
    const posts = await db.posts.findByUserId(ctx.user.id);
    postsSpan.setAttribute('posts.count', posts.length);
    endSpan(postsSpan);

    // Fetch analytics from external service
    const analyticsSpan = startSpan('http.client.analytics');
    const analytics = await fetch('http://analytics-service/api/data');
    analyticsSpan.setAttribute('http.status_code', analytics.status);
    endSpan(analyticsSpan);

    ctx.json({ user, posts, analytics });
  } finally {
    endSpan(dashboardSpan);
  }
});

app.listen(3000);
```

---

## Next Steps

- [Flow Visualization](flow-visualization.md)
- [Performance Profiling](profiling.md)
- [Monitoring Integration](monitoring-integration.md)
- [Observability Overview](overview.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
