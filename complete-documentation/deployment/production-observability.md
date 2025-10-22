# Production Observability

Complete guide for setting up observability in production: distributed tracing, metrics, logging, and alerting.

> **Note**: This guide contains documentation examples for production observability. The observability infrastructure configurations (Jaeger, Prometheus, ELK, Grafana) are production-ready. RamAPI's built-in observability configuration has been verified against the source code. Some advanced features like custom span methods (`ctx.startSpan()`, `ctx.endSpan()`) are conceptual and may require implementation.

## Table of Contents

1. [Overview](#overview)
2. [Distributed Tracing](#distributed-tracing)
3. [Metrics Collection](#metrics-collection)
4. [Log Aggregation](#log-aggregation)
5. [Alerting](#alerting)
6. [Dashboards](#dashboards)
7. [APM Integration](#apm-integration)
8. [Complete Stack Example](#complete-stack-example)

---

## Overview

### Observability Pillars

1. **Traces**: Request flow across services
2. **Metrics**: System performance indicators
3. **Logs**: Detailed event records

### Architecture

```
┌─────────────┐
│  RamAPI App │
└──────┬──────┘
       │
   ┌───┴────┬─────────┬──────────┐
   │        │         │          │
┌──▼───┐ ┌──▼────┐ ┌──▼─────┐ ┌──▼──────┐
│Traces│ │Metrics│ │  Logs  │ │ Events  │
└──┬───┘ └───┬───┘ └───┬────┘ └────┬────┘
   │         │         │           │
┌──▼─────────▼─────────▼───────────▼────┐
│       Observability Backend            │
│  (Jaeger, Prometheus, Elasticsearch)   │
└────────────────┬───────────────────────┘
                 │
         ┌───────▼────────┐
         │    Grafana     │
         │   (Dashboards) │
         └────────────────┘
```

---

## Distributed Tracing

### Jaeger Setup

**Best for**: Distributed tracing, service dependencies

#### 1. Deploy Jaeger

**Docker Compose:**

```yaml
# docker-compose-observability.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "5775:5775/udp"     # Agent (zipkin.thrift, deprecated)
      - "6831:6831/udp"     # Agent (jaeger.thrift, compact)
      - "6832:6832/udp"     # Agent (jaeger.thrift, binary)
      - "5778:5778"         # Agent (config, sampling)
      - "16686:16686"       # UI
      - "14268:14268"       # Collector (jaeger.thrift)
      - "14250:14250"       # Collector (gRPC)
      - "4317:4317"         # OTLP gRPC
      - "4318:4318"         # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    restart: unless-stopped

  elasticsearch:
    image: elasticsearch:8.10.2
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    restart: unless-stopped

volumes:
  es_data:
```

#### 2. Configure RamAPI

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'otlp',
      endpoint: 'http://jaeger:4318', // OTLP HTTP endpoint
      sampleRate: 1.0, // 100% sampling (reduce in production)
      attributes: {
        'deployment.environment': process.env.NODE_ENV || 'development',
        'service.version': process.env.VERSION || '1.0.0',
      },
    },
  },
});
```

#### 3. Custom Spans

> **Note**: The `ctx.startSpan()` and `ctx.endSpan()` methods shown below are conceptual. For custom spans, you may need to use the OpenTelemetry API directly with `trace.getTracer()` and create spans manually until these convenience methods are implemented.

```typescript
app.get('/api/users/:id', async (ctx) => {
  // Start custom span
  const dbSpan = ctx.startSpan?.('db.query.user', {
    'db.system': 'postgresql',
    'db.operation': 'SELECT',
    'user.id': ctx.params.id,
  });

  try {
    const user = await db.query('SELECT * FROM users WHERE id = $1', [ctx.params.id]);

    // End span successfully
    ctx.endSpan?.(dbSpan);

    // Start cache span
    const cacheSpan = ctx.startSpan?.('cache.set', {
      'cache.key': `user:${ctx.params.id}`,
    });

    await redis.set(`user:${ctx.params.id}`, JSON.stringify(user), 'EX', 3600);
    ctx.endSpan?.(cacheSpan);

    ctx.json({ user });
  } catch (error) {
    // End span with error
    ctx.endSpan?.(dbSpan, error as Error);
    throw error;
  }
});
```

#### 4. Trace Context Propagation

```typescript
// Client-side: propagate trace context
import { trace, context } from '@opentelemetry/api';

async function callExternalService(url: string) {
  const span = trace.getSpan(context.active());

  const headers: Record<string, string> = {};

  // Inject trace context into headers
  propagation.inject(context.active(), headers);

  const response = await fetch(url, {
    headers: {
      ...headers,
      'Content-Type': 'application/json',
    },
  });

  return response.json();
}
```

#### 5. Access Jaeger UI

```bash
# Start services
docker-compose -f docker-compose-observability.yml up -d

# Open Jaeger UI
open http://localhost:16686
```

**Jaeger UI Features:**
- Search traces by service, operation, tags
- View trace timeline (waterfall)
- Analyze service dependencies
- Compare traces
- Find performance bottlenecks

---

### Zipkin Setup

**Alternative to Jaeger**

```yaml
# docker-compose.yml
zipkin:
  image: openzipkin/zipkin:latest
  ports:
    - "9411:9411"
  restart: unless-stopped
```

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'my-api',
      exporter: 'zipkin',
      endpoint: 'http://zipkin:9411/api/v2/spans',
    },
  },
});
```

---

## Metrics Collection

### Prometheus Setup

**Best for**: Time-series metrics, alerting

#### 1. Deploy Prometheus

**prometheus.yml:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # RamAPI application
  - job_name: 'ramapi'
    static_configs:
      - targets: ['api:3000']
    metrics_path: '/metrics'
    scrape_interval: 10s

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # PostgreSQL metrics
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  # Redis metrics
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

# Alerting configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - '/etc/prometheus/alerts.yml'
```

**docker-compose.yml:**

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    restart: unless-stopped

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      - DATA_SOURCE_NAME=postgresql://user:password@postgres:5432/mydb?sslmode=disable
    ports:
      - "9187:9187"
    restart: unless-stopped

  redis-exporter:
    image: oliver006/redis_exporter:latest
    environment:
      - REDIS_ADDR=redis://redis:6379
    ports:
      - "9121:9121"
    restart: unless-stopped

volumes:
  prometheus_data:
```

#### 2. Configure RamAPI Metrics

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  observability: {
    metrics: {
      enabled: true,
      // Note: RamAPI's MetricsConfig does not include an 'endpoint' field for serving metrics
      // You may need to add a custom route to expose metrics at /metrics
    },
  },
});

// Add custom metrics endpoint if needed
// app.get('/metrics', async (ctx) => {
//   const metrics = await getMetrics(); // Implement based on your metrics collection
//   ctx.text(metrics);
// });
```

#### 3. Custom Metrics

```typescript
import { Counter, Histogram, Gauge } from 'prom-client';

// Counter: monotonically increasing (requests, errors)
const requestCounter = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

// Histogram: measure duration (request latency)
const requestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5], // seconds
});

// Gauge: arbitrary value that can go up/down (active connections)
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
});

// Middleware to track metrics
app.use(async (ctx, next) => {
  const start = Date.now();
  activeConnections.inc();

  try {
    await next();

    const duration = (Date.now() - start) / 1000;
    requestCounter.labels(ctx.method, ctx.path, String(ctx.status)).inc();
    requestDuration.labels(ctx.method, ctx.path, String(ctx.status)).observe(duration);
  } catch (error) {
    requestCounter.labels(ctx.method, ctx.path, '500').inc();
    throw error;
  } finally {
    activeConnections.dec();
  }
});
```

#### 4. Business Metrics

```typescript
// Order metrics
const orderCounter = new Counter({
  name: 'orders_total',
  help: 'Total orders created',
  labelNames: ['status'],
});

const orderValue = new Histogram({
  name: 'order_value_dollars',
  help: 'Order value in dollars',
  buckets: [10, 50, 100, 500, 1000, 5000],
});

app.post('/orders', async (ctx) => {
  const order = await createOrder(ctx.body);

  // Track metrics
  orderCounter.labels(order.status).inc();
  orderValue.observe(order.total);

  ctx.json({ order }, 201);
});
```

#### 5. Access Prometheus

```bash
# Open Prometheus UI
open http://localhost:9090

# Query examples in UI:
# - rate(http_requests_total[5m])
# - http_request_duration_seconds{quantile="0.95"}
# - active_connections
```

---

### StatsD/Graphite (Alternative)

```typescript
import StatsD from 'node-statsd';

const statsd = new StatsD({
  host: 'statsd',
  port: 8125,
  prefix: 'ramapi.',
});

app.use(async (ctx, next) => {
  const start = Date.now();

  await next();

  const duration = Date.now() - start;
  statsd.timing(`request.${ctx.method}.${ctx.path}`, duration);
  statsd.increment(`status.${ctx.status}`);
});
```

---

## Log Aggregation

### ELK Stack (Elasticsearch, Logstash, Kibana)

#### 1. Deploy ELK Stack

**docker-compose.yml:**

```yaml
services:
  elasticsearch:
    image: elasticsearch:8.10.2
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    restart: unless-stopped

  logstash:
    image: logstash:8.10.2
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    environment:
      - "LS_JAVA_OPTS=-Xms512m -Xmx512m"
    depends_on:
      - elasticsearch
    restart: unless-stopped

  kibana:
    image: kibana:8.10.2
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    restart: unless-stopped

volumes:
  es_data:
```

**logstash.conf:**

```conf
input {
  tcp {
    port => 5000
    codec => json
  }
  udp {
    port => 5000
    codec => json
  }
}

filter {
  # Parse timestamp
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Add geolocation
  if [clientIp] {
    geoip {
      source => "clientIp"
      target => "geoip"
    }
  }

  # Parse user agent
  if [userAgent] {
    useragent {
      source => "userAgent"
      target => "ua"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "ramapi-logs-%{+YYYY.MM.dd}"
  }

  # Debug output
  stdout { codec => rubydebug }
}
```

#### 2. Configure RamAPI Logging

```typescript
import { createApp } from 'ramapi';
import winston from 'winston';
import LogstashTransport from 'winston-logstash/lib/winston-logstash-latest';

// Winston logger with Logstash transport
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.Console({
      format: winston.format.simple(),
    }),
    new LogstashTransport({
      host: 'logstash',
      port: 5000,
      node_name: 'ramapi-app',
    }),
  ],
});

const app = createApp({
  observability: {
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
    },
  },
});

// Structured logging middleware
app.use(async (ctx, next) => {
  const start = Date.now();

  await next();

  const duration = Date.now() - start;

  logger.info('Request completed', {
    method: ctx.method,
    path: ctx.path,
    status: ctx.status,
    duration,
    traceId: ctx.trace?.traceId,
    spanId: ctx.trace?.spanId,
    userId: ctx.state.userId,
    clientIp: ctx.headers['x-forwarded-for'] || ctx.ip,
    userAgent: ctx.headers['user-agent'],
    timestamp: new Date().toISOString(),
  });
});
```

#### 3. Access Kibana

```bash
# Open Kibana
open http://localhost:5601

# Create index pattern: ramapi-logs-*
# Explore logs in Discover tab
```

---

### CloudWatch Logs (AWS)

```typescript
import winston from 'winston';
import WinstonCloudWatch from 'winston-cloudwatch';

const logger = winston.createLogger({
  transports: [
    new WinstonCloudWatch({
      logGroupName: '/ramapi/production',
      logStreamName: `instance-${process.env.INSTANCE_ID}`,
      awsRegion: 'us-east-1',
      jsonMessage: true,
    }),
  ],
});

app.use(async (ctx, next) => {
  await next();
  logger.info('Request', {
    method: ctx.method,
    path: ctx.path,
    status: ctx.status,
  });
});
```

---

### Google Cloud Logging

```typescript
import { Logging } from '@google-cloud/logging';

const logging = new Logging({
  projectId: 'your-project-id',
});
const log = logging.log('ramapi-logs');

app.use(async (ctx, next) => {
  await next();

  const metadata = {
    resource: { type: 'global' },
    severity: ctx.status >= 400 ? 'ERROR' : 'INFO',
  };

  const entry = log.entry(metadata, {
    method: ctx.method,
    path: ctx.path,
    status: ctx.status,
    traceId: ctx.trace?.traceId,
  });

  await log.write(entry);
});
```

---

## Alerting

### Prometheus Alertmanager

#### 1. Alert Rules

**alerts.yml:**

```yaml
groups:
  - name: ramapi_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
          /
          rate(http_requests_total[5m])
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.instance }}"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High request latency"
          description: "95th percentile latency is {{ $value }}s"

      # Service down
      - alert: ServiceDown
        expr: up{job="ramapi"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
          / node_memory_MemTotal_bytes
          > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # High CPU usage
      - alert: HighCPUUsage
        expr: |
          100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
          > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value }}%"

      # Database connection pool exhausted
      - alert: DatabasePoolExhausted
        expr: pg_stat_activity_count >= pg_settings_max_connections
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool exhausted"
          description: "All database connections are in use"

      # Slow queries
      - alert: SlowQueries
        expr: |
          rate(http_request_duration_seconds_sum{route="/api/users"}[5m])
          /
          rate(http_request_duration_seconds_count{route="/api/users"}[5m])
          > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow queries detected"
          description: "Average query time is {{ $value }}s"
```

#### 2. Alertmanager Configuration

**alertmanager.yml:**

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'password'

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

**docker-compose.yml:**

```yaml
services:
  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    restart: unless-stopped
```

---

### Application-Level Alerts

```typescript
import { Counter } from 'prom-client';
import axios from 'axios';

const errorCounter = new Counter({
  name: 'app_errors_total',
  help: 'Total application errors',
  labelNames: ['type', 'severity'],
});

// Custom alerting function
async function sendAlert(severity: 'info' | 'warning' | 'critical', message: string, details?: any) {
  // Log
  console.error(`[${severity.toUpperCase()}] ${message}`, details);

  // Increment metric
  errorCounter.labels('application', severity).inc();

  // Send to Slack
  if (severity === 'critical') {
    await axios.post(process.env.SLACK_WEBHOOK_URL!, {
      text: `🚨 *${severity.toUpperCase()}*: ${message}`,
      attachments: [
        {
          color: 'danger',
          fields: [
            {
              title: 'Details',
              value: JSON.stringify(details, null, 2),
            },
          ],
        },
      ],
    });
  }

  // Send to PagerDuty
  if (severity === 'critical') {
    await axios.post('https://events.pagerduty.com/v2/enqueue', {
      routing_key: process.env.PAGERDUTY_KEY,
      event_action: 'trigger',
      payload: {
        summary: message,
        severity,
        source: 'ramapi-app',
        custom_details: details,
      },
    });
  }
}

// Usage in error handler
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error: any) {
    if (error.statusCode >= 500) {
      await sendAlert('critical', 'Internal server error', {
        error: error.message,
        stack: error.stack,
        path: ctx.path,
        method: ctx.method,
        traceId: ctx.trace?.traceId,
      });
    }
    throw error;
  }
});
```

---

## Dashboards

### Grafana Setup

#### 1. Deploy Grafana

**docker-compose.yml:**

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
      - elasticsearch
    restart: unless-stopped

volumes:
  grafana_data:
```

#### 2. Data Sources

**grafana/provisioning/datasources/datasources.yml:**

```yaml
apiVersion: 1

datasources:
  # Prometheus
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  # Elasticsearch (logs)
  - name: Elasticsearch
    type: elasticsearch
    access: proxy
    url: http://elasticsearch:9200
    database: "ramapi-logs-*"
    jsonData:
      esVersion: "8.0.0"
      timeField: "@timestamp"
      logLevelField: "level"

  # Jaeger (traces)
  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger:16686
    jsonData:
      tracesToLogsV2:
        datasourceUid: 'elasticsearch'
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
```

#### 3. Dashboard JSON

**grafana/provisioning/dashboards/ramapi.json:**

```json
{
  "dashboard": {
    "title": "RamAPI - Production Overview",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{ method }} {{ route }}"
          }
        ]
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])",
            "legendFormat": "{{ status }}"
          }
        ]
      },
      {
        "id": 3,
        "title": "Request Duration (p95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{ route }}"
          }
        ]
      },
      {
        "id": 4,
        "title": "Active Connections",
        "type": "stat",
        "targets": [
          {
            "expr": "active_connections"
          }
        ]
      },
      {
        "id": 5,
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "process_resident_memory_bytes / 1024 / 1024",
            "legendFormat": "Memory (MB)"
          }
        ]
      },
      {
        "id": 6,
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(process_cpu_seconds_total[5m]) * 100",
            "legendFormat": "CPU %"
          }
        ]
      }
    ]
  }
}
```

#### 4. Access Grafana

```bash
# Open Grafana
open http://localhost:3001

# Login: admin / admin
# Navigate to Dashboards > RamAPI
```

---

## APM Integration

### Datadog

```typescript
import tracer from 'dd-trace';

// Initialize Datadog tracer
tracer.init({
  service: 'ramapi-app',
  env: process.env.NODE_ENV || 'development',
  version: process.env.VERSION || '1.0.0',
  logInjection: true,
  runtimeMetrics: true,
  profiling: true,
});

const app = createApp({
  observability: {
    tracing: { enabled: false }, // Use Datadog instead
  },
});

// Datadog automatically instruments HTTP, database, Redis, etc.
```

---

### New Relic

```typescript
// Must be first import
import newrelic from 'newrelic';

const app = createApp();

// New Relic auto-instruments
```

**newrelic.js:**

```javascript
exports.config = {
  app_name: ['RamAPI Production'],
  license_key: 'YOUR_LICENSE_KEY',
  logging: {
    level: 'info',
  },
  distributed_tracing: {
    enabled: true,
  },
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 'apdex_f',
  },
};
```

---

### Sentry (Error Tracking)

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Express({ app: app as any }),
  ],
});

// Request handler (must be first)
app.use((ctx, next) => {
  return Sentry.runWithAsyncContext(async () => {
    const hub = Sentry.getCurrentHub();
    hub.configureScope(scope => {
      scope.setUser({
        id: ctx.state.userId,
        ip_address: ctx.ip,
      });
      scope.setTag('path', ctx.path);
      scope.setTag('method', ctx.method);
    });
    await next();
  });
});

// Error handler (must be last)
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    Sentry.captureException(error);
    throw error;
  }
});
```

---

## Complete Stack Example

### Production-Ready Observability Stack

**docker-compose.observability.yml:**

```yaml
version: '3.8'

services:
  # Application
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - TRACING_ENDPOINT=http://jaeger:4318
      - METRICS_ENABLED=true
    depends_on:
      - jaeger
      - prometheus
    restart: unless-stopped

  # Distributed Tracing
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    restart: unless-stopped

  # Metrics Collection
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    restart: unless-stopped

  # Alerting
  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    restart: unless-stopped

  # Log Storage
  elasticsearch:
    image: elasticsearch:8.10.2
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    restart: unless-stopped

  # Log Processing
  logstash:
    image: logstash:8.10.2
    ports:
      - "5000:5000"
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    depends_on:
      - elasticsearch
    restart: unless-stopped

  # Log/Metrics UI
  kibana:
    image: kibana:8.10.2
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    restart: unless-stopped

  # Dashboards
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
      - elasticsearch
      - jaeger
    restart: unless-stopped

  # System Metrics
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus_data:
  es_data:
  grafana_data:
```

**Start complete stack:**

```bash
docker-compose -f docker-compose.observability.yml up -d
```

**Access UIs:**
- Jaeger (Traces): http://localhost:16686
- Prometheus (Metrics): http://localhost:9090
- Grafana (Dashboards): http://localhost:3001
- Kibana (Logs): http://localhost:5601
- Alertmanager: http://localhost:9093

---

## Best Practices

### 1. Sampling Strategy

```typescript
// Production: sample 10% of requests
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      sampleRate: 0.1, // 10%
    },
  },
});

// Always trace errors
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    // Force trace on error
    if (ctx.trace) {
      ctx.trace.sampleRate = 1.0;
    }
    throw error;
  }
});
```

### 2. Cardinality Control

```typescript
// BAD: High cardinality (user IDs in labels)
requestCounter.labels(ctx.method, ctx.path, ctx.state.userId).inc();

// GOOD: Low cardinality (aggregated labels)
requestCounter.labels(ctx.method, ctx.path.replace(/\/\d+/, '/:id')).inc();
```

### 3. Sensitive Data Filtering

```typescript
// Filter sensitive data from logs
function sanitize(data: any) {
  const sensitive = ['password', 'token', 'secret', 'apiKey'];
  const sanitized = { ...data };

  for (const key of sensitive) {
    if (key in sanitized) {
      sanitized[key] = '[REDACTED]';
    }
  }

  return sanitized;
}

app.use(async (ctx, next) => {
  await next();
  logger.info('Request', sanitize({
    method: ctx.method,
    path: ctx.path,
    body: ctx.body,
  }));
});
```

### 4. Cost Optimization

```typescript
// Reduce log volume in production
const logLevel = process.env.NODE_ENV === 'production' ? 'warn' : 'info';

// Compress logs
import { createGzip } from 'zlib';
import { createWriteStream } from 'fs';

const gzipStream = createGzip();
const outputStream = createWriteStream('logs.json.gz');
gzipStream.pipe(outputStream);
```

---

## See Also

- [Production Setup](production-setup.md)
- [Docker Deployment](docker.md)
- [Cloud Deployment](cloud-deployment.md)
- [Tracing Guide](../observability/tracing.md)
- [Metrics Guide](../observability/metrics.md)
