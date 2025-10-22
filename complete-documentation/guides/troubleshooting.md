# Troubleshooting Guide

Common issues, solutions, and FAQs for RamAPI.

> **Verification Status**: All solutions based on verified RamAPI APIs and common Node.js issues

## Table of Contents

1. [Installation & Setup](#installation--setup)
2. [Server Issues](#server-issues)
3. [Routing Problems](#routing-problems)
4. [Middleware Issues](#middleware-issues)
5. [Authentication Problems](#authentication-problems)
6. [Observability Issues](#observability-issues)
7. [Performance Problems](#performance-problems)
8. [TypeScript Errors](#typescript-errors)
9. [FAQ](#faq)

---

## Installation & Setup

### Issue: Module not found errors

**Error:**
```
Error: Cannot find module 'ramapi'
```

**Solutions:**

1. Ensure RamAPI is installed:
```bash
npm install ramapi```

2. Check your package.json:
```json
{
  "dependencies": {
    "ramapi": "^0.1.0"
  }
}
```

3. Clear node_modules and reinstall:
```bash
rm -rf node_modules package-lock.json
npm install
```

---

### Issue: ESM vs CommonJS errors

**Error:**
```
require() of ES Module not supported
```

**Solution:**

Add `"type": "module"` to package.json:
```json
{
  "type": "module"
}
```

Use ES6 imports:
```typescript
import { createApp } from 'ramapi';  // ✅ Correct
const { createApp } = require('ramapi');  // ❌ Won't work
```

---

## Server Issues

### Issue: Port already in use

**Error:**
```
Error: listen EADDRINUSE: address already in use :::3000
```

**Solutions:**

1. Kill process on port:
```bash
# macOS/Linux
lsof -ti:3000 | xargs kill -9

# Windows
netstat -ano | findstr :3000
taskkill /PID <PID> /F
```

2. Use different port:
```typescript
app.listen(3001);
```

3. Set port from environment:
```typescript
const port = parseInt(process.env.PORT || '3000');
app.listen(port);
```

---

### Issue: Server starts but requests hang

**Symptoms:**
- Server starts successfully
- Requests never complete
- No errors in console

**Common Causes:**

1. **Missing `await next()` in middleware:**
```typescript
// ❌ Wrong
app.use(async (ctx, next) => {
  console.log('Before');
  next();  // Missing await!
});

// ✅ Correct
app.use(async (ctx, next) => {
  console.log('Before');
  await next();
});
```

2. **Not calling next() at all:**
```typescript
// ❌ Wrong - request hangs
app.use(async (ctx, next) => {
  console.log('Logging...');
  // Missing next()!
});

// ✅ Correct
app.use(async (ctx, next) => {
  console.log('Logging...');
  await next();
});
```

3. **Not sending response:**
```typescript
// ❌ Wrong - no response sent
app.get('/users', async (ctx) => {
  const users = await getUsers();
  // Missing ctx.json()!
});

// ✅ Correct
app.get('/users', async (ctx) => {
  const users = await getUsers();
  ctx.json({ users });
});
```

---

## Routing Problems

### Issue: 404 Not Found

**Error:**
```
404 Not Found
```

**Solutions:**

1. **Check route path exactly matches:**
```typescript
app.get('/api/users', handler);  // Must request exactly /api/users
```

2. **Check HTTP method:**
```typescript
app.post('/users', handler);  // Won't match GET /users
```

3. **Check route order:**
```typescript
// ❌ Wrong order
app.get('/users/:id', getUser);
app.get('/users/me', getMe);  // Never matches! :id catches it

// ✅ Correct order
app.get('/users/me', getMe);  // Specific routes first
app.get('/users/:id', getUser);
```

4. **Check if middleware blocks request:**
```typescript
// Middleware that stops request
app.use(async (ctx, next) => {
  if (someCondition) {
    ctx.status(403);
    ctx.json({ error: 'Forbidden' });
    return;  // Stops here, never reaches routes
  }
  await next();
});
```

---

### Issue: Route parameters not working

**Error:**
```typescript
console.log(ctx.params.id);  // undefined
```

**Solutions:**

1. **Use correct parameter syntax:**
```typescript
// ✅ Correct
app.get('/users/:id', handler);  // Use colon (:)

// ❌ Wrong
app.get('/users/{id}', handler);  // Don't use braces
app.get('/users/$id', handler);   // Don't use $
```

2. **Check parameter name matches:**
```typescript
app.get('/users/:userId', (ctx) => {
  console.log(ctx.params.userId);  // Use same name
});
```

---

## Middleware Issues

### Issue: Middleware not executing

**Symptoms:**
- Middleware defined but not running
- console.log in middleware doesn't print

**Solutions:**

1. **Register middleware before routes:**
```typescript
// ✅ Correct order
app.use(myMiddleware);
app.get('/users', handler);

// ❌ Wrong order
app.get('/users', handler);
app.use(myMiddleware);  // Too late!
```

2. **Check middleware returns properly:**
```typescript
// ❌ Wrong
function myMiddleware() {
  return (ctx: Context, next: () => Promise<void>) => {
    // ...
  };
}
app.use(myMiddleware);  // Missing ()!

// ✅ Correct
app.use(myMiddleware());  // Call the function
```

---

### Issue: Validation not working

**Error:**
```
Request body not validated
```

**Solutions:**

1. **Ensure validate middleware is registered:**
```typescript
import { validate } from 'ramapi';  // ✅ Import first

const schema = z.object({ name: z.string() });
app.post('/users', validate({ body: schema }), handler);
```

2. **Check Content-Type header:**
```bash
# ✅ Correct
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John"}'

# ❌ Missing Content-Type
curl -X POST http://localhost:3000/api/users \
  -d '{"name":"John"}'
```

3. **Verify schema structure:**
```typescript
// ✅ Correct - object wrapper
validate({ body: userSchema })

// ❌ Wrong - missing wrapper
validate(userSchema)
```

---

## Authentication Problems

### Issue: "Authorization header missing"

**Error:**
```json
{ "error": "Authorization header missing" }
```

**Solutions:**

1. **Include Authorization header:**
```bash
curl http://localhost:3000/api/protected \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

2. **Check header format:**
```bash
# ✅ Correct format
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# ❌ Wrong formats
Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...  # Missing "Bearer"
Authorization: Token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...  # Wrong prefix
```

---

### Issue: "Invalid token"

**Error:**
```json
{ "error": "Invalid token" }
```

**Solutions:**

1. **Check JWT secret matches:**
```typescript
// Server
const jwtService = new JWTService({ secret: 'secret123' });

// Must use same secret!
const token = jwtService.sign({ sub: 'user-id' });
```

2. **Check token not expired:**
```typescript
const jwtService = new JWTService({
  secret: 'secret',
  expiresIn: 3600,  // 1 hour
});

// Token expires after 1 hour!
```

3. **Verify token format:**
```typescript
// ✅ Valid JWT format
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U

// ❌ Invalid - missing parts
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

---

## Observability Issues

### Issue: Tracing not working

**Symptoms:**
- No trace output in console
- Jaeger shows no traces

**Solutions:**

1. **Enable tracing in config:**
```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,  // Must be true!
      exporter: 'console',
    },
  },
});
```

2. **Check sample rate:**
```typescript
tracing: {
  enabled: true,
  sampleRate: 1.0,  // Must be > 0
}
```

3. **Verify Jaeger is running:**
```bash
docker ps | grep jaeger
curl http://localhost:14268/api/traces
```

---

### Issue: Flow tracking returns 404

**Error:**
```
GET /profile/123/waterfall -> 404 Not Found
```

**Solution:**

Flow tracking requires tracing to be enabled:
```typescript
const app = createApp({
  observability: {
    tracing: {
      enabled: true,  // Required!
    },
  },
});

app.use(flowTrackingMiddleware());
registerFlowRoutes(app as any);
```

---

## Performance Problems

### Issue: High memory usage

**Symptoms:**
- Memory usage keeps growing
- Server crashes with out of memory

**Solutions:**

1. **Reduce trace sample rate:**
```typescript
tracing: {
  sampleRate: 0.1,  // Sample only 10%
}
```

2. **Disable memory profiling:**
```typescript
profiling: {
  enabled: true,
  captureMemory: false,
}
```

3. **Check for memory leaks:**
```bash
node --inspect dist/index.js
# Open chrome://inspect
# Take heap snapshots
```

---

### Issue: Slow responses

**Symptoms:**
- Requests take longer than expected
- High latency

**Solutions:**

1. **Enable profiling to identify bottlenecks:**
```typescript
observability: {
  profiling: {
    enabled: true,
    slowThreshold: 100,  // Log requests > 100ms
  },
}
```

2. **Check database queries:**
```typescript
// Add query timing
const start = Date.now();
const users = await db.query('SELECT * FROM users');
console.log(`Query took ${Date.now() - start}ms`);
```

3. **Add caching:**
```typescript
import Redis from 'ioredis';
const redis = new Redis();

app.get('/users', async (ctx) => {
  const cached = await redis.get('users');
  if (cached) {
    ctx.json(JSON.parse(cached));
    return;
  }

  const users = await db.query('SELECT * FROM users');
  await redis.setex('users', 60, JSON.stringify(users));
  ctx.json(users);
});
```

---

## TypeScript Errors

### Issue: Type errors with Context

**Error:**
```
Property 'user' does not exist on type 'Context'
```

**Solution:**

Extend Context type:
```typescript
declare module 'ramapi' {
  interface Context {
    user?: {
      id: string;
      email: string;
    };
  }
}

// Now ctx.user is typed
app.get('/profile', auth, (ctx) => {
  console.log(ctx.user?.email);  // ✅ Type-safe
});
```

---

### Issue: "Cannot find module" in TypeScript

**Error:**
```
Cannot find module 'ramapi' or its corresponding type declarations
```

**Solutions:**

1. **Install @types if needed:**
```bash
npm install -D @types/node
```

2. **Check tsconfig.json:**
```json
{
  "compilerOptions": {
    "moduleResolution": "node",
    "esModuleInterop": true
  }
}
```

3. **Rebuild:**
```bash
rm -rf dist
npm run build
```

---

## FAQ

### Q: How do I debug my RamAPI app?

**A:** Use VS Code debugger or Node inspector:

**VS Code launch.json:**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug RamAPI",
      "program": "${workspaceFolder}/src/index.ts",
      "preLaunchTask": "tsc: build - tsconfig.json",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"]
    }
  ]
}
```

Or use node inspect:
```bash
node --inspect-brk dist/index.js
# Open chrome://inspect
```

---

### Q: How do I handle file uploads?

**A:** Use multipart/form-data parser:

```bash
npm install formidable
npm install -D @types/formidable
```

```typescript
import formidable from 'formidable';

app.post('/upload', async (ctx) => {
  const form = formidable({ multiples: true });

  const [fields, files] = await new Promise((resolve, reject) => {
    form.parse(ctx.req, (err, fields, files) => {
      if (err) reject(err);
      else resolve([fields, files]);
    });
  });

  ctx.json({ files });
});
```

---

### Q: Can I use RamAPI with existing middleware?

**A:** Only RamAPI middleware works directly. Convert Express/Koa middleware:

```typescript
// Express middleware
function expressMiddleware(req, res, next) {
  // ...
}

// Convert to RamAPI
function ramApiMiddleware(ctx: Context, next: () => Promise<void>) {
  return new Promise((resolve, reject) => {
    expressMiddleware(ctx.req, ctx.res, (err) => {
      if (err) reject(err);
      else next().then(resolve).catch(reject);
    });
  });
}
```

---

### Q: How do I serve static files?

**A:** Use a dedicated static file server or middleware:

```typescript
import { readFile } from 'fs/promises';
import { join } from 'path';

app.get('/static/*', async (ctx) => {
  const filePath = ctx.path.replace('/static/', '');
  const fullPath = join(process.cwd(), 'public', filePath);

  try {
    const content = await readFile(fullPath);
    ctx.send(content);
  } catch {
    ctx.status(404);
    ctx.text('Not found');
  }
});
```

Or use nginx for production static file serving.

---

### Q: How do I handle WebSockets?

**A:** RamAPI focuses on HTTP APIs. Use a separate WebSocket library:

```typescript
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  ws.on('message', (data) => {
    ws.send(`Echo: ${data}`);
  });
});

// Run alongside RamAPI
app.listen(3000);  // HTTP API
// WebSocket on 8080
```

---

### Q: How do I test my API?

**A:** See [Testing Strategies](../advanced/testing.md) guide.

Quick example:
```typescript
import { describe, it, expect } from 'vitest';

describe('API', () => {
  it('should return users', async () => {
    const response = await fetch('http://localhost:3000/api/users');
    const data = await response.json();
    expect(data.users).toBeDefined();
  });
});
```

---

### Q: Production checklist?

**A:** See [Production Setup](../deployment/production-setup.md).

Quick checklist:
- [ ] Set NODE_ENV=production
- [ ] Use strong JWT secrets
- [ ] Enable HTTPS
- [ ] Add rate limiting
- [ ] Set up logging
- [ ] Configure error handling
- [ ] Add health checks
- [ ] Enable observability
- [ ] Set up monitoring
- [ ] Test error scenarios

---

## Getting Help

- **Documentation**: Browse complete docs in `complete-documentation/`
- **Examples**: Check `examples/` directory for working code
- **Issues**: Report bugs on GitHub
- **Verified APIs**: All guides marked with ✅ are verified against source code

---

## See Also

- [Quick Start](../getting-started/quick-start.md)
- [Building REST API](building-rest-api.md)
- [Security Best Practices](../advanced/security.md)
- [Performance Optimization](../performance/optimization.md)
