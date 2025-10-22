# Metrics & Logging

Collect metrics automatically and configure structured logging for production observability.

## Table of Contents

1. [Metrics Collection](#metrics-collection)
2. [Structured Logging](#structured-logging)
3. [Prometheus Export](#prometheus-export)
4. [Health Checks](#health-checks)

---

## Metrics Collection

### Automatic Metrics

RamAPI automatically collects:
- **Request rate**: Requests per second
- **Latency**: avg, p50, p95, p99
- **Error rate**: 5xx errors / total requests
- **By protocol**: REST, GraphQL, gRPC counts
- **By status code**: 200, 404, 500, etc.

### Configuration

```typescript
const app = createApp({
  observability: {
    metrics: {
      enabled: true,
      collectInterval: 60000, // 1 minute
      prefix: 'ramapi',
    },
  },
});
```

### Accessing Metrics

```typescript
import { getMetrics } from 'ramapi';

app.get('/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.json(metrics);
});
```

**Output:**
```json
{
  "totalRequests": 15234,
  "requestsPerSecond": 25.39,
  "averageLatency": 45.6,
  "p50Latency": 32.1,
  "p95Latency": 125.4,
  "p99Latency": 250.8,
  "errorRate": 0.012,
  "byProtocol": {
    "rest": 15000,
    "graphql": 200,
    "grpc": 34
  },
  "byStatusCode": {
    "200": 14500,
    "404": 500,
    "500": 234
  }
}
```

---

## Structured Logging

### Configuration

```typescript
const app = createApp({
  observability: {
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
      redactFields: ['password', 'ssn'],
      includeStackTrace: true,
    },
  },
});
```

### Log Levels

- **trace**: Very detailed debugging
- **debug**: Debugging information
- **info**: Informational messages
- **warn**: Warning messages
- **error**: Error messages
- **fatal**: Critical errors

### Log Format

```json
{
  "timestamp": "2025-01-15T10:30:45.123Z",
  "level": "info",
  "message": "Request processed",
  "traceId": "3f2504e04f8911edb13900505634b5f1",
  "spanId": "8a3c60f7d188f8fa",
  "protocol": "rest",
  "operationName": "GET /api/users",
  "duration": 45.6,
  "statusCode": 200
}
```

---

## Prometheus Export

### Enable Prometheus Endpoint

```typescript
import { exportPrometheusMetrics } from 'ramapi';

app.get('/metrics', async (ctx) => {
  ctx.setHeader('Content-Type', 'text/plain');
  ctx.text(exportPrometheusMetrics());
});
```

### Prometheus Output

```
# HELP ramapi_requests_total Total number of requests
# TYPE ramapi_requests_total counter
ramapi_requests_total 15234

# HELP ramapi_requests_per_second Requests per second
# TYPE ramapi_requests_per_second gauge
ramapi_requests_per_second 25.39

# HELP ramapi_latency_p95_ms P95 latency in milliseconds
# TYPE ramapi_latency_p95_ms gauge
ramapi_latency_p95_ms 125.40

# HELP ramapi_error_rate Error rate (5xx)
# TYPE ramapi_error_rate gauge
ramapi_error_rate 0.0120
```

---

## Health Checks

### Enable Health Endpoint

```typescript
const app = createApp({
  observability: {
    health: {
      enabled: true,
      path: '/health',
      includeMetrics: true,
    },
  },
});
```

### Health Response

```json
{
  "status": "healthy",
  "timestamp": "2025-01-15T10:30:45.123Z",
  "uptime": 3600000,
  "version": "1.0.0",
  "services": {
    "http": true,
    "grpc": false,
    "database": true
  },
  "metrics": {
    "totalRequests": 15234,
    "requestsPerSecond": 25.39,
    "errorRate": 0.012
  }
}
```

---

## Complete Example

```typescript
import { createApp, getMetrics, exportPrometheusMetrics } from 'ramapi';

const app = createApp({
  observability: {
    metrics: {
      enabled: true,
    },
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
    },
    health: {
      enabled: true,
      includeMetrics: true,
    },
  },
});

// JSON metrics
app.get('/api/metrics', async (ctx) => {
  ctx.json(getMetrics());
});

// Prometheus metrics
app.get('/metrics', async (ctx) => {
  ctx.setHeader('Content-Type', 'text/plain');
  ctx.text(exportPrometheusMetrics());
});

app.listen(3000);
```

---

## Next Steps

- [Monitoring Integration](monitoring-integration.md)
- [Distributed Tracing](tracing.md)
- [Performance Profiling](profiling.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md).
