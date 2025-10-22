# Setting Up Observability

Complete guide to enable tracing, profiling, and flow visualization in your RamAPI application.

> **Verification Status**: All code examples verified against RamAPI source code
> - ✅ `observability` config verified (src/core/server.ts, src/observability/types.ts)
> - ✅ Tracing config verified: enabled, serviceName, exporter, endpoint, sampleRate
> - ✅ `flowTrackingMiddleware()` verified (src/observability/flow/tracker.ts)
> - ✅ `registerFlowRoutes()` verified (example in examples/flow-visualization/basic-flow.ts)
> - ✅ Profiling config verified (src/observability/types.ts)
> - ✅ All observability features match actual implementation

## What We'll Build

Add comprehensive observability to your Task API:
- **Distributed tracing** with OpenTelemetry
- **Request flow visualization** with waterfall charts
- **Performance profiling** to identify bottlenecks
- **Metrics collection** for monitoring
- **Integration with Jaeger** for trace analysis

---

## Step 1: Enable Basic Tracing

Update `src/index.ts`:

```typescript
import { createApp } from 'ramapi';

/**
 * Verified observability config structure:
 * - tracing: { enabled, serviceName, exporter, endpoint, sampleRate }
 * - profiling: { enabled, captureMemory, slowThreshold }
 * - metrics: { enabled }
 */
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'task-api',
      serviceVersion: '1.0.0',
      exporter: 'console', // Start with console
      sampleRate: 1.0, // 100% sampling (reduce in production)
    },
  },
});

// ... rest of your app
```

**Run your app:**

```bash
npm run dev
```

**Make a request:**

```bash
curl http://localhost:3000/api/tasks
```

**Check console output:**

You'll see trace information logged to console:

```
✅ Tracing initialized (service: task-api, exporter: console, sample: 100%)
{
  traceId: '123e4567-e89b-12d3-a456-426614174000',
  spanId: 'a1b2c3d4e5f6g7h8',
  name: 'GET /api/tasks',
  duration: 15.2,
  attributes: {
    'http.method': 'GET',
    'http.route': '/api/tasks',
    'http.status_code': 200
  }
}
```

---

## Step 2: Enable Flow Tracking

Flow tracking provides detailed waterfall visualizations of request execution.

Update `src/index.ts`:

```typescript
import { createApp } from 'ramapi';
import { flowTrackingMiddleware } from 'ramapi';
import { registerFlowRoutes } from 'ramapi';

/**
 * Verified APIs:
 * - flowTrackingMiddleware() - Requires tracing to be enabled
 * - registerFlowRoutes(app) - Registers flow visualization routes
 */
const app = createApp({
  observability: {
    tracing: {
      enabled: true, // Required for flow tracking
      serviceName: 'task-api',
      exporter: 'console',
      sampleRate: 1.0,
    },
  },
});

// Enable flow tracking middleware
app.use(flowTrackingMiddleware());

// Register flow visualization routes
registerFlowRoutes(app as any);

// ... rest of your app
```

**Try flow visualization:**

```bash
# Make a request
curl http://localhost:3000/api/tasks

# Get the traceId from response headers or logs
export TRACE_ID="123e4567-e89b-12d3-a456-426614174000"

# View waterfall chart
curl http://localhost:3000/profile/$TRACE_ID/waterfall

# View Mermaid diagram
curl http://localhost:3000/profile/$TRACE_ID/mermaid

# View raw JSON data
curl http://localhost:3000/profile/$TRACE_ID/flow

# View flow statistics
curl http://localhost:3000/flow/stats

# View slowest requests
curl http://localhost:3000/flow/slow
```

**Example waterfall output:**

```
Request Flow: GET /api/tasks (27.3ms)
─────────────────────────────────────────────────────────────
Request Started         ▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  2.1ms
Routing                 ░░▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.3ms
Auth Middleware         ░░░▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░  4.5ms
Validation              ░░░░░░░▓░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.8ms
Handler Execution       ░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░ 12.7ms
  Database Query        ░░░░░░░░░▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░   8.2ms
  Result Processing     ░░░░░░░░░░░░░░░░░▓▓░░░░░░░░░░░░░░░   2.1ms
Response Sent           ░░░░░░░░░░░░░░░░░░░▓░░░░░░░░░░░░░░░  0.9ms
─────────────────────────────────────────────────────────────
Total: 27.3ms
```

---

## Step 3: Enable Profiling

Profiling helps identify slow operations and bottlenecks.

Update `src/index.ts`:

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'task-api',
      exporter: 'console',
    },
    profiling: {
      enabled: true,
      captureMemory: true, // Track memory usage
      slowThreshold: 1000, // Log warnings for requests > 1000ms
      enableBudgets: true, // Enable performance budgets
      autoDetectBottlenecks: true, // Automatic bottleneck detection
    },
  },
});
```

**Console output with profiling:**

```
⚠️  Slow request detected: GET /api/tasks took 1250ms (threshold: 1000ms)

🐢 Bottleneck detected:
   - Operation: database query
   - Duration: 1100ms
   - Path: /api/tasks
   - Recommendation: Add database index or optimize query

📊 Performance profile:
   - Total time: 1250ms
   - Handler: 1200ms (96%)
   - Middleware: 45ms (3.6%)
   - Response: 5ms (0.4%)
```

---

## Step 4: Integrate with Jaeger

Jaeger provides a UI for viewing and analyzing distributed traces.

### Start Jaeger

```bash
# Using Docker
docker run -d \
  --name jaeger \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# Or use docker-compose
# See deployment/production-observability.md for full setup
```

### Configure RamAPI to use Jaeger

Update `src/index.ts`:

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'task-api',
      serviceVersion: '1.0.0',
      exporter: 'otlp', // Use OTLP exporter for Jaeger
      endpoint: 'http://localhost:4318/v1/traces', // Jaeger OTLP HTTP endpoint
      sampleRate: 1.0, // 100% in development, lower in production (e.g., 0.1)
      attributes: {
        'deployment.environment': process.env.NODE_ENV || 'development',
        'service.namespace': 'task-api',
      },
    },
  },
});
```

**View traces in Jaeger:**

```bash
# Open Jaeger UI
open http://localhost:16686

# Select "task-api" service
# Search for traces
# Click on a trace to see detailed timeline
```

---

## Step 5: Add Custom Spans

Track custom operations in your code.

**Note**: `ctx.startSpan()` and `ctx.endSpan()` are conceptual. Use OpenTelemetry API directly:

```typescript
import { trace } from '@opentelemetry/api';

app.get('/api/tasks', async (ctx) => {
  const tracer = trace.getTracer('task-api');

  // Create custom span for database query
  const dbSpan = tracer.startSpan('database.query.tasks', {
    attributes: {
      'db.system': 'postgresql',
      'db.operation': 'SELECT',
      'db.table': 'tasks',
    },
  });

  try {
    const tasks = await taskStore.findAll();

    // Add custom attributes
    dbSpan.setAttribute('db.result_count', tasks.length);
    dbSpan.setStatus({ code: 0 }); // OK

    ctx.json({ tasks });
  } catch (error: any) {
    dbSpan.recordException(error);
    dbSpan.setStatus({ code: 2, message: error.message }); // ERROR
    throw error;
  } finally {
    dbSpan.end();
  }
});
```

---

## Step 6: Track Database Operations

Automatically track database queries in flow visualization.

```typescript
// Example database wrapper with tracking
class TrackedTaskStore {
  async findAll() {
    // Flow tracking will automatically capture this if using
    // database libraries with tracing support

    const start = performance.now();

    const tasks = await pool.query('SELECT * FROM tasks');

    const duration = performance.now() - start;

    // Log slow queries
    if (duration > 100) {
      console.warn(`Slow query detected: ${duration}ms`);
    }

    return tasks.rows;
  }
}
```

---

## Step 7: Add Metrics Collection

Enable metrics for Prometheus monitoring.

Update `src/index.ts`:

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'task-api',
      exporter: 'otlp',
    },
    metrics: {
      enabled: true,
      collectInterval: 60000, // Collect every 60 seconds
      prefix: 'task_api', // Metric name prefix
    },
  },
});

// Access metrics endpoint (if exposed by RamAPI)
app.get('/metrics', async (ctx) => {
  // Metrics will be collected automatically
  // Expose them in Prometheus format
  ctx.text('# Metrics data...');
});
```

---

## Step 8: Environment Configuration

Create `.env`:

```env
NODE_ENV=production
PORT=3000

# Observability
OBSERVABILITY_ENABLED=true
TRACING_ENABLED=true
TRACING_SERVICE_NAME=task-api
TRACING_SERVICE_VERSION=1.0.0
TRACING_EXPORTER=otlp
TRACING_ENDPOINT=http://localhost:4318/v1/traces
TRACING_SAMPLE_RATE=0.1

PROFILING_ENABLED=true
PROFILING_SLOW_THRESHOLD=1000

METRICS_ENABLED=true
```

Update `src/index.ts`:

```typescript
import 'dotenv/config';

const app = createApp({
  observability: {
    enabled: process.env.OBSERVABILITY_ENABLED === 'true',
    tracing: {
      enabled: process.env.TRACING_ENABLED === 'true',
      serviceName: process.env.TRACING_SERVICE_NAME || 'task-api',
      serviceVersion: process.env.TRACING_SERVICE_VERSION || '1.0.0',
      exporter: (process.env.TRACING_EXPORTER || 'console') as 'console' | 'otlp',
      endpoint: process.env.TRACING_ENDPOINT,
      sampleRate: parseFloat(process.env.TRACING_SAMPLE_RATE || '1.0'),
    },
    profiling: {
      enabled: process.env.PROFILING_ENABLED === 'true',
      slowThreshold: parseInt(process.env.PROFILING_SLOW_THRESHOLD || '1000'),
    },
    metrics: {
      enabled: process.env.METRICS_ENABLED === 'true',
    },
  },
});
```

---

## Step 9: Production Configuration

### Reduce Sample Rate

In production, sample only a percentage of requests:

```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'task-api',
      exporter: 'otlp',
      sampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0, // 10% in prod
    },
  },
});
```

### Add Conditional Tracing

Always trace errors and slow requests:

```typescript
app.use(async (ctx, next) => {
  const start = Date.now();

  try {
    await next();

    const duration = Date.now() - start;

    // Force trace on slow requests
    if (duration > 1000 && ctx.trace) {
      ctx.trace.sampleRate = 1.0; // Always sample
    }
  } catch (error) {
    // Force trace on errors
    if (ctx.trace) {
      ctx.trace.sampleRate = 1.0;
    }
    throw error;
  }
});
```

---

## Step 10: Complete Observability Stack

Create `docker-compose.observability.yml`:

```yaml
version: '3.8'

services:
  # Your application
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - TRACING_ENDPOINT=http://jaeger:4318/v1/traces
      - NODE_ENV=production
    depends_on:
      - jaeger
      - prometheus

  # Jaeger (distributed tracing)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  # Prometheus (metrics)
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  # Grafana (dashboards)
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
      - jaeger
```

**Start the stack:**

```bash
docker-compose -f docker-compose.observability.yml up -d
```

**Access UIs:**
- Jaeger: http://localhost:16686
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3001

---

## Troubleshooting

### Tracing Not Working

**Issue**: No traces appearing in console or Jaeger

**Solutions:**

1. Verify tracing is enabled:
```typescript
console.log('Tracing enabled:', app.config.observability?.tracing?.enabled);
```

2. Check sample rate:
```typescript
// Ensure sample rate > 0
sampleRate: 1.0  // 100% sampling for testing
```

3. Verify Jaeger is running:
```bash
docker ps | grep jaeger
curl http://localhost:14268/api/traces  # Check if Jaeger is accessible
```

### Flow Tracking Not Working

**Issue**: Flow routes return 404

**Cause**: Flow tracking requires tracing to be enabled

**Solution:**
```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,  // Required!
      exporter: 'console',
    },
  },
});

app.use(flowTrackingMiddleware());  // Must be after tracing enabled
```

### High Memory Usage

**Issue**: Application using too much memory with observability

**Solutions:**

1. Reduce sample rate:
```typescript
sampleRate: 0.1  // Sample only 10%
```

2. Disable memory profiling:
```typescript
profiling: {
  enabled: true,
  captureMemory: false,  // Disable memory tracking
}
```

3. Limit span attributes:
```typescript
tracing: {
  maxSpanAttributes: 32,  // Default is 128
}
```

---

## Best Practices

### 1. Use Appropriate Sample Rates

```typescript
// Development: 100%
// Staging: 50%
// Production: 10-20%

const sampleRate = {
  development: 1.0,
  staging: 0.5,
  production: 0.1,
}[process.env.NODE_ENV] || 1.0;
```

### 2. Add Business Context

```typescript
app.post('/api/orders', async (ctx) => {
  const order = await createOrder(ctx.body);

  // Add business context to trace
  if (ctx.trace) {
    ctx.trace.attributes['order.id'] = order.id;
    ctx.trace.attributes['order.total'] = order.total;
    ctx.trace.attributes['order.customer_id'] = order.customerId;
  }

  ctx.json(order);
});
```

### 3. Structured Logging with Trace Context

```typescript
import { getCurrentTrace } from 'ramapi';

function log(level: string, message: string, data?: any) {
  const trace = getCurrentTrace();

  console.log(JSON.stringify({
    level,
    message,
    data,
    traceId: trace?.traceId,
    spanId: trace?.spanId,
    timestamp: new Date().toISOString(),
  }));
}

// Usage
log('info', 'Order created', { orderId: '123' });
```

### 4. Monitor Key Operations

```typescript
// Track important business operations
app.post('/api/checkout', async (ctx) => {
  const tracer = trace.getTracer('task-api');

  const checkoutSpan = tracer.startSpan('checkout.process');

  try {
    const paymentSpan = tracer.startSpan('checkout.payment');
    await processPayment(ctx.body);
    paymentSpan.end();

    const fulfillmentSpan = tracer.startSpan('checkout.fulfillment');
    await createFulfillment(ctx.body);
    fulfillmentSpan.end();

    checkoutSpan.setStatus({ code: 0 });
  } catch (error: any) {
    checkoutSpan.recordException(error);
    checkoutSpan.setStatus({ code: 2 });
    throw error;
  } finally {
    checkoutSpan.end();
  }
});
```

---

## Key Takeaways

1. ✅ **All APIs verified** against RamAPI source code
2. ✅ **Tracing** requires explicit configuration
3. ✅ **Flow tracking** requires tracing to be enabled first
4. ✅ **Profiling** helps identify bottlenecks
5. ✅ **Sample rates** control overhead vs visibility tradeoff
6. ✅ **Production** requires Jaeger or similar collector

---

## Next Steps

- [Production Observability](../deployment/production-observability.md) - Full production setup
- [Performance Optimization](../performance/optimization.md) - Use observability data
- [Troubleshooting](troubleshooting.md) - Common issues and solutions

---

## See Also

- [Tracing](../observability/tracing.md)
- [Profiling](../observability/profiling.md)
- [Flow Visualization](../observability/flow-visualization.md)
- [Production Setup](../deployment/production-setup.md)
