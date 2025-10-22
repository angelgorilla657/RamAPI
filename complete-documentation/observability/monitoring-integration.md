# Monitoring Integration

Integrate RamAPI with popular monitoring platforms including Grafana, Prometheus, Datadog, New Relic, and more.

## Table of Contents

1. [Prometheus + Grafana](#prometheus--grafana)
2. [Jaeger](#jaeger)
3. [Datadog](#datadog)
4. [New Relic](#new-relic)
5. [Generic OTLP](#generic-otlp)

---

## Prometheus + Grafana

### Setup Prometheus

**prometheus.yml:**
```yaml
scrape_configs:
  - job_name: 'ramapi'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:3000']
    metrics_path: '/metrics'
```

**RamAPI Config:**
```typescript
import { createApp, exportPrometheusMetrics } from 'ramapi';

const app = createApp({
  observability: {
    metrics: {
      enabled: true,
    },
  },
});

app.get('/metrics', async (ctx) => {
  ctx.setHeader('Content-Type', 'text/plain');
  ctx.text(exportPrometheusMetrics());
});

app.listen(3000);
```

### Grafana Dashboard

Import the RamAPI Grafana dashboard or create custom panels:

**Query Examples:**
```promql
# Request rate
rate(ramapi_requests_total[5m])

# P95 Latency
ramapi_latency_p95_ms

# Error rate
rate(ramapi_requests_by_status{code=~"5.."}[5m]) /
rate(ramapi_requests_total[5m])
```

---

## Jaeger

### Docker Setup

```bash
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

### RamAPI Config

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318/v1/traces',
      sampleRate: 1.0,
    },
  },
});
```

Access Jaeger UI: http://localhost:16686

---

## Datadog

### Install Datadog Agent

```bash
DD_API_KEY=<YOUR_API_KEY> \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```

### RamAPI Config

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318/v1/traces',
    },
    metrics: {
      enabled: true,
    },
  },
});

// Datadog metrics endpoint
app.get('/metrics', async (ctx) => {
  ctx.text(exportPrometheusMetrics());
});
```

---

## New Relic

### Install New Relic Agent

```bash
npm install @newrelic/native-metrics
```

### RamAPI Config

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'https://otlp.nr-data.net:4318/v1/traces',
    },
  },
});
```

---

## Generic OTLP

### Any OTLP-Compatible Backend

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: process.env.OTLP_ENDPOINT || 'http://localhost:4318/v1/traces',
    },
  },
});
```

Compatible with:
- Jaeger
- Zipkin
- Grafana Tempo
- Honeycomb
- Lightstep
- Any OTLP receiver

---

## Complete Example

```typescript
import { createApp, exportPrometheusMetrics } from 'ramapi';

const app = createApp({
  observability: {
    // Distributed tracing
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: process.env.OTLP_ENDPOINT || 'http://jaeger:4318/v1/traces',
      sampleRate: parseFloat(process.env.TRACE_SAMPLE_RATE || '0.1'),
    },

    // Metrics
    metrics: {
      enabled: true,
    },

    // Profiling
    profiling: {
      enabled: true,
      slowThreshold: 1000,
      autoDetectBottlenecks: true,
    },

    // Health checks
    health: {
      enabled: true,
      includeMetrics: true,
    },
  },
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

- [Observability Overview](overview.md)
- [Distributed Tracing](tracing.md)
- [Performance Profiling](profiling.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md).
