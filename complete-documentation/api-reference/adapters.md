# Adapters API Reference

Complete API reference for RamAPI's HTTP server adapters.

## Table of Contents

1. [ServerAdapter Interface](#serveradapter-interface)
2. [createAdapter()](#createadapter)
3. [getAvailableAdapters()](#getavailableadapters)
4. [NodeHTTPAdapter](#nodehttpadapter)
5. [UWebSocketsAdapter](#uwebsocketsadapter)
6. [Custom Adapters](#custom-adapters)

---

## ServerAdapter Interface

Interface that all HTTP adapters must implement.

### Type Definition

```typescript
interface ServerAdapter {
  readonly name: string;
  listen(port: number, host: string): Promise<void>;
  close(): Promise<void>;
  onRequest(handler: RequestHandler): void;
  getRequestInfo(raw: any): RawRequestInfo;
  sendResponse(
    raw: any,
    statusCode: number,
    headers: Record<string, string>,
    body: Buffer | string
  ): void;
  parseBody(raw: any): Promise<unknown>;
  readonly supportsStreaming?: boolean;
  readonly supportsHTTP2?: boolean;
}
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `string` | Adapter name (e.g., 'node-http', 'uwebsockets') |
| `supportsStreaming` | `boolean` | Whether adapter supports response streaming |
| `supportsHTTP2` | `boolean` | Whether adapter supports HTTP/2 |

### Methods

#### listen()

Start listening for HTTP requests.

```typescript
listen(port: number, host: string): Promise<void>
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `port` | `number` | Port to listen on |
| `host` | `string` | Host to bind to |

---

#### close()

Stop the HTTP server.

```typescript
close(): Promise<void>
```

---

#### onRequest()

Register request handler.

```typescript
onRequest(handler: RequestHandler): void
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handler` | `RequestHandler` | Function to handle requests |

---

#### getRequestInfo()

Extract normalized request information.

```typescript
getRequestInfo(raw: any): RawRequestInfo
```

**Returns:**

```typescript
interface RawRequestInfo {
  method: string;
  url: string;
  headers: Record<string, string | string[]>;
}
```

---

#### sendResponse()

Send HTTP response.

```typescript
sendResponse(
  raw: any,
  statusCode: number,
  headers: Record<string, string>,
  body: Buffer | string
): void
```

---

#### parseBody()

Parse request body based on Content-Type.

```typescript
parseBody(raw: any): Promise<unknown>
```

---

## createAdapter()

Factory function to create HTTP adapter instances.

### Signature

```typescript
function createAdapter(
  type?: 'node-http' | 'uwebsockets',
  options?: Record<string, any>
): ServerAdapter
```

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `type` | `'node-http' \| 'uwebsockets'` | No | `'node-http'` | Adapter type |
| `options` | `Record<string, any>` | No | `{}` | Adapter-specific options |

### Returns

`ServerAdapter` - Configured adapter instance

### Examples

**Node.js HTTP adapter:**

```typescript
import { createAdapter } from 'ramapi';

const adapter = createAdapter('node-http');
```

**uWebSockets adapter:**

```typescript
const adapter = createAdapter('uwebsockets', {
  idleTimeout: 120,
  maxBackpressure: 1024 * 1024,
});
```

**With server:**

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      idleTimeout: 120,
    },
  },
});
```

---

## getAvailableAdapters()

Get list of available adapter types.

### Signature

```typescript
function getAvailableAdapters(): string[]
```

### Returns

`string[]` - Array of available adapter types

### Example

```typescript
import { getAvailableAdapters } from 'ramapi';

const adapters = getAvailableAdapters();
console.log(adapters);
// ['node-http', 'uwebsockets']
```

---

## NodeHTTPAdapter

Built-in adapter using Node.js `http` module.

### Features

- **Performance:** 124,000 req/s
- **HTTP/1.1:** Full support
- **HTTP/2:** Not supported
- **Streaming:** Supported
- **Platform:** All (Windows, macOS, Linux)
- **Dependencies:** None (uses Node.js core)

### Options

Uses standard Node.js `http.createServer()` options.

```typescript
interface NodeHTTPOptions {
  keepAlive?: boolean;
  keepAliveTimeout?: number;
  maxHeaderSize?: number;
  insecureHTTPParser?: boolean;
}
```

### Example

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: {
    type: 'node-http',
    options: {
      keepAlive: true,
      keepAliveTimeout: 5000,
      maxHeaderSize: 16384,
    },
  },
});

await app.listen(3000);
// 📡 Using Node.js HTTP adapter
// 🚀 RamAPI server running at http://0.0.0.0:3000
```

### When to Use

**Best for:**
- Default choice for most applications
- Cloud platforms (Vercel, Railway, Fly.io)
- Windows development
- CI/CD environments
- Maximum compatibility
- gRPC integration (currently required)

---

## UWebSocketsAdapter

High-performance adapter using uWebSockets.js.

### Features

- **Performance:** 350,000 req/s (2-3x faster than Node.js)
- **HTTP/1.1:** Full support
- **HTTP/2:** Full support
- **Streaming:** Supported
- **WebSockets:** Native support
- **Platform:** Linux, macOS (limited Windows support)
- **Dependencies:** Requires `uWebSockets.js` package

### Installation

```bash
npm install uWebSockets.js
```

### Options

```typescript
interface UWebSocketsOptions {
  idleTimeout?: number;
  maxBackpressure?: number;
  maxPayloadLength?: number;
  compression?: 0 | 1 | 2;
  ssl?: {
    key_file_name: string;
    cert_file_name: string;
    passphrase?: string;
    dh_params_file_name?: string;
    ssl_prefer_low_memory_usage?: boolean;
  };
}
```

#### idleTimeout

- **Type:** `number`
- **Default:** `120`
- **Description:** Seconds before idle connection closes

#### maxBackpressure

- **Type:** `number`
- **Default:** `1024 * 1024` (1MB)
- **Description:** Max bytes buffered per connection

#### maxPayloadLength

- **Type:** `number`
- **Default:** `16 * 1024 * 1024` (16MB)
- **Description:** Max request body size

#### compression

- **Type:** `0 | 1 | 2`
- **Default:** `0`
- **Description:**
  - `0` - Disabled
  - `1` - Enabled
  - `2` - Enabled with shared compressor

#### ssl

- **Type:** `object`
- **Description:** SSL/TLS configuration

### Examples

**Basic usage:**

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: {
    type: 'uwebsockets',
  },
});

await app.listen(3000);
// 🚀 Using uWebSockets adapter for maximum performance
//    💡 Tip: ~2-3x faster than Node.js HTTP
// 🚀 RamAPI server running at http://0.0.0.0:3000
```

**With options:**

```typescript
const app = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      idleTimeout: 120,
      maxBackpressure: 1024 * 1024,
      maxPayloadLength: 16 * 1024 * 1024,
      compression: 1,
    },
  },
});
```

**With SSL:**

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

### When to Use

**Best for:**
- High-performance APIs (>100K req/s)
- Cost optimization (fewer servers needed)
- Linux/macOS production servers
- Microservices architecture
- WebSocket applications
- Real-time APIs

---

## Custom Adapters

### Creating Custom Adapters

Implement the `ServerAdapter` interface:

```typescript
import { ServerAdapter, RequestHandler, RawRequestInfo } from 'ramapi';

class MyCustomAdapter implements ServerAdapter {
  readonly name = 'my-custom-adapter';
  readonly supportsStreaming = true;
  readonly supportsHTTP2 = false;

  private requestHandler?: RequestHandler;
  private server?: any;

  async listen(port: number, host: string): Promise<void> {
    this.server = createMyServer();

    this.server.on('request', async (req, res) => {
      const requestInfo = this.getRequestInfo({ req, res });
      const response = await this.requestHandler!(requestInfo, { req, res });

      this.sendResponse(
        { req, res },
        response.statusCode,
        response.headers,
        response.body
      );
    });

    await this.server.listen(port, host);
    console.log(`🚀 My custom adapter listening on ${host}:${port}`);
  }

  async close(): Promise<void> {
    if (this.server) {
      await this.server.close();
    }
  }

  onRequest(handler: RequestHandler): void {
    this.requestHandler = handler;
  }

  getRequestInfo(raw: any): RawRequestInfo {
    const { req } = raw;
    return {
      method: req.method || 'GET',
      url: req.url || '/',
      headers: req.headers,
    };
  }

  sendResponse(
    raw: any,
    statusCode: number,
    headers: Record<string, string>,
    body: Buffer | string
  ): void {
    const { res } = raw;
    res.writeHead(statusCode, headers);
    res.end(body);
  }

  async parseBody(raw: any): Promise<unknown> {
    const { req } = raw;
    return new Promise((resolve) => {
      let body = '';
      req.on('data', (chunk: Buffer) => {
        body += chunk.toString();
      });
      req.on('end', () => {
        resolve(JSON.parse(body));
      });
    });
  }
}
```

### Registering Custom Adapter

```typescript
import { createApp } from 'ramapi';

const customAdapter = new MyCustomAdapter();

const app = createApp();
// Manually set adapter (advanced usage)
```

---

## Adapter Comparison

### Performance

| Adapter | Req/s | Latency (avg) | Latency (p95) |
|---------|-------|---------------|---------------|
| **uWebSockets** | 350,000 | 0.28ms | 0.9ms |
| **Node.js HTTP** | 124,000 | 0.80ms | 2.1ms |

### Features

| Feature | Node.js HTTP | uWebSockets |
|---------|--------------|-------------|
| **HTTP/1.1** | ✅ | ✅ |
| **HTTP/2** | ❌ | ✅ |
| **Streaming** | ✅ | ✅ |
| **WebSockets** | ❌ | ✅ |
| **SSL/TLS** | ✅ | ✅ |
| **Zero Dependencies** | ✅ | ❌ |
| **Windows Support** | ✅ | Limited |

### Use Cases

| Scenario | Recommended |
|----------|-------------|
| Default choice | Node.js HTTP |
| High performance (>100K req/s) | uWebSockets |
| Cloud platforms | Node.js HTTP |
| Cost optimization | uWebSockets |
| WebSocket support | uWebSockets |
| gRPC integration | Node.js HTTP |
| Maximum compatibility | Node.js HTTP |

---

## Automatic Adapter Selection

RamAPI automatically selects the best adapter:

```typescript
const app = createApp(); // No adapter specified

// Selection logic:
// 1. Check if gRPC is enabled → Use Node.js HTTP
// 2. Try uWebSockets → Use if available
// 3. Fallback to Node.js HTTP
```

**Console output examples:**

```
🚀 Using uWebSockets adapter for maximum performance
   💡 Tip: ~2-3x faster than Node.js HTTP
```

```
📡 Using Node.js HTTP adapter (uWebSockets not available)
   💡 Tip: Install uWebSockets.js for 2-3x performance boost
   npm install uWebSockets.js
```

```
📡 Using Node.js HTTP adapter (required for gRPC)
```

---

## Complete Example

```typescript
import { createApp, createAdapter } from 'ramapi';

// Option 1: Via server config
const app1 = createApp({
  adapter: {
    type: 'uwebsockets',
    options: {
      idleTimeout: 120,
      maxBackpressure: 1024 * 1024,
    },
  },
});

// Option 2: Create adapter manually
const adapter = createAdapter('uwebsockets', {
  idleTimeout: 120,
});

// Option 3: Automatic selection
const app2 = createApp(); // Tries uWebSockets, falls back to Node.js

// Routes work the same regardless of adapter
app1.get('/users', async (ctx) => {
  ctx.json({ users: [] });
});

await app1.listen(3000);
```

---

## See Also

- [HTTP Adapters Guide](../performance/http-adapters.md) - Detailed adapter guide
- [Performance Benchmarks](../performance/benchmarks.md) - Adapter performance
- [Server API](server.md) - Server configuration

---

**Need help?** Check the [HTTP Adapters Guide](../performance/http-adapters.md) or [GitHub Issues](https://github.com/yourusername/ramapi/issues).
