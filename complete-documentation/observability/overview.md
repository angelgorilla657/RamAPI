# Observability Overview

RamAPI provides comprehensive observability features built on OpenTelemetry, enabling you to monitor, trace, and optimize your API in production.

## Table of Contents

1. [What is Observability?](#what-is-observability)
2. [Why Observability Matters](#why-observability-matters)
3. [RamAPI Observability Features](#ramapi-observability-features)
4. [Quick Start](#quick-start)
5. [OpenTelemetry Integration](#opentelemetry-integration)
6. [The Three Pillars](#the-three-pillars)
7. [Configuration Overview](#configuration-overview)
8. [Use Cases](#use-cases)

---

## What is Observability?

Observability is the ability to understand the internal state of your system by examining its outputs. Unlike traditional monitoring that tells you **when** something breaks, observability helps you understand **why** it broke.

### Observability vs Monitoring

| Monitoring | Observability |
|------------|---------------|
| Predefined metrics | Explore any question |
| Known unknowns | Unknown unknowns |
| Dashboards and alerts | Interactive investigation |
| "Is it broken?" | "Why is it broken?" |

---

## Why Observability Matters

### Production Debugging

```typescript
// Traditional approach: Add console.logs and redeploy
console.log('User ID:', userId);
console.log('Request:', request);

// Observability approach: Query existing traces
// No redeployment needed - data is already there
```

### Performance Optimization

Identify bottlenecks without guessing:
- Which database queries are slow?
- Which external APIs add latency?
- Where is memory being consumed?

### User Experience

Understand real user impact:
- P95 and P99 latency for actual users
- Error rates by region or device
- Full request context for errors

### Cost Optimization

Find inefficiencies:
- Unnecessary database queries
- Redundant API calls
- Resource-intensive operations

---

## RamAPI Observability Features

### 1. Distributed Tracing

Track requests across your entire system:

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'jaeger',
    },
  },
});
```

**Features:**
- W3C Trace Context propagation
- Automatic span creation for all requests
- Support for REST, GraphQL, and gRPC
- Integration with Jaeger, Zipkin, and more

### 2. Performance Profiling

Identify bottlenecks automatically:

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 1000, // Alert on >1s requests
      autoDetectBottlenecks: true,
    },
  },
});
```

**Features:**
- Automatic bottleneck detection
- Timeline visualization
- Memory profiling
- Performance budgets

### 3. Metrics Collection

Track key metrics automatically:

```typescript
const app = createApp({
  observability: {
    metrics: {
      enabled: true,
    },
  },
});
```

**Metrics Collected:**
- Request rate (req/s)
- Latency (avg, p50, p95, p99)
- Error rate
- By protocol (REST, GraphQL, gRPC)
- By status code

### 4. Structured Logging

Context-aware logging:

```typescript
const app = createApp({
  observability: {
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
    },
  },
});
```

**Features:**
- Trace ID correlation
- Structured JSON output
- PII redaction
- Multiple log levels

---

## Quick Start

### Minimal Setup

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'console', // Start with console
    },
  },
});

app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});

app.listen(3000);
```

### View Traces

Traces will be logged to console:

```json
{
  "traceId": "3f2504e04f8911edb13900505634b5f1",
  "spanId": "8a3c60f7d188f8fa",
  "name": "GET /api/users",
  "attributes": {
    "http.method": "GET",
    "http.url": "/api/users",
    "http.status_code": 200
  }
}
```

### Production Setup with Jaeger

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://jaeger:4318/v1/traces',
      sampleRate: 0.1, // Sample 10% in production
    },
    metrics: {
      enabled: true,
    },
    profiling: {
      enabled: true,
      slowThreshold: 1000,
    },
  },
});
```

---

## OpenTelemetry Integration

RamAPI is built on [OpenTelemetry](https://opentelemetry.io/), the industry-standard observability framework.

### Benefits

1. **Vendor-Neutral**: Not locked into any specific vendor
2. **Standard Protocol**: Works with any OTLP-compatible backend
3. **Rich Ecosystem**: Extensive tooling and integrations
4. **Future-Proof**: Backed by CNCF

### Supported Exporters

- **Console**: Development/debugging
- **OTLP**: Universal exporter (Jaeger, Tempo, etc.)
- **Jaeger**: Direct Jaeger integration
- **Zipkin**: Direct Zipkin integration

### Trace Propagation

RamAPI automatically propagates trace context using W3C Trace Context headers:

```http
GET /api/users HTTP/1.1
traceparent: 00-3f2504e04f8911edb13900505634b5f1-8a3c60f7d188f8fa-01
tracestate: vendor1=value1,vendor2=value2
```

---

## The Three Pillars

### 1. Traces

Distributed traces show the complete journey of a request:

```
Request → API Gateway → Auth Service → Database → Response
  |           |              |            |
  20ms       5ms           50ms         100ms
```

**See:** [Distributed Tracing Guide](tracing.md)

### 2. Metrics

Aggregated statistics about your system:

```
Requests/sec: 1,234
P95 Latency: 125ms
Error Rate: 0.02%
```

**See:** [Metrics & Logging Guide](metrics-and-logging.md)

### 3. Logs

Structured, searchable log events:

```json
{
  "level": "error",
  "traceId": "3f2504e04f8911edb13900505634b5f1",
  "message": "Database connection failed",
  "error": "ETIMEDOUT"
}
```

**See:** [Metrics & Logging Guide](metrics-and-logging.md)

---

## Configuration Overview

### ObservabilityConfig Interface

```typescript
interface ObservabilityConfig {
  enabled?: boolean;        // Global toggle
  tracing?: TracingConfig;
  logging?: LoggingConfig;
  metrics?: MetricsConfig;
  profiling?: ProfilingConfig;
  health?: HealthConfig;
  metricsEndpoint?: MetricsEndpointConfig;
}
```

### Tracing Configuration

```typescript
interface TracingConfig {
  enabled: boolean;
  serviceName: string;
  serviceVersion?: string;
  exporter?: 'console' | 'otlp' | 'memory';
  endpoint?: string;
  sampleRate?: number;      // 0-1, default 1.0

  // Advanced options
  captureStackTraces?: boolean;
  maxSpanAttributes?: number;
  redactHeaders?: string[];
  captureRequestBody?: boolean;
  captureResponseBody?: boolean;
}
```

### Profiling Configuration

```typescript
interface ProfilingConfig {
  enabled: boolean;
  captureMemory?: boolean;
  bufferSize?: number;
  slowThreshold?: number;       // ms
  enableBudgets?: boolean;
  autoDetectBottlenecks?: boolean;
  captureStacks?: boolean;
}
```

### Metrics Configuration

```typescript
interface MetricsConfig {
  enabled: boolean;
  collectInterval?: number;     // ms
  endpoint?: string;
  prefix?: string;              // Default: 'ramapi'
}
```

**See:** [Configuration Guide](../core-concepts/configuration.md)

---

## Use Cases

### Use Case 1: Find Slow Endpoints

```typescript
// Enable profiling
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 500, // Alert on >500ms
    },
  },
});

// Access profiling data
app.get('/debug/profiles', async (ctx) => {
  const profiles = await getProfiles({ slowOnly: true });
  ctx.json({ profiles });
});
```

### Use Case 2: Distributed Tracing Across Services

```typescript
// Service A
const serviceA = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'service-a',
      exporter: 'otlp',
    },
  },
});

// Service B
const serviceB = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'service-b',
      exporter: 'otlp',
    },
  },
});

// Traces automatically propagate between services
```

### Use Case 3: Production Monitoring

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: process.env.OTLP_ENDPOINT,
      sampleRate: 0.1, // 10% sampling
      redactHeaders: ['authorization', 'cookie'],
    },
    metrics: {
      enabled: true,
      endpoint: process.env.METRICS_ENDPOINT,
    },
    profiling: {
      enabled: true,
      slowThreshold: 2000,
      autoDetectBottlenecks: true,
    },
  },
});

// Access metrics endpoint
app.get('/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.text(exportPrometheusMetrics());
});
```

### Use Case 4: Debug Production Issues

```typescript
// 1. Find slow traces in Jaeger
// Filter: service=my-api duration>5s

// 2. Examine trace details
// See which spans are slow

// 3. Check profiling data
GET /debug/profiles?traceId=3f2504e04f8911edb13900505634b5f1

// 4. View waterfall chart
GET /debug/flows/3f2504e04f8911edb13900505634b5f1
```

---

## Environment-Based Configuration

### Development

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'console', // Console for dev
      sampleRate: 1.0,     // Sample everything
    },
    profiling: {
      enabled: true,
      slowThreshold: 100,  // Alert on >100ms
    },
  },
});
```

### Production

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: process.env.OTLP_ENDPOINT,
      sampleRate: 0.1,           // 10% sampling
      redactHeaders: ['authorization', 'cookie', 'x-api-key'],
      captureRequestBody: false, // Don't capture bodies
    },
    profiling: {
      enabled: true,
      slowThreshold: 2000,       // Alert on >2s
      autoDetectBottlenecks: true,
    },
    metrics: {
      enabled: true,
    },
  },
});
```

---

## Performance Impact

Observability has minimal performance overhead:

| Feature | Overhead | Notes |
|---------|----------|-------|
| Tracing (sampled) | <1% | With 10% sampling |
| Tracing (100%) | ~2-3% | Full sampling |
| Metrics | <0.5% | In-memory aggregation |
| Profiling | ~1-2% | Enabled by default |
| Logging | Variable | Depends on log volume |

**Recommendation**: Use 10-20% sampling in production for optimal balance.

---

## Next Steps

- **[Distributed Tracing](tracing.md)** - Set up tracing with Jaeger/Zipkin
- **[Flow Visualization](flow-visualization.md)** - Visualize request flows
- **[Performance Profiling](profiling.md)** - Identify bottlenecks
- **[Metrics & Logging](metrics-and-logging.md)** - Configure metrics and logs
- **[Monitoring Integration](monitoring-integration.md)** - Integrate with monitoring tools

---

## Common Questions

### Do I need observability?

Yes, if you want to:
- Debug production issues efficiently
- Optimize performance based on data
- Understand user experience
- Monitor system health

### Is OpenTelemetry required?

RamAPI's observability features use OpenTelemetry, but you can:
- Disable observability entirely
- Use only metrics or logs without tracing
- Use console exporter for development

### What's the cost?

Observability is free in RamAPI. Backend costs depend on:
- Self-hosted (Jaeger): Free but requires infrastructure
- Managed (Datadog, New Relic): Varies by plan

### Can I use my existing monitoring?

Yes! RamAPI supports:
- Prometheus metrics export
- OTLP for any compatible backend
- Custom exporters

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
