# Custom Adapters

Learn how to create custom HTTP adapters for RamAPI to support different server backends and runtime environments.

> **Note**: This documentation has been verified against the RamAPI source code. Verified components:
> - ✅ `ServerAdapter` interface - All methods and properties verified
> - ✅ `RawRequestInfo` and `RawResponseData` types verified
> - ✅ `RequestHandler` type verified
> - ✅ `NodeHTTPAdapter` implementation reviewed and patterns confirmed
> - ✅ Adapter interface includes: `name`, `listen()`, `close()`, `onRequest()`, `getRequestInfo()`, `sendResponse()`, `parseBody()`
> - ⚠️ Bun, Deno adapters are conceptual examples for other runtimes

## Table of Contents

1. [Adapter Overview](#adapter-overview)
2. [Adapter Interface](#adapter-interface)
3. [Creating a Custom Adapter](#creating-a-custom-adapter)
4. [Built-in Adapters](#built-in-adapters)
5. [Advanced Patterns](#advanced-patterns)
6. [Testing Adapters](#testing-adapters)

---

## Adapter Overview

### What is an Adapter?

An adapter is a bridge between RamAPI's core routing/middleware system and the underlying HTTP server implementation. Adapters allow RamAPI to run on different server backends without changing application code.

```
┌─────────────────────┐
│   Your Application  │
│   (RamAPI Routes)   │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   RamAPI Core       │
│  (Router/Context)   │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   Server Adapter    │  ← Custom adapters plug in here
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   HTTP Server       │
│ (Node.js, uWS, etc) │
└─────────────────────┘
```

### Why Custom Adapters?

- **Different runtimes**: Bun, Deno, Cloudflare Workers
- **Performance**: uWebSockets.js, Hyper-Express
- **Special requirements**: HTTP/2, WebSocket, custom protocols
- **Testing**: Mock adapters for unit tests

---

## Adapter Interface

### ServerAdapter Interface

```typescript
interface ServerAdapter {
  /**
   * Adapter name (e.g., "node-http", "uwebsockets")
   */
  readonly name: string;

  /**
   * Start listening for incoming requests
   */
  listen(port: number, host: string): Promise<void>;

  /**
   * Stop the server and close all connections
   */
  close(): Promise<void>;

  /**
   * Register the request handler that processes all requests
   */
  onRequest(handler: RequestHandler): void;

  /**
   * Extract request information from raw request
   */
  getRequestInfo(raw: any): RawRequestInfo;

  /**
   * Send response to client
   */
  sendResponse(
    raw: any,
    statusCode: number,
    headers: Record<string, string>,
    body: Buffer | string
  ): void;

  /**
   * Parse request body from raw request
   */
  parseBody(raw: any): Promise<unknown>;

  /**
   * Optional: Supports streaming
   */
  readonly supportsStreaming?: boolean;

  /**
   * Optional: Supports HTTP/2
   */
  readonly supportsHTTP2?: boolean;
}
```

### Supporting Types

```typescript
/**
 * Normalized request information
 */
interface RawRequestInfo {
  method: string;
  url: string;
  headers: Record<string, string | string[]>;
}

/**
 * Response data to send to client
 */
interface RawResponseData {
  statusCode: number;
  headers: Record<string, string>;
  body: Buffer | string;
}

/**
 * Request handler type
 */
type RequestHandler = (
  requestInfo: RawRequestInfo,
  rawRequest: any
) => Promise<RawResponseData>;
```

---

## Creating a Custom Adapter

### Basic Adapter Template

```typescript
import type {
  ServerAdapter,
  RequestHandler,
  RawRequestInfo,
} from 'ramapi';

export class MyCustomAdapter implements ServerAdapter {
  readonly name = 'my-custom-adapter';
  readonly supportsStreaming = false;
  readonly supportsHTTP2 = false;

  private requestHandler?: RequestHandler;
  private server?: any; // Your server instance

  /**
   * Register request handler
   */
  onRequest(handler: RequestHandler): void {
    this.requestHandler = handler;
  }

  /**
   * Start server
   */
  async listen(port: number, host: string): Promise<void> {
    if (!this.requestHandler) {
      throw new Error('Request handler not registered');
    }

    // Create and start your server
    this.server = createYourServer({
      onRequest: async (req, res) => {
        const info = this.getRequestInfo(req);
        const raw = { req, res };

        const response = await this.requestHandler!(info, raw);
        this.sendResponse(raw, response.statusCode, response.headers, response.body);
      },
    });

    await this.server.listen(port, host);
    console.log(`🚀 Server (${this.name}) running at http://${host}:${port}`);
  }

  /**
   * Stop server
   */
  async close(): Promise<void> {
    if (this.server) {
      await this.server.close();
      console.log('🛑 Server stopped');
    }
  }

  /**
   * Extract request info
   */
  getRequestInfo(raw: any): RawRequestInfo {
    return {
      method: raw.method || 'GET',
      url: raw.url || '/',
      headers: raw.headers || {},
    };
  }

  /**
   * Send response
   */
  sendResponse(
    raw: any,
    statusCode: number,
    headers: Record<string, string>,
    body: Buffer | string
  ): void {
    const { res } = raw;
    res.statusCode = statusCode;

    for (const [key, value] of Object.entries(headers)) {
      res.setHeader(key, value);
    }

    res.end(body);
  }

  /**
   * Parse request body
   */
  async parseBody(raw: any): Promise<unknown> {
    const { req } = raw;
    // Implement body parsing based on content-type
    return {};
  }
}

/**
 * Factory function
 */
export function createMyCustomAdapter(config?: Record<string, any>): ServerAdapter {
  return new MyCustomAdapter();
}
```

### Using Your Custom Adapter

```typescript
import { createApp } from 'ramapi';
import { MyCustomAdapter } from './my-adapter.js';

const app = createApp({
  adapter: new MyCustomAdapter(),
});

app.get('/', (ctx) => {
  ctx.json({ message: 'Hello from custom adapter!' });
});

await app.listen(3000);
```

---

## Built-in Adapters

### Node.js HTTP Adapter

The default adapter using Node.js built-in `http` module:

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: { type: 'node-http' },
});

// Characteristics:
// - Stable and battle-tested
// - ~50-100k req/s
// - Full HTTP/1.1 support
// - Compatible with all Node.js middleware
```

### uWebSockets.js Adapter

Ultra-fast adapter using uWebSockets.js:

```typescript
import { createApp } from 'ramapi';

const app = createApp({
  adapter: { type: 'uwebsockets' },
});

// Characteristics:
// - Extremely fast (~400k+ req/s)
// - Lower memory usage
// - HTTP/1.1 and WebSocket support
// - Requires native compilation
```

---

## Advanced Patterns

### Bun Adapter Example

```typescript
import type { ServerAdapter, RequestHandler, RawRequestInfo } from 'ramapi';
import { serve, type Server } from 'bun';

export class BunAdapter implements ServerAdapter {
  readonly name = 'bun';
  readonly supportsStreaming = true;
  readonly supportsHTTP2 = false;

  private requestHandler?: RequestHandler;
  private server?: Server;

  onRequest(handler: RequestHandler): void {
    this.requestHandler = handler;
  }

  async listen(port: number, host: string): Promise<void> {
    if (!this.requestHandler) {
      throw new Error('Request handler not registered');
    }

    this.server = serve({
      port,
      hostname: host,
      fetch: async (req) => {
        const info: RawRequestInfo = {
          method: req.method,
          url: new URL(req.url).pathname + new URL(req.url).search,
          headers: Object.fromEntries(req.headers.entries()),
        };

        const raw = { req };

        try {
          const response = await this.requestHandler!(info, raw);

          return new Response(response.body, {
            status: response.statusCode,
            headers: response.headers,
          });
        } catch (error: any) {
          return new Response(
            JSON.stringify({ error: 'Internal Server Error' }),
            { status: 500, headers: { 'Content-Type': 'application/json' } }
          );
        }
      },
    });

    console.log(`🚀 RamAPI server (${this.name}) running at http://${host}:${port}`);
  }

  async close(): Promise<void> {
    if (this.server) {
      this.server.stop();
      console.log('🛑 RamAPI server stopped');
    }
  }

  getRequestInfo(raw: any): RawRequestInfo {
    const { req } = raw;
    return {
      method: req.method,
      url: new URL(req.url).pathname + new URL(req.url).search,
      headers: Object.fromEntries(req.headers.entries()),
    };
  }

  sendResponse(
    raw: any,
    statusCode: number,
    headers: Record<string, string>,
    body: Buffer | string
  ): void {
    // Not used in Bun adapter (uses Response in fetch handler)
  }

  async parseBody(raw: any): Promise<unknown> {
    const { req } = raw;
    const contentType = req.headers.get('content-type') || '';

    if (contentType.includes('application/json')) {
      return await req.json();
    }

    if (contentType.includes('application/x-www-form-urlencoded')) {
      const text = await req.text();
      const params = new URLSearchParams(text);
      const result: Record<string, string> = {};
      params.forEach((value, key) => {
        result[key] = value;
      });
      return result;
    }

    return await req.text();
  }
}

export function createBunAdapter(): ServerAdapter {
  return new BunAdapter();
}
```

### Deno Adapter Example

```typescript
import type { ServerAdapter, RequestHandler, RawRequestInfo } from 'ramapi';

export class DenoAdapter implements ServerAdapter {
  readonly name = 'deno';
  readonly supportsStreaming = true;
  readonly supportsHTTP2 = false;

  private requestHandler?: RequestHandler;
  private server?: Deno.HttpServer;

  onRequest(handler: RequestHandler): void {
    this.requestHandler = handler;
  }

  async listen(port: number, host: string): Promise<void> {
    if (!this.requestHandler) {
      throw new Error('Request handler not registered');
    }

    this.server = Deno.serve({
      port,
      hostname: host,
      handler: async (req) => {
        const url = new URL(req.url);
        const info: RawRequestInfo = {
          method: req.method,
          url: url.pathname + url.search,
          headers: Object.fromEntries(req.headers.entries()),
        };

        const raw = { req };

        try {
          const response = await this.requestHandler!(info, raw);

          return new Response(response.body, {
            status: response.statusCode,
            headers: response.headers,
          });
        } catch (error) {
          return new Response(
            JSON.stringify({ error: 'Internal Server Error' }),
            { status: 500, headers: { 'Content-Type': 'application/json' } }
          );
        }
      },
    });

    console.log(`🚀 RamAPI server (${this.name}) running at http://${host}:${port}`);
  }

  async close(): Promise<void> {
    if (this.server) {
      await this.server.shutdown();
      console.log('🛑 RamAPI server stopped');
    }
  }

  getRequestInfo(raw: any): RawRequestInfo {
    const { req } = raw;
    const url = new URL(req.url);
    return {
      method: req.method,
      url: url.pathname + url.search,
      headers: Object.fromEntries(req.headers.entries()),
    };
  }

  sendResponse(): void {
    // Not used in Deno adapter
  }

  async parseBody(raw: any): Promise<unknown> {
    const { req } = raw;
    const contentType = req.headers.get('content-type') || '';

    if (contentType.includes('application/json')) {
      return await req.json();
    }

    return await req.text();
  }
}

export function createDenoAdapter(): ServerAdapter {
  return new DenoAdapter();
}
```

### Mock Adapter for Testing

```typescript
import type { ServerAdapter, RequestHandler, RawRequestInfo } from 'ramapi';

export class MockAdapter implements ServerAdapter {
  readonly name = 'mock';
  readonly supportsStreaming = false;
  readonly supportsHTTP2 = false;

  private requestHandler?: RequestHandler;
  public requests: Array<{ info: RawRequestInfo; response: any }> = [];

  onRequest(handler: RequestHandler): void {
    this.requestHandler = handler;
  }

  async listen(_port: number, _host: string): Promise<void> {
    // No-op for testing
  }

  async close(): Promise<void> {
    // No-op for testing
  }

  /**
   * Simulate a request for testing
   */
  async simulateRequest(
    method: string,
    url: string,
    headers: Record<string, string> = {},
    body?: any
  ) {
    if (!this.requestHandler) {
      throw new Error('Request handler not registered');
    }

    const info: RawRequestInfo = {
      method,
      url,
      headers,
    };

    const raw = { body };

    const response = await this.requestHandler(info, raw);

    this.requests.push({ info, response });

    return response;
  }

  getRequestInfo(raw: any): RawRequestInfo {
    return {
      method: 'GET',
      url: '/',
      headers: {},
    };
  }

  sendResponse(): void {
    // No-op for testing
  }

  async parseBody(raw: any): Promise<unknown> {
    return raw.body;
  }

  /**
   * Clear recorded requests
   */
  clearRequests(): void {
    this.requests = [];
  }
}
```

**Using Mock Adapter in Tests:**

```typescript
import { describe, it, expect } from 'vitest';
import { createApp } from 'ramapi';
import { MockAdapter } from './mock-adapter.js';

describe('API Tests', () => {
  it('should respond to GET requests', async () => {
    const mockAdapter = new MockAdapter();
    const app = createApp({ adapter: mockAdapter });

    app.get('/users', (ctx) => {
      ctx.json({ users: [] });
    });

    await app.listen(3000);

    const response = await mockAdapter.simulateRequest('GET', '/users');

    expect(response.statusCode).toBe(200);
    expect(JSON.parse(response.body as string)).toEqual({ users: [] });
  });
});
```

---

## Testing Adapters

### Adapter Compliance Tests

```typescript
import { describe, it, expect } from 'vitest';
import type { ServerAdapter } from 'ramapi';

export function testAdapter(createAdapter: () => ServerAdapter) {
  describe('Adapter Compliance', () => {
    it('should have required properties', () => {
      const adapter = createAdapter();

      expect(adapter.name).toBeDefined();
      expect(typeof adapter.name).toBe('string');
    });

    it('should implement all required methods', () => {
      const adapter = createAdapter();

      expect(typeof adapter.listen).toBe('function');
      expect(typeof adapter.close).toBe('function');
      expect(typeof adapter.onRequest).toBe('function');
      expect(typeof adapter.getRequestInfo).toBe('function');
      expect(typeof adapter.sendResponse).toBe('function');
      expect(typeof adapter.parseBody).toBe('function');
    });

    it('should register request handler', () => {
      const adapter = createAdapter();
      const handler = async () => ({
        statusCode: 200,
        headers: {},
        body: 'OK',
      });

      expect(() => {
        adapter.onRequest(handler);
      }).not.toThrow();
    });

    it('should throw if listening without handler', async () => {
      const adapter = createAdapter();

      await expect(adapter.listen(3000, 'localhost')).rejects.toThrow();
    });
  });
}

// Use in your adapter tests
import { MyCustomAdapter } from './my-adapter.js';

testAdapter(() => new MyCustomAdapter());
```

---

## Best Practices

### 1. Error Handling

```typescript
export class SafeAdapter implements ServerAdapter {
  // ... other methods

  async listen(port: number, host: string): Promise<void> {
    if (!this.requestHandler) {
      throw new Error('Request handler not registered');
    }

    this.server = createServer(async (req, res) => {
      try {
        const info = this.getRequestInfo({ req, res });
        const raw = { req, res };

        const response = await this.requestHandler!(info, raw);
        this.sendResponse(raw, response.statusCode, response.headers, response.body);
      } catch (error) {
        console.error('Adapter error:', error);

        // Always send a response
        if (!res.headersSent) {
          res.statusCode = 500;
          res.setHeader('Content-Type', 'application/json');
          res.end(JSON.stringify({ error: 'Internal Server Error' }));
        }
      }
    });

    await this.server.listen(port, host);
  }
}
```

### 2. Performance Optimization

```typescript
export class OptimizedAdapter implements ServerAdapter {
  private requestHandler?: RequestHandler;

  // Reuse objects to reduce GC pressure
  private headerCache = new Map<string, Record<string, string>>();

  getRequestInfo(raw: any): RawRequestInfo {
    // Minimize allocations
    return {
      method: raw.method || 'GET',
      url: raw.url || '/',
      headers: raw.headers || {},
    };
  }

  sendResponse(
    raw: any,
    statusCode: number,
    headers: Record<string, string>,
    body: Buffer | string
  ): void {
    const { res } = raw;

    // Fast path for common status codes
    res.statusCode = statusCode;

    // Batch header writes
    const headerEntries = Object.entries(headers);
    for (let i = 0; i < headerEntries.length; i++) {
      res.setHeader(headerEntries[i][0], headerEntries[i][1]);
    }

    res.end(body);
  }
}
```

### 3. Graceful Shutdown

```typescript
export class GracefulAdapter implements ServerAdapter {
  private activeConnections = new Set<any>();

  async listen(port: number, host: string): Promise<void> {
    this.server = createServer(async (req, res) => {
      this.activeConnections.add(res);

      res.on('close', () => {
        this.activeConnections.delete(res);
      });

      // Handle request...
    });

    await this.server.listen(port, host);
  }

  async close(): Promise<void> {
    if (!this.server) return;

    // Stop accepting new connections
    await new Promise<void>((resolve) => {
      this.server!.close(() => resolve());
    });

    // Wait for active connections to finish (with timeout)
    const timeout = setTimeout(() => {
      this.activeConnections.forEach(conn => conn.destroy());
    }, 10000);

    while (this.activeConnections.size > 0) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }

    clearTimeout(timeout);
  }
}
```

---

## See Also

- [Getting Started](../getting-started/quick-start.md)
- [Performance Optimization](../performance/optimization.md)
- [Testing Strategies](testing.md)
- [Scaling & Architecture](scaling.md)
