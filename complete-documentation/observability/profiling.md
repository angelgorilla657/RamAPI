# Performance Profiling

Automatic performance profiling with bottleneck detection, timeline visualization, and performance budgets. Identify slow operations without manual instrumentation.

## Table of Contents

1. [Overview](#overview)
2. [Configuration](#configuration)
3. [Timeline Visualization](#timeline-visualization)
4. [Bottleneck Detection](#bottleneck-detection)
5. [Performance Budgets](#performance-budgets)
6. [API Access](#api-access)

---

## Overview

RamAPI automatically profiles every request, breaking down time spent in:
- Routing
- Authentication
- Validation
- Middleware execution
- Handler logic
- Response serialization

### Automatic Profiling

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 1000, // Flag requests >1s
      autoDetectBottlenecks: true,
    },
  },
});
```

---

## Configuration

### ProfilingConfig Interface

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

### Basic Setup

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 1000, // ms
    },
  },
});
```

### Full Configuration

```typescript
const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      captureMemory: true, // Enable memory profiling
      bufferSize: 100, // Keep last 100 profiles
      slowThreshold: 1000, // >1s is slow
      enableBudgets: true, // Track budget violations
      autoDetectBottlenecks: true, // Auto-detect issues
      captureStacks: true, // Capture stack traces
    },
  },
});
```

---

## Timeline Visualization

### Profile Structure

```typescript
interface RequestProfile {
  traceId: string;
  operationName: string;
  method: string;
  path: string;
  timestamp: number;

  stages: {
    routing: number;
    validation?: number;
    authentication?: number;
    middleware: StageTiming[];
    handler: number;
    serialization: number;
    total: number;
  };

  breakdown: StageTiming[];
  memory?: MemoryProfile;
  slow: boolean;
  bottlenecks: string[];
}
```

### Accessing Profile Data

```typescript
import { getProfiles } from 'ramapi';

app.get('/debug/profiles', async (ctx) => {
  const profiles = await getProfiles({
    limit: 10,
    slowOnly: true,
  });

  ctx.json({ profiles });
});
```

### Timeline Breakdown

```typescript
app.get('/debug/profile/:traceId', async (ctx) => {
  const profile = await getProfileByTraceId(ctx.params.traceId);

  // Breakdown shows time spent in each stage
  ctx.json({
    traceId: profile.traceId,
    total: profile.stages.total,
    breakdown: profile.breakdown.map(stage => ({
      name: stage.name,
      duration: stage.duration,
      percentage: (stage.duration / profile.stages.total * 100).toFixed(1) + '%',
    })),
  });
});
```

**Example Output:**
```json
{
  "traceId": "3f2504e04f8911edb13900505634b5f1",
  "total": 156.5,
  "breakdown": [
    { "name": "routing", "duration": 2.1, "percentage": "1.3%" },
    { "name": "auth", "duration": 8.4, "percentage": "5.4%" },
    { "name": "validation", "duration": 5.2, "percentage": "3.3%" },
    { "name": "handler", "duration": 125.8, "percentage": "80.4%" },
    { "name": "serialization", "duration": 15.0, "percentage": "9.6%" }
  ]
}
```

---

## Bottleneck Detection

### Automatic Detection

RamAPI automatically detects common performance issues:

- **Slow Operations**: Operations exceeding threshold
- **Slow Middleware**: Middleware taking too long
- **N+1 Patterns**: Multiple similar operations
- **Memory Leaks**: Growing memory usage
- **High CPU**: Excessive CPU usage

### Example Detection

```typescript
const profile = {
  traceId: '3f2504...',
  slow: true,
  bottlenecks: [
    'Handler took 2500ms (exceeds 1000ms threshold)',
    'Database middleware took 2000ms (80% of total time)',
    'Possible N+1 query pattern detected',
  ],
};
```

### Accessing Bottleneck Data

```typescript
app.get('/debug/bottlenecks', async (ctx) => {
  const profiles = await getProfiles({ slowOnly: true });

  const bottlenecks = profiles
    .filter(p => p.bottlenecks.length > 0)
    .map(p => ({
      traceId: p.traceId,
      operation: p.operationName,
      duration: p.stages.total,
      bottlenecks: p.bottlenecks,
    }));

  ctx.json({ bottlenecks });
});
```

---

## Performance Budgets

### Setting Budgets

```typescript
import { setPerformanceBudget } from 'ramapi';

// Set budget for user endpoints
setPerformanceBudget('GET /api/users/:id', {
  budget: 100, // Warning at 100ms
  p95Threshold: 200, // Alert at 200ms
});

// Set budget for dashboard
setPerformanceBudget('GET /api/dashboard', {
  budget: 500,
  p95Threshold: 1000,
});
```

### Tracking Violations

```typescript
app.get('/debug/budgets', async (ctx) => {
  const budgets = getPerformanceBudgets();

  ctx.json({
    budgets: budgets.map(b => ({
      operation: b.operationName,
      budget: b.budget,
      exceeded: b.exceeded,
      violations: b.violations,
      lastViolation: b.lastViolation,
    })),
  });
});
```

---

## API Access

### Profile Endpoints

```typescript
// Get all profiles
app.get('/debug/profiles', async (ctx) => {
  const { limit = 10, slowOnly = false } = ctx.query;

  const profiles = await getProfiles({
    limit: parseInt(limit as string),
    slowOnly: slowOnly === 'true',
  });

  ctx.json({ profiles });
});

// Get specific profile
app.get('/debug/profile/:traceId', async (ctx) => {
  const profile = await getProfileByTraceId(ctx.params.traceId);
  ctx.json({ profile });
});

// Get profile statistics
app.get('/debug/stats', async (ctx) => {
  const stats = getProfileStats();
  ctx.json({ stats });
});

// Get slow profiles
app.get('/debug/slow', async (ctx) => {
  const profiles = await getProfiles({
    slowOnly: true,
    limit: 20,
  });

  ctx.json({ profiles });
});
```

### Complete Example

```typescript
import { createApp, getProfiles, getProfileStats } from 'ramapi';

const app = createApp({
  observability: {
    profiling: {
      enabled: true,
      slowThreshold: 1000,
      autoDetectBottlenecks: true,
      enableBudgets: true,
    },
  },
});

// Debug dashboard
app.get('/debug/dashboard', async (ctx) => {
  const stats = getProfileStats();
  const slowProfiles = await getProfiles({ slowOnly: true, limit: 10 });

  ctx.html(`
    <h1>Performance Dashboard</h1>
    <h2>Statistics</h2>
    <ul>
      <li>Total Requests: ${stats.totalRequests}</li>
      <li>Slow Requests: ${stats.slowRequests}</li>
      <li>P95 Duration: ${stats.p95Duration.toFixed(2)}ms</li>
      <li>Budget Violations: ${stats.budgetViolations}</li>
    </ul>

    <h2>Recent Slow Requests</h2>
    ${slowProfiles.map(p => `
      <div>
        <strong>${p.method} ${p.path}</strong>
        (${p.stages.total.toFixed(2)}ms)
        ${p.bottlenecks.length > 0 ? `
          <ul>
            ${p.bottlenecks.map(b => `<li>${b}</li>`).join('')}
          </ul>
        ` : ''}
      </div>
    `).join('')}
  `);
});

app.listen(3000);
```

---

## Next Steps

- [Flow Visualization](flow-visualization.md)
- [Distributed Tracing](tracing.md)
- [Metrics & Logging](metrics-and-logging.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md).
