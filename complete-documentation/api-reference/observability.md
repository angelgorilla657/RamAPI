# Observability API Reference

Complete API reference for RamAPI's observability features including tracing, logging, metrics, and profiling.

## Table of Contents

1. [Observability Configuration](#observability-configuration)
2. [Tracing](#tracing)
3. [Logging](#logging)
4. [Metrics](#metrics)
5. [Profiling](#profiling)
6. [Context Methods](#context-methods)
7. [Complete Examples](#complete-examples)

---

## Observability Configuration

### ObservabilityConfig

Main configuration interface for all observability features.

```typescript
interface ObservabilityConfig {
  enabled?: boolean;
  tracing?: TracingConfig;
  logging?: LoggingConfig;
  metrics?: MetricsConfig;
  health?: HealthConfig;
  metricsEndpoint?: MetricsEndpointConfig;
  profiling?: ProfilingConfig;
}
```

### Basic Setup

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
    },
    metrics: {
      enabled: true,
    },
    profiling: {
      enabled: true,
      slowThreshold: 100,
    },
  },
});
```

---

## Tracing

### TracingConfig

Configuration for distributed tracing with OpenTelemetry.

```typescript
interface TracingConfig {
  enabled: boolean;
  serviceName: string;
  serviceVersion?: string;
  exporter?: 'console' | 'otlp' | 'memory';
  endpoint?: string;
  sampleRate?: number;
  captureStackTraces?: boolean;
  maxSpanAttributes?: number;
  redactHeaders?: string[];
  captureRequestBody?: boolean;
  captureResponseBody?: boolean;
  spanNaming?: 'default' | 'http.route' | 'operation';
  defaultAttributes?: Record<string, any>;
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | `boolean` | - | Enable tracing |
| `serviceName` | `string` | - | Service name for traces |
| `serviceVersion` | `string` | `undefined` | Service version |
| `exporter` | `'console' \| 'otlp' \| 'memory'` | `'console'` | Trace exporter type |
| `endpoint` | `string` | `undefined` | OTLP endpoint URL |
| `sampleRate` | `number` | `1.0` | Sampling rate (0-1) |
| `captureStackTraces` | `boolean` | `false` | Capture stack traces on errors |
| `maxSpanAttributes` | `number` | `128` | Max attributes per span |
| `redactHeaders` | `string[]` | `[]` | Headers to redact |
| `captureRequestBody` | `boolean` | `false` | Capture request bodies |
| `captureResponseBody` | `boolean` | `false` | Capture response bodies |
| `spanNaming` | `string` | `'default'` | Span naming strategy |
| `defaultAttributes` | `object` | `{}` | Default span attributes |

### Example

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'user-api',
      serviceVersion: '1.0.0',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
      sampleRate: 1.0, // 100% sampling
      captureStackTraces: true,
      redactHeaders: ['authorization', 'cookie'],
      defaultAttributes: {
        'environment': 'production',
        'region': 'us-east-1',
      },
    },
  },
});
```

---

## Logging

### LoggingConfig

Configuration for structured logging.

```typescript
interface LoggingConfig {
  enabled: boolean;
  level: 'trace' | 'debug' | 'info' | 'warn' | 'error' | 'fatal';
  format: 'json' | 'pretty';
  redactFields?: string[];
  includeStackTrace?: boolean;
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | `boolean` | - | Enable logging |
| `level` | `string` | `'info'` | Minimum log level |
| `format` | `'json' \| 'pretty'` | `'json'` | Log format |
| `redactFields` | `string[]` | `[]` | Fields to redact (PII) |
| `includeStackTrace` | `boolean` | `true` | Include stack traces in error logs |

### Log Levels

```typescript
// trace < debug < info < warn < error < fatal
```

### Example

```typescript
const app = createApp({
  observability: {
    logging: {
      enabled: true,
      level: 'info', // Only log info and above
      format: 'json',
      redactFields: ['password', 'ssn', 'creditCard'],
      includeStackTrace: true,
    },
  },
});
```

---

## Metrics

### MetricsConfig

Configuration for metrics collection.

```typescript
interface MetricsConfig {
  enabled: boolean;
  collectInterval?: number;
  endpoint?: string;
  prefix?: string;
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | `boolean` | - | Enable metrics |
| `collectInterval` | `number` | `60000` | Collection interval (ms) |
| `endpoint` | `string` | `undefined` | Metrics push endpoint |
| `prefix` | `string` | `'ramapi'` | Metric name prefix |

### Example

```typescript
const app = createApp({
  observability: {
    metrics: {
      enabled: true,
      collectInterval: 60000, // 1 minute
      prefix: 'myapi',
    },
  },
});
```

### getMetrics()

Get current metrics snapshot.

```typescript
import { getMetrics } from 'ramapi';

const metrics = getMetrics();
console.log(metrics);
```

**Returns:**

```typescript
interface RequestMetrics {
  totalRequests: number;
  requestsPerSecond: number;
  averageLatency: number;
  p50Latency: number;
  p95Latency: number;
  p99Latency: number;
  errorRate: number;
  byProtocol: {
    rest: number;
    graphql: number;
    grpc: number;
  };
  byStatusCode: Record<number, number>;
}
```

---

## Profiling

### ProfilingConfig

Configuration for performance profiling.

```typescript
interface ProfilingConfig {
  enabled: boolean;
  captureMemory?: boolean;
  bufferSize?: number;
  slowThreshold?: number;
  enableBudgets?: boolean;
  autoDetectBottlenecks?: boolean;
  captureStacks?: boolean;
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | `boolean` | - | Enable profiling |
| `captureMemory` | `boolean` | `false` | Enable memory profiling |
| `bufferSize` | `number` | `100` | Profiles to keep in memory |
| `slowThreshold` | `number` | `1000` | Slow request threshold (ms) |
| `enableBudgets` | `boolean` | `false` | Enable performance budgets |
| `autoDetectBottlenecks` | `boolean` | `false` | Auto-detect bottlenecks |
| `captureStacks` | `boolean` | `false` | Capture stack traces |

### Example

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 100, // Flag requests >100ms
      autoDetectBottlenecks: true,
      captureStacks: true,
    },
  },
});
```

### getProfiles()

Get profiling data.

```typescript
import { getProfiles } from 'ramapi';

const profiles = await getProfiles({ slowOnly: true });
console.log(profiles);
```

---

## Context Methods

### startSpan()

Start a new span for tracing operations.

```typescript
ctx.startSpan(name: string, attributes?: Record<string, any>): Span | undefined
```

**Example:**

```typescript
app.get('/users', async (ctx) => {
  const dbSpan = ctx.startSpan('database.query', {
    'db.system': 'postgresql',
    'db.operation': 'SELECT',
    'db.table': 'users',
  });

  try {
    const users = await db.query('SELECT * FROM users');
    ctx.endSpan(dbSpan);
    ctx.json({ users });
  } catch (error) {
    ctx.endSpan(dbSpan, error);
    throw error;
  }
});
```

### endSpan()

End a span (with optional error).

```typescript
ctx.endSpan(span: Span | undefined, error?: Error): void
```

**Example:**

```typescript
const span = ctx.startSpan('operation');

try {
  await performOperation();
  ctx.endSpan(span);
} catch (error) {
  ctx.endSpan(span, error); // Records error
  throw error;
}
```

### addEvent()

Add an event to the current span.

```typescript
ctx.addEvent(name: string, attributes?: Record<string, any>): void
```

**Example:**

```typescript
app.post('/users', async (ctx) => {
  ctx.addEvent('validation.started');

  // Validate
  ctx.addEvent('validation.completed', { valid: true });

  const user = await createUser(ctx.body);

  ctx.addEvent('user.created', { userId: user.id });

  ctx.json({ user }, 201);
});
```

### setAttributes()

Set attributes on the current span.

```typescript
ctx.setAttributes(attributes: Record<string, any>): void
```

**Example:**

```typescript
app.get('/users/:id', async (ctx) => {
  ctx.setAttributes({
    'user.id': ctx.params.id,
    'user.type': 'premium',
  });

  const user = await getUser(ctx.params.id);
  ctx.json({ user });
});
```

---

## Complete Examples

### Full Observability Setup

```typescript
import { createApp, getMetrics, getProfiles } from 'ramapi';

const app = createApp({
  observability: {
    // Tracing
    tracing: {
      enabled: true,
      serviceName: 'user-api',
      serviceVersion: '1.0.0',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
      sampleRate: 1.0,
      defaultAttributes: {
        'environment': process.env.NODE_ENV,
        'region': process.env.AWS_REGION,
      },
    },

    // Logging
    logging: {
      enabled: true,
      level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
      format: process.env.NODE_ENV === 'production' ? 'json' : 'pretty',
      redactFields: ['password', 'token', 'ssn'],
    },

    // Metrics
    metrics: {
      enabled: true,
      collectInterval: 60000,
      prefix: 'userapi',
    },

    // Profiling
    profiling: {
      enabled: true,
      slowThreshold: 100,
      autoDetectBottlenecks: true,
    },
  },
});

// Metrics endpoint
app.get('/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.json({ metrics });
});

// Profiling endpoint (development only)
if (process.env.NODE_ENV !== 'production') {
  app.get('/debug/profiles', async (ctx) => {
    const profiles = await getProfiles({ slowOnly: true });
    ctx.json({ profiles });
  });
}

await app.listen(3000);
```

### Custom Tracing

```typescript
app.get('/complex-operation', async (ctx) => {
  // Main operation span
  const mainSpan = ctx.startSpan('complex.operation');

  try {
    // Step 1: Validate
    const validateSpan = ctx.startSpan('validate.input');
    await validateInput(ctx.body);
    ctx.endSpan(validateSpan);

    // Step 2: Database
    const dbSpan = ctx.startSpan('database.query', {
      'db.system': 'postgresql',
      'db.operation': 'SELECT',
    });
    const data = await db.query('SELECT ...');
    ctx.endSpan(dbSpan);

    // Step 3: External API
    const apiSpan = ctx.startSpan('external.api.call', {
      'http.url': 'https://api.example.com',
      'http.method': 'POST',
    });
    const result = await fetch('https://api.example.com/data');
    ctx.endSpan(apiSpan);

    // Step 4: Process
    const processSpan = ctx.startSpan('process.data');
    const processed = await processData(data, result);
    ctx.endSpan(processSpan);

    ctx.endSpan(mainSpan);
    ctx.json({ result: processed });
  } catch (error) {
    ctx.endSpan(mainSpan, error);
    throw error;
  }
});
```

### Performance Monitoring

```typescript
// Alert on high latency
setInterval(() => {
  const metrics = getMetrics();

  if (metrics.p95Latency > 100) {
    console.error('⚠️  High latency:', metrics.p95Latency, 'ms');
  }

  if (metrics.errorRate > 0.01) {
    console.error('⚠️  High error rate:', (metrics.errorRate * 100).toFixed(2), '%');
  }

  if (metrics.requestsPerSecond > 10000) {
    console.warn('🔥 High traffic:', metrics.requestsPerSecond, 'req/s');
  }
}, 60000);
```

---

## See Also

- [Tracing Guide](../observability/tracing.md) - Distributed tracing guide
- [Profiling Guide](../observability/profiling.md) - Performance profiling guide
- [Metrics Guide](../observability/metrics.md) - Metrics collection guide

---

**Need help?** Check the [Observability Guide](../observability/overview.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
