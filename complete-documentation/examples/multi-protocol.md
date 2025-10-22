# Multi-Protocol Example

Single service serving REST, GraphQL, and gRPC from the same codebase with shared business logic.

> **Note:** This is a documentation example showing how to use RamAPI in your own project. The code assumes you have RamAPI installed via npm.

## Prerequisites

```bash
# Install RamAPI
npm install ramapi

# Install dependencies
npm install graphql @grpc/grpc-js @grpc/proto-loader zod better-sqlite3
npm install -D @types/better-sqlite3
```

## Overview

This example demonstrates:
- Unified business logic layer
- REST, GraphQL, and gRPC endpoints
- Shared authentication across protocols
- Distributed tracing across all protocols
- Single database for all protocols

## Complete Code

```typescript
import { createApp, JWTService, validate, logger } from 'ramapi';
import { buildSchema } from 'graphql';
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import Database from 'better-sqlite3';
import { z } from 'zod';

// Database
const db = new Database('multi-protocol.db');
db.exec(`
  CREATE TABLE IF NOT EXISTS products (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL NOT NULL,
    stock INTEGER NOT NULL,
    created_at TEXT NOT NULL
  )
`);

// Business Logic Layer (shared across all protocols)
class ProductService {
  getAll() {
    return db.prepare('SELECT * FROM products').all();
  }

  getById(id: string) {
    return db.prepare('SELECT * FROM products WHERE id = ?').get(id);
  }

  create(name: string, price: number, stock: number) {
    const id = `prod-${Date.now()}`;
    const created_at = new Date().toISOString();

    db.prepare(`
      INSERT INTO products (id, name, price, stock, created_at)
      VALUES (?, ?, ?, ?, ?)
    `).run(id, name, price, stock, created_at);

    return this.getById(id);
  }

  update(id: string, data: any) {
    const updates: string[] = [];
    const values: any[] = [];

    if (data.name) { updates.push('name = ?'); values.push(data.name); }
    if (data.price !== undefined) { updates.push('price = ?'); values.push(data.price); }
    if (data.stock !== undefined) { updates.push('stock = ?'); values.push(data.stock); }

    if (updates.length > 0) {
      values.push(id);
      db.prepare(`UPDATE products SET ${updates.join(', ')} WHERE id = ?`).run(...values);
    }

    return this.getById(id);
  }

  delete(id: string) {
    const result = db.prepare('DELETE FROM products WHERE id = ?').run(id);
    return result.changes > 0;
  }
}

const productService = new ProductService();

// JWT Service (shared authentication)
const jwtService = new JWTService({
  secret: process.env.JWT_SECRET || 'your-secret',
  expiresIn: 86400,
});

// GraphQL Schema
const graphqlSchema = buildSchema(`
  type Product {
    id: ID!
    name: String!
    price: Float!
    stock: Int!
    createdAt: String!
  }

  type Query {
    products: [Product!]!
    product(id: ID!): Product
  }

  type Mutation {
    createProduct(name: String!, price: Float!, stock: Int!): Product!
    updateProduct(id: ID!, name: String, price: Float, stock: Int): Product!
    deleteProduct(id: ID!): Boolean!
  }
`);

const graphqlRoot = {
  products: () => productService.getAll(),
  product: ({ id }: any) => productService.getById(id),
  createProduct: ({ name, price, stock }: any) => productService.create(name, price, stock),
  updateProduct: ({ id, name, price, stock }: any) => productService.update(id, { name, price, stock }),
  deleteProduct: ({ id }: any) => productService.delete(id),
};

// gRPC Proto
const packageDefinition = protoLoader.loadSync('product.proto');
const proto = grpc.loadPackageDefinition(packageDefinition) as any;

const grpcService = {
  GetProducts: (call: any, callback: any) => {
    const products = productService.getAll();
    callback(null, { products });
  },

  GetProduct: (call: any, callback: any) => {
    const product = productService.getById(call.request.id);
    if (!product) {
      callback({ code: grpc.status.NOT_FOUND, message: 'Product not found' });
      return;
    }
    callback(null, product);
  },

  CreateProduct: (call: any, callback: any) => {
    const { name, price, stock } = call.request;
    const product = productService.create(name, price, stock);
    callback(null, product);
  },

  UpdateProduct: (call: any, callback: any) => {
    const { id, name, price, stock } = call.request;
    const product = productService.update(id, { name, price, stock });
    if (!product) {
      callback({ code: grpc.status.NOT_FOUND, message: 'Product not found' });
      return;
    }
    callback(null, product);
  },

  DeleteProduct: (call: any, callback: any) => {
    const success = productService.delete(call.request.id);
    callback(null, { success });
  },
};

// Create Multi-Protocol Server
const app = createApp({
  adapter: { type: 'node-http' }, // Required for gRPC
  protocols: {
    graphql: {
      path: '/graphql',
      schema: graphqlSchema,
      rootValue: graphqlRoot,
      playground: true,
    },
    grpc: {
      port: 50051,
      services: {
        'product.ProductService': grpcService,
      },
    },
  },
  observability: {
    tracing: {
      enabled: true,
      serviceName: 'multi-protocol-api',
      exporter: 'console',
    },
    logging: {
      enabled: true,
      level: 'info',
      format: 'json',
    },
  },
});

app.use(logger());

// REST API Routes
const productSchema = z.object({
  name: z.string().min(1),
  price: z.number().positive(),
  stock: z.number().int().min(0),
});

app.get('/api/products', async (ctx) => {
  const products = productService.getAll();
  ctx.json({ products });
});

app.get('/api/products/:id', async (ctx) => {
  const product = productService.getById(ctx.params.id);
  if (!product) {
    ctx.json({ error: 'Product not found' }, 404);
    return;
  }
  ctx.json({ product });
});

app.post('/api/products',
  validate({ body: productSchema }),
  async (ctx) => {
    const { name, price, stock } = ctx.body;
    const product = productService.create(name, price, stock);
    ctx.json({ product }, 201);
  }
);

app.put('/api/products/:id',
  validate({ body: productSchema.partial() }),
  async (ctx) => {
    const product = productService.update(ctx.params.id, ctx.body);
    if (!product) {
      ctx.json({ error: 'Product not found' }, 404);
      return;
    }
    ctx.json({ product });
  }
);

app.delete('/api/products/:id', async (ctx) => {
  const success = productService.delete(ctx.params.id);
  if (!success) {
    ctx.json({ error: 'Product not found' }, 404);
    return;
  }
  ctx.status(204);
});

await app.listen(3000);

console.log('🚀 Multi-Protocol API Running:');
console.log('   REST API:    http://localhost:3000/api');
console.log('   GraphQL:     http://localhost:3000/graphql');
console.log('   gRPC:        localhost:50051');
console.log('\nAll protocols use the same business logic and database!');
```

## product.proto

```protobuf
syntax = "proto3";
package product;

service ProductService {
  rpc GetProducts (Empty) returns (ProductList);
  rpc GetProduct (GetProductRequest) returns (Product);
  rpc CreateProduct (CreateProductRequest) returns (Product);
  rpc UpdateProduct (UpdateProductRequest) returns (Product);
  rpc DeleteProduct (DeleteProductRequest) returns (DeleteResponse);
}

message Empty {}

message Product {
  string id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
  string created_at = 5;
}

message ProductList {
  repeated Product products = 1;
}

message GetProductRequest {
  string id = 1;
}

message CreateProductRequest {
  string name = 1;
  double price = 2;
  int32 stock = 3;
}

message UpdateProductRequest {
  string id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
}

message DeleteProductRequest {
  string id = 1;
}

message DeleteResponse {
  bool success = 1;
}
```

## Usage Examples

### REST

```bash
# Create product
curl -X POST http://localhost:3000/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","price":999.99,"stock":10}'

# List products
curl http://localhost:3000/api/products
```

### GraphQL

```graphql
# Create product
mutation {
  createProduct(name: "Laptop", price: 999.99, stock: 10) {
    id
    name
    price
  }
}

# List products
query {
  products {
    id
    name
    price
    stock
  }
}
```

### gRPC (Client)

```typescript
const client = new proto.product.ProductService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Create product
client.CreateProduct(
  { name: 'Laptop', price: 999.99, stock: 10 },
  (error, response) => {
    console.log('Created:', response);
  }
);
```

## Benefits

1. **Single Codebase** - One business logic layer
2. **Protocol Flexibility** - Clients choose REST, GraphQL, or gRPC
3. **Shared Database** - Consistent data across protocols
4. **Unified Observability** - Traces across all protocols
5. **Easy Maintenance** - Update logic once, affects all protocols

## See Also

- [Multi-Protocol Guide](../multi-protocol/overview.md)
- [GraphQL Example](graphql-api.md)
- [gRPC Example](grpc-service.md)
