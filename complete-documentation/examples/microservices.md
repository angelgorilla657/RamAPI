# Microservices Example

Distributed microservices architecture with distributed tracing, service discovery, and inter-service communication.

> **Note:** This is a documentation example showing how to use RamAPI in your own project. The code assumes you have RamAPI installed via npm.

## Prerequisites

```bash
# Install RamAPI (in each service directory)
npm install ramapi

# Install dependencies
npm install zod better-sqlite3
npm install -D @types/better-sqlite3

# Start OpenTelemetry Collector (for distributed tracing)
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

## Overview

This example demonstrates:
- Multiple microservices (User Service, Order Service, Payment Service)
- Distributed tracing with OpenTelemetry
- Inter-service communication (HTTP)
- Error handling across services
- Observability and monitoring

## Architecture

```
┌─────────────┐
│   Gateway   │  :3000
└──────┬──────┘
       │
   ┌───┴────┬─────────┬──────────┐
   │        │         │          │
┌──▼───┐ ┌──▼────┐ ┌──▼─────┐ ┌──▼────────┐
│ User │ │ Order │ │ Product│ │  Payment  │
│:3001 │ │ :3002 │ │ :3003  │ │   :3004   │
└──────┘ └───────┘ └────────┘ └───────────┘
```

## User Service (Port 3001)

```typescript
import { createApp, validate, logger } from 'ramapi';
import { z } from 'zod';
import Database from 'better-sqlite3';

const db = new Database('users.db');
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TEXT NOT NULL
  )
`);

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'user-service',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
    logging: { enabled: true, level: 'info', format: 'json' },
    metrics: { enabled: true },
  },
});

app.use(logger());

const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

app.get('/users', async (ctx) => {
  const users = db.prepare('SELECT * FROM users').all();
  ctx.json({ users });
});

app.get('/users/:id', async (ctx) => {
  const user = db.prepare('SELECT * FROM users WHERE id = ?').get(ctx.params.id);
  if (!user) {
    ctx.json({ error: 'User not found' }, 404);
    return;
  }
  ctx.json({ user });
});

app.post('/users', validate({ body: userSchema }), async (ctx) => {
  const id = `user-${Date.now()}`;
  const created_at = new Date().toISOString();

  db.prepare('INSERT INTO users (id, name, email, created_at) VALUES (?, ?, ?, ?)')
    .run(id, ctx.body.name, ctx.body.email, created_at);

  const user = db.prepare('SELECT * FROM users WHERE id = ?').get(id);
  ctx.json({ user }, 201);
});

await app.listen(3001);
console.log('👤 User Service running on http://localhost:3001');
```

## Order Service (Port 3002)

```typescript
import { createApp, validate, logger } from 'ramapi';
import { z } from 'zod';
import Database from 'better-sqlite3';

const db = new Database('orders.db');
db.exec(`
  CREATE TABLE IF NOT EXISTS orders (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    total REAL NOT NULL,
    status TEXT DEFAULT 'pending',
    created_at TEXT NOT NULL
  )
`);

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'order-service',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
    logging: { enabled: true, level: 'info', format: 'json' },
  },
});

app.use(logger());

// Helper: Call other services
async function callService(url: string) {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`Service call failed: ${response.statusText}`);
  return response.json();
}

const orderSchema = z.object({
  userId: z.string(),
  productId: z.string(),
  quantity: z.number().int().positive(),
});

app.post('/orders', validate({ body: orderSchema }), async (ctx) => {
  const { userId, productId, quantity } = ctx.body;

  // Start distributed trace
  const orderSpan = ctx.startSpan?.('order.create');

  try {
    // Verify user exists (call User Service)
    const userSpan = ctx.startSpan?.('user.fetch', { 'user.id': userId });
    const { user } = await callService(`http://localhost:3001/users/${userId}`);
    ctx.endSpan?.(userSpan);

    // Verify product exists and get price (call Product Service)
    const productSpan = ctx.startSpan?.('product.fetch', { 'product.id': productId });
    const { product } = await callService(`http://localhost:3003/products/${productId}`);
    ctx.endSpan?.(productSpan);

    // Calculate total
    const total = product.price * quantity;

    // Process payment (call Payment Service)
    const paymentSpan = ctx.startSpan?.('payment.process', { amount: total });
    const { payment } = await callService('http://localhost:3004/payments', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, amount: total }),
    } as any);
    ctx.endSpan?.(paymentSpan);

    // Create order
    const id = `order-${Date.now()}`;
    const created_at = new Date().toISOString();

    db.prepare(`
      INSERT INTO orders (id, user_id, product_id, quantity, total, status, created_at)
      VALUES (?, ?, ?, ?, ?, 'completed', ?)
    `).run(id, userId, productId, quantity, total, created_at);

    const order = db.prepare('SELECT * FROM orders WHERE id = ?').get(id);

    ctx.endSpan?.(orderSpan);
    ctx.json({ order }, 201);
  } catch (error: any) {
    ctx.endSpan?.(orderSpan, error);
    ctx.json({ error: error.message }, 400);
  }
});

app.get('/orders', async (ctx) => {
  const orders = db.prepare('SELECT * FROM orders').all();
  ctx.json({ orders });
});

app.get('/orders/:id', async (ctx) => {
  const order = db.prepare('SELECT * FROM orders WHERE id = ?').get(ctx.params.id);
  if (!order) {
    ctx.json({ error: 'Order not found' }, 404);
    return;
  }
  ctx.json({ order });
});

await app.listen(3002);
console.log('📦 Order Service running on http://localhost:3002');
```

## Product Service (Port 3003)

```typescript
import { createApp, validate, logger } from 'ramapi';
import { z } from 'zod';
import Database from 'better-sqlite3';

const db = new Database('products.db');
db.exec(`
  CREATE TABLE IF NOT EXISTS products (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL NOT NULL,
    stock INTEGER NOT NULL,
    created_at TEXT NOT NULL
  )
`);

// Seed data
db.prepare(`
  INSERT OR IGNORE INTO products (id, name, price, stock, created_at)
  VALUES ('prod-1', 'Laptop', 999.99, 10, '2024-01-01T00:00:00.000Z')
`).run();

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'product-service',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
  },
});

app.use(logger());

app.get('/products', async (ctx) => {
  const products = db.prepare('SELECT * FROM products').all();
  ctx.json({ products });
});

app.get('/products/:id', async (ctx) => {
  const product = db.prepare('SELECT * FROM products WHERE id = ?').get(ctx.params.id);
  if (!product) {
    ctx.json({ error: 'Product not found' }, 404);
    return;
  }
  ctx.json({ product });
});

await app.listen(3003);
console.log('🛍️  Product Service running on http://localhost:3003');
```

## Payment Service (Port 3004)

```typescript
import { createApp, validate, logger } from 'ramapi';
import { z } from 'zod';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'payment-service',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
  },
});

app.use(logger());

const paymentSchema = z.object({
  userId: z.string(),
  amount: z.number().positive(),
});

app.post('/payments', validate({ body: paymentSchema }), async (ctx) => {
  const { userId, amount } = ctx.body;

  // Simulate payment processing
  const paymentSpan = ctx.startSpan?.('payment.process.stripe');

  await new Promise((resolve) => setTimeout(resolve, 100)); // Simulate API call

  ctx.endSpan?.(paymentSpan);

  const payment = {
    id: `pay-${Date.now()}`,
    userId,
    amount,
    status: 'success',
    timestamp: new Date().toISOString(),
  };

  ctx.json({ payment }, 201);
});

await app.listen(3004);
console.log('💳 Payment Service running on http://localhost:3004');
```

## API Gateway (Port 3000)

```typescript
import { createApp, logger } from 'ramapi';

const app = createApp({
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'api-gateway',
      exporter: 'otlp',
      endpoint: 'http://localhost:4318',
    },
  },
});

app.use(logger());

// Proxy to services
app.all('/api/users/*', async (ctx) => {
  const path = ctx.path.replace('/api/users', '/users');
  const response = await fetch(`http://localhost:3001${path}`);
  const data = await response.json();
  ctx.json(data, response.status);
});

app.all('/api/orders/*', async (ctx) => {
  const path = ctx.path.replace('/api/orders', '/orders');
  const response = await fetch(`http://localhost:3002${path}`, {
    method: ctx.method,
    headers: ctx.headers as any,
    body: ctx.method !== 'GET' ? JSON.stringify(ctx.body) : undefined,
  });
  const data = await response.json();
  ctx.json(data, response.status);
});

await app.listen(3000);
console.log('🌐 API Gateway running on http://localhost:3000');
```

## Usage

```bash
# Create user
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'

# Create order (triggers User, Product, Payment services)
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":"user-1","productId":"prod-1","quantity":1}'
```

## Distributed Tracing

View traces in Jaeger:
```bash
# Start Jaeger
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# Open UI
open http://localhost:16686
```

## See Also

- [Observability Guide](../observability/overview.md)
- [Distributed Tracing](../observability/tracing.md)
- [Multi-Protocol Example](multi-protocol.md)
