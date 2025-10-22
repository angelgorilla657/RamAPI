# HTTP Adapters

RamAPI uses a pluggable adapter pattern for HTTP servers, allowing you to choose between different server implementations based on your performance needs.

## Table of Contents

1. [Adapter Overview](#adapter-overview)
2. [Node.js HTTP Adapter](#nodejs-http-adapter)
3. [uWebSockets.js Adapter](#uwebsocketsjs-adapter)
4. [Smart Adapter Selection](#smart-adapter-selection)
5. [Configuration](#configuration)
6. [Migration Guide](#migration-guide)
7. [Troubleshooting](#troubleshooting)

---

## Adapter Overview

### What is an Adapter?

An adapter abstracts the underlying HTTP server implementation, allowing RamAPI to work with different server backends without changing your application code.

```typescript
// Same code, different performance
const app = createApp({
  adapter: { type: 'node-http' },    // 124K req/s
  adapter: { type: 'uwebsockets' },  // 350K req/s
});
```

### Available Adapters

| Adapter | Performance | Stability | Compatibility | Setup |
|---------|-------------|-----------|---------------|-------|
| **node-http** | 124K req/s | Excellent | 100% | Zero config |
| **uwebsockets** | 350K req/s | Good | 95% | Requires install |

### Benefits of Adapter Pattern

1. **Flexibility**: Switch servers without code changes
2. **Performance**: Choose the right performance level
3. **Future-proof**: New adapters can be added without breaking changes
4. **Compatibility**: Fallback to stable options when needed

---

## Node.js HTTP Adapter

### Overview

The Node.js HTTP adapter uses the standard `http` module from Node.js core. It's the default adapter and requires no additional dependencies.

### Performance Characteristics

```
Requests/sec:        124,000
Latency (avg):       0.80ms
Latency (p95):       2.1ms
Latency (p99):       4.8ms
Memory usage:        Normal
CPU usage:           Moderate
```

### Features

- **HTTP/1.1**: Full support
- **HTTP/2**: Not supported (use uWebSockets)
- **Streaming**: Full support
- **WebSockets**: Not supported (use uWebSockets)
- **Keep-Alive**: Supported
- **Request pipelining**: Supported

### When to Use

**Best for:**
- Default choice for most applications
- Maximum compatibility required
- Cloud platforms (Vercel, Railway, Fly.io)
- Windows development
- CI/CD environments
- gRPC integration (currently required)
- Traffic < 100K req/s

**Ideal scenarios:**
```typescript
// Cloud deployments
const app = createApp({
  adapter: { type: 'node-http' },
});

// Development environments
const app = createApp(); // Auto-falls back to node-http if uws unavailable
```

### Example

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: {
    type: 'node-http',
    options: {
      // Node.js http.createServer options
      keepAlive: true,
      keepAliveTimeout: 5000,
    },
  },
});

app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});

await app.listen(3000);
// 📡 Using Node.js HTTP adapter
// 🚀 RamAPI server running at http://0.0.0.0:3000
```

---

## uWebSockets.js Adapter

### Overview

The uWebSockets.js adapter uses the ultra-fast [uWebSockets.js](https://github.com/uNetworking/uWebSockets.js) library, providing 2-3x better performance than Node.js HTTP.

### Performance Characteristics

```
Requests/sec:        350,000
Latency (avg):       0.28ms
Latency (p95):       0.9ms
Latency (p99):       2.1ms
Memory usage:        Lower
CPU usage:           Lower
```

### Features

- **HTTP/1.1**: Full support
- **HTTP/2**: Full support
- **Streaming**: Full support
- **WebSockets**: Native support
- **SSL/TLS**: Native support
- **Backpressure**: Automatic handling

### When to Use

**Best for:**
- Maximum performance required
- High traffic (>100K req/s)
- Cost optimization (fewer servers)
- Linux/macOS production servers
- Microservices architecture
- WebSocket applications
- Real-time APIs

**Ideal scenarios:**
```typescript
// High-performance API
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// WebSocket support
const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      // uWebSockets options
      idleTimeout: 120,
      maxBackpressure: 1024 * 1024,
    },
  },
});
```

### Installation

```bash
npm install uWebSockets.js
```

**Platform support:**
- Linux: Full support
- macOS: Full support
- Windows: Limited (use WSL2)

### Example

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      // uWebSockets options
      idleTimeout: 120,
      maxBackpressure: 1024 * 1024,
      maxPayloadLength: 16 * 1024 * 1024,
    },
  },
});

app.get('/api/fast', async (ctx) => {
  ctx.json({ message: 'Ultra fast!' });
});

await app.listen(3000);
// 🚀 Using uWebSockets adapter for maximum performance
//    💡 Tip: ~2-3x faster than Node.js HTTP
// 🚀 RamAPI server running at http://0.0.0.0:3000
```

### SSL/TLS Configuration

```typescript
const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      ssl: {
        key_file_name: '/path/to/key.pem',
        cert_file_name: '/path/to/cert.pem',
        passphrase: 'optional_passphrase',
      },
    },
  },
});
```

---

## Smart Adapter Selection

### Automatic Selection

RamAPI automatically selects the best adapter based on availability and requirements:

```typescript
// No adapter specified - RamAPI chooses automatically
const app = createApp();

// Selection priority:
// 1. Try uWebSockets for maximum performance
// 2. Fallback to Node.js HTTP if unavailable
```

**Selection logic:**

```
1. User explicitly configured adapter? → Use it
2. gRPC enabled? → Use Node.js HTTP (required)
3. uWebSockets available? → Use uWebSockets (performance)
4. Otherwise → Use Node.js HTTP (fallback)
```

### Console Output

**With uWebSockets installed:**
```
🚀 Using uWebSockets adapter for maximum performance
   💡 Tip: ~2-3x faster than Node.js HTTP
🚀 RamAPI server running at http://0.0.0.0:3000
```

**Without uWebSockets:**
```
📡 Using Node.js HTTP adapter (uWebSockets not available)
   💡 Tip: Install uWebSockets.js for 2-3x performance boost
   npm install uWebSockets.js
🚀 RamAPI server running at http://0.0.0.0:3000
```

**With gRPC enabled:**
```
📡 Using Node.js HTTP adapter (required for gRPC)
🚀 RamAPI server running at http://0.0.0.0:3000
```

---

## Configuration

### Explicit Adapter Selection

```typescript
// Force Node.js HTTP
const app = createApp({
  adapter: { type: 'node-http' },
});

// Force uWebSockets
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// Automatic (recommended)
const app = createApp(); // Smart selection
```

### Adapter-Specific Options

**Node.js HTTP options:**

```typescript
const app = createApp({
  adapter: {
    type: 'node-http',
    options: {
      // Node.js http.createServer options
      keepAlive: true,
      keepAliveTimeout: 5000,
      maxHeaderSize: 16384,
      insecureHTTPParser: false,
    },
  },
});
```

**uWebSockets options:**

```typescript
const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      // Connection settings
      idleTimeout: 120,                    // Seconds before idle connection closes
      maxBackpressure: 1024 * 1024,        // Max bytes buffered per connection
      maxPayloadLength: 16 * 1024 * 1024,  // Max request body size (16MB)

      // SSL/TLS
      ssl: {
        key_file_name: '/path/to/key.pem',
        cert_file_name: '/path/to/cert.pem',
        passphrase: 'optional',
        dh_params_file_name: '/path/to/dhparams.pem',
        ssl_prefer_low_memory_usage: false,
      },

      // Performance tuning
      compression: 0, // 0 = disabled, 1 = enabled, 2 = enabled + shared compressor
    },
  },
});
```

### Environment-Based Selection

```typescript
const app = createApp({
  adapter: {
    type: process.env.HTTP_ADAPTER as 'node-http' | 'uwebsockets',
  },
});

// Production: HTTP_ADAPTER=uwebsockets
// Development: HTTP_ADAPTER=node-http
```

---

## Migration Guide

### From Node.js HTTP to uWebSockets

**Step 1: Install uWebSockets**

```bash
npm install uWebSockets.js
```

**Step 2: Update configuration**

```typescript
// Before
const app = createApp({
  adapter: { type: 'node-http' },
});

// After
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// Or use automatic selection
const app = createApp(); // Will auto-select uWebSockets
```

**Step 3: Test thoroughly**

- uWebSockets has stricter HTTP parsing
- Some edge cases may behave differently
- Test with your actual traffic patterns

**Step 4: Monitor performance**

```typescript
import { getMetrics } from 'ramapi';

setInterval(() => {
  const metrics = getMetrics();
  console.log('Requests/sec:', metrics.requestsPerSecond);
  console.log('P95 latency:', metrics.p95Latency);
}, 60000);
```

### From uWebSockets to Node.js HTTP

Sometimes you need to fallback to Node.js HTTP:

```typescript
// Before
const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// After
const app = createApp({
  adapter: { type: 'node-http' },
});
```

**Reasons to fallback:**
- Compatibility issues
- Platform constraints (Windows without WSL)
- gRPC integration required
- Debugging/troubleshooting

---

## Troubleshooting

### uWebSockets Installation Fails

**Problem:** `npm install uWebSockets.js` fails

**Solutions:**

1. **Install build tools:**
   ```bash
   # macOS
   xcode-select --install

   # Ubuntu/Debian
   sudo apt-get install build-essential

   # Windows
   npm install --global windows-build-tools
   ```

2. **Use prebuilt binaries:**
   ```bash
   npm install uWebSockets.js --force
   ```

3. **Fallback to Node.js:**
   ```typescript
   const app = createApp({
     adapter: { type: 'node-http' },
   });
   ```

### Performance Not Improving

**Problem:** Switched to uWebSockets but no performance gain

**Check:**

1. **Verify adapter is actually being used:**
   ```
   🚀 Using uWebSockets adapter for maximum performance
   ```

2. **Profile your application:**
   ```typescript
   const app = createApp({
     adapter: { type: 'uwebsockets' },
     observability: {
       profiling: { enabled: true },
     },
   });
   ```

3. **Bottleneck might be elsewhere:**
   - Database queries
   - External API calls
   - Middleware overhead
   - Business logic

### Adapter Not Loading

**Problem:** Explicit adapter configuration ignored

**Solution:** Check error messages in console

```typescript
try {
  const app = createApp({
    adapter: { type: 'uwebsockets' },
  });
} catch (error) {
  console.error('Adapter error:', error);
  // Fallback
  const app = createApp({
    adapter: { type: 'node-http' },
  });
}
```

### gRPC Requires Node.js HTTP

**Problem:** gRPC doesn't work with uWebSockets

**Explanation:** Currently, gRPC integration requires Node.js HTTP/2 module

**Solution:**

```typescript
const app = createApp({
  adapter: { type: 'node-http' }, // Required for gRPC
  protocols: {
    grpc: { port: 50051 },
  },
});
```

Future versions may support running gRPC on a separate port.

---

## Performance Comparison

### Simple JSON Response

```typescript
app.get('/api/hello', async (ctx) => {
  ctx.json({ message: 'Hello World' });
});
```

| Adapter | Req/s | Latency (avg) | Latency (p95) |
|---------|-------|---------------|---------------|
| uWebSockets | 350,000 | 0.28ms | 0.9ms |
| Node.js HTTP | 124,000 | 0.80ms | 2.1ms |
| **Improvement** | **2.8x faster** | **2.9x faster** | **2.3x faster** |

### With Middleware Stack

```typescript
app.use(authenticate);
app.use(logger);
app.get('/api/users', async (ctx) => {
  ctx.json({ users: await db.getUsers() });
});
```

| Adapter | Req/s | Latency (avg) | Latency (p95) |
|---------|-------|---------------|---------------|
| uWebSockets | 285,000 | 0.35ms | 1.2ms |
| Node.js HTTP | 98,000 | 1.02ms | 2.8ms |
| **Improvement** | **2.9x faster** | **2.9x faster** | **2.3x faster** |

---

## Complete Example

```typescript
import { createApp, getMetrics } from 'ramapi';

// Create app with automatic adapter selection
const app = createApp({
  // RamAPI will automatically select the best adapter:
  // - uWebSockets if available (350K req/s)
  // - Node.js HTTP as fallback (124K req/s)

  observability: {
    metrics: { enabled: true },
  },
});

// Your routes work the same regardless of adapter
app.get('/api/users', async (ctx) => {
  ctx.json({ users: [] });
});

app.get('/api/metrics', async (ctx) => {
  const metrics = getMetrics();
  ctx.json({
    adapter: 'Check console output',
    requestsPerSecond: metrics.requestsPerSecond,
    p95Latency: metrics.p95Latency,
  });
});

// Start server - adapter is selected automatically
await app.listen(3000);
```

**Console output example:**

```
🚀 Using uWebSockets adapter for maximum performance
   💡 Tip: ~2-3x faster than Node.js HTTP
🚀 RamAPI server running at http://0.0.0.0:3000
```

---

## Next Steps

- [Performance Overview](overview.md)
- [Benchmarks](benchmarks.md)
- [Optimization Guide](optimization-guide.md)

---

**Need help?** Check the [Troubleshooting Guide](../guides/troubleshooting.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
