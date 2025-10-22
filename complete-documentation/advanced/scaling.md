# Scaling & Architecture

Comprehensive guide to scaling RamAPI applications: horizontal scaling, load balancing, caching strategies, and architecture patterns for high-performance systems.

> **Note**: This documentation covers production-ready scaling patterns and architecture strategies:
> - ✅ RamAPI server methods verified (listen(), close())
> - ✅ Context structure verified for stateless patterns
> - ✅ All scaling patterns are industry-standard Node.js best practices
> - ✅ nginx/HAProxy configurations are production-tested patterns
> - ✅ Redis, database pooling, and caching patterns are standard implementations
> - ⚠️ Architecture patterns (microservices, CQRS, event-driven) are conceptual examples

## Table of Contents

1. [Scaling Fundamentals](#scaling-fundamentals)
2. [Horizontal Scaling](#horizontal-scaling)
3. [Load Balancing](#load-balancing)
4. [Caching Strategies](#caching-strategies)
5. [Database Optimization](#database-optimization)
6. [Architecture Patterns](#architecture-patterns)
7. [Performance Monitoring](#performance-monitoring)
8. [Capacity Planning](#capacity-planning)

---

## Scaling Fundamentals

### Vertical vs Horizontal Scaling

```
┌─────────────────────────┐
│  Vertical Scaling       │
│  (Scale Up)             │
├─────────────────────────┤
│  Single Server          │
│  - More CPU             │
│  - More RAM             │
│  - Faster disk          │
│                         │
│  Pros:                  │
│  ✓ Simpler             │
│  ✓ No code changes     │
│                         │
│  Cons:                  │
│  ✗ Hardware limits     │
│  ✗ Single point failure│
│  ✗ Expensive           │
└─────────────────────────┘

┌─────────────────────────┐
│  Horizontal Scaling     │
│  (Scale Out)            │
├─────────────────────────┤
│  Multiple Servers       │
│  ┌─────┐ ┌─────┐       │
│  │ S1  │ │ S2  │       │
│  └─────┘ └─────┘       │
│  ┌─────┐ ┌─────┐       │
│  │ S3  │ │ S4  │       │
│  └─────┘ └─────┘       │
│                         │
│  Pros:                  │
│  ✓ No limits           │
│  ✓ Fault tolerant      │
│  ✓ Cost effective      │
│                         │
│  Cons:                  │
│  ✗ More complex        │
│  ✗ Stateless required  │
└─────────────────────────┘
```

### Scaling Checklist

Before scaling horizontally:

- [ ] Application is stateless
- [ ] Sessions stored externally (Redis, database)
- [ ] File uploads go to object storage (S3, GCS)
- [ ] Database connections properly pooled
- [ ] Health checks implemented
- [ ] Logging centralized
- [ ] Metrics collected

---

## Horizontal Scaling

### Making Your App Stateless

```typescript
import { createApp } from 'ramapi';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const app = createApp();

// BAD: In-memory state (not scalable)
const sessions = new Map<string, any>();

app.post('/login', async (ctx) => {
  const token = generateToken();
  sessions.set(token, { userId: '123' }); // Stored in memory!
  ctx.json({ token });
});

// GOOD: External state (scalable)
app.post('/login', async (ctx) => {
  const token = generateToken();
  await redis.set(`session:${token}`, JSON.stringify({ userId: '123' }), 'EX', 3600);
  ctx.json({ token });
});

// GOOD: Stateless with JWT
import { JWTService } from 'ramapi';

const jwtService = new JWTService({ secret: process.env.JWT_SECRET! });

app.post('/login', async (ctx) => {
  const token = jwtService.sign({ sub: '123' }); // No server state!
  ctx.json({ token });
});
```

### Process Management with PM2

```bash
# Install PM2
npm install -g pm2

# Start with cluster mode
pm2 start dist/index.js -i max --name ramapi

# Or use ecosystem file
pm2 start ecosystem.config.js
```

**ecosystem.config.js:**

```javascript
module.exports = {
  apps: [{
    name: 'ramapi',
    script: './dist/index.js',
    instances: 'max', // Use all CPU cores
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
    error_file: './logs/error.log',
    out_file: './logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
  }],
};
```

### Node.js Cluster Module

```typescript
import cluster from 'cluster';
import { cpus } from 'os';
import { createApp } from 'ramapi';

const numCPUs = cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    // Restart worker
    cluster.fork();
  });
} else {
  // Workers share TCP connection
  const app = createApp();

  app.get('/', (ctx) => {
    ctx.json({ pid: process.pid, message: 'Hello' });
  });

  app.listen(3000);
  console.log(`Worker ${process.pid} started`);
}
```

---

## Load Balancing

### nginx Load Balancer

**nginx.conf:**

```nginx
upstream ramapi {
    # Load balancing method
    least_conn; # or: round_robin, ip_hash

    # Backend servers
    server 127.0.0.1:3000 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3001 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3003 weight=1 max_fails=3 fail_timeout=30s;

    # Health check
    keepalive 32;
}

server {
    listen 80;
    server_name api.example.com;

    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/ssl/certs/api.example.com.crt;
    ssl_certificate_key /etc/ssl/private/api.example.com.key;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    # Buffer sizes
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;
    proxy_busy_buffers_size 8k;

    location / {
        proxy_pass http://ramapi;
        proxy_http_version 1.1;

        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Disable buffering for SSE
        proxy_buffering off;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://ramapi/health;
        access_log off;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
    limit_req zone=api_limit burst=200 nodelay;
}
```

### HAProxy Load Balancer

**haproxy.cfg:**

```cfg
global
    maxconn 4096
    log /dev/log local0
    log /dev/log local1 notice

defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http_front
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/api.example.com.pem
    default_backend ramapi_back

    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny if { sc_http_req_rate(0) gt 100 }

backend ramapi_back
    balance leastconn
    option httpchk GET /health

    # Backend servers
    server server1 127.0.0.1:3000 check inter 2000 rise 2 fall 3
    server server2 127.0.0.1:3001 check inter 2000 rise 2 fall 3
    server server3 127.0.0.1:3002 check inter 2000 rise 2 fall 3
    server server4 127.0.0.1:3003 check inter 2000 rise 2 fall 3

# Stats page
listen stats
    bind *:8080
    stats enable
    stats uri /stats
    stats auth admin:password
```

---

## Caching Strategies

### Response Caching with Redis

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// Cache middleware
function cache(ttl: number = 60) {
  return async (ctx: Context, next: () => Promise<void>) => {
    // Only cache GET requests
    if (ctx.method !== 'GET') {
      await next();
      return;
    }

    const key = `cache:${ctx.path}`;

    // Check cache
    const cached = await redis.get(key);
    if (cached) {
      ctx.setHeader('X-Cache', 'HIT');
      ctx.json(JSON.parse(cached));
      return;
    }

    // Execute handler
    await next();

    // Cache response
    if (ctx.statusCode === 200 && ctx.responseBody) {
      await redis.setex(key, ttl, ctx.responseBody);
      ctx.setHeader('X-Cache', 'MISS');
    }
  };
}

// Usage
app.get('/users', cache(300), async (ctx) => {
  const users = await db.query('SELECT * FROM users');
  ctx.json({ users });
});

// Cache with query parameters
function cacheWithQuery(ttl: number = 60) {
  return async (ctx: Context, next: () => Promise<void>) => {
    const url = new URL(ctx.path, `http://${ctx.headers.host}`);
    const key = `cache:${ctx.path}:${url.searchParams.toString()}`;

    const cached = await redis.get(key);
    if (cached) {
      ctx.json(JSON.parse(cached));
      return;
    }

    await next();

    if (ctx.statusCode === 200) {
      await redis.setex(key, ttl, ctx.responseBody!);
    }
  };
}
```

### Cache Invalidation

```typescript
// Invalidate on write operations
app.post('/users', async (ctx) => {
  const user = await createUser(ctx.body);

  // Invalidate cache
  await redis.del('cache:/users');

  ctx.json(user, 201);
});

app.put('/users/:id', async (ctx) => {
  const user = await updateUser(ctx.params.id, ctx.body);

  // Invalidate specific user and list
  await redis.del(`cache:/users/${ctx.params.id}`, 'cache:/users');

  ctx.json(user);
});

// Pattern-based invalidation
async function invalidatePattern(pattern: string) {
  const keys = await redis.keys(pattern);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}

app.post('/posts', async (ctx) => {
  const post = await createPost(ctx.body);

  // Invalidate all post-related caches
  await invalidatePattern('cache:/posts*');

  ctx.json(post, 201);
});
```

### HTTP Caching Headers

```typescript
// ETag caching
import { createHash } from 'crypto';

app.get('/users/:id', async (ctx) => {
  const user = await getUser(ctx.params.id);

  // Generate ETag
  const etag = createHash('md5')
    .update(JSON.stringify(user))
    .digest('hex');

  // Check if client has cached version
  const clientETag = ctx.headers['if-none-match'];
  if (clientETag === etag) {
    ctx.status(304); // Not Modified
    return;
  }

  // Set caching headers
  ctx.setHeader('ETag', etag);
  ctx.setHeader('Cache-Control', 'private, max-age=300'); // 5 minutes
  ctx.json(user);
});

// Last-Modified caching
app.get('/posts/:id', async (ctx) => {
  const post = await getPost(ctx.params.id);

  const lastModified = new Date(post.updatedAt).toUTCString();
  const clientLastModified = ctx.headers['if-modified-since'];

  if (clientLastModified === lastModified) {
    ctx.status(304);
    return;
  }

  ctx.setHeader('Last-Modified', lastModified);
  ctx.setHeader('Cache-Control', 'public, max-age=600'); // 10 minutes
  ctx.json(post);
});
```

---

## Database Optimization

### Connection Pooling

```typescript
import { Pool } from 'pg';

// Configure connection pool
const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  // Pool configuration
  min: 2,                    // Minimum connections
  max: 10,                   // Maximum connections
  idleTimeoutMillis: 30000,  // Close idle connections after 30s
  connectionTimeoutMillis: 2000, // Timeout if can't get connection
});

// Use pool
app.get('/users', async (ctx) => {
  const client = await pool.connect();

  try {
    const result = await client.query('SELECT * FROM users');
    ctx.json({ users: result.rows });
  } finally {
    client.release(); // Return to pool
  }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await pool.end();
  process.exit(0);
});
```

### Query Optimization

```typescript
// Use indexes
// CREATE INDEX idx_users_email ON users(email);
// CREATE INDEX idx_posts_author_id ON posts(author_id);

// Limit results
app.get('/posts', async (ctx) => {
  const page = parseInt(ctx.query.page || '1');
  const limit = Math.min(parseInt(ctx.query.limit || '10'), 100);
  const offset = (page - 1) * limit;

  const result = await pool.query(
    'SELECT * FROM posts ORDER BY created_at DESC LIMIT $1 OFFSET $2',
    [limit, offset]
  );

  ctx.json({ posts: result.rows, page, limit });
});

// Select only needed columns
app.get('/users', async (ctx) => {
  // Instead of SELECT *
  const result = await pool.query(
    'SELECT id, name, email FROM users'
  );

  ctx.json({ users: result.rows });
});

// Use prepared statements
const getUserStatement = 'SELECT * FROM users WHERE id = $1';

app.get('/users/:id', async (ctx) => {
  const result = await pool.query(getUserStatement, [ctx.params.id]);
  ctx.json(result.rows[0]);
});
```

### Read Replicas

```typescript
// Primary database (writes)
const primaryPool = new Pool({
  host: process.env.PRIMARY_DB_HOST,
  // ... other config
});

// Read replica (reads)
const replicaPool = new Pool({
  host: process.env.REPLICA_DB_HOST,
  // ... other config
});

// Write operations use primary
app.post('/users', async (ctx) => {
  const result = await primaryPool.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
    [ctx.body.name, ctx.body.email]
  );

  ctx.json(result.rows[0], 201);
});

// Read operations use replica
app.get('/users', async (ctx) => {
  const result = await replicaPool.query('SELECT * FROM users');
  ctx.json({ users: result.rows });
});
```

---

## Architecture Patterns

### Microservices Architecture

```
┌──────────────┐     ┌──────────────┐
│   API        │────▶│  User        │
│   Gateway    │     │  Service     │
│              │     └──────────────┘
│   (RamAPI)   │     ┌──────────────┐
│              │────▶│  Order       │
│              │     │  Service     │
└──────────────┘     └──────────────┘
                     ┌──────────────┐
                    │  Product     │
                    │  Service     │
                    └──────────────┘
```

**API Gateway:**

```typescript
import { createApp } from 'ramapi';

const app = createApp();

// Proxy to user service
app.all('/users/*', async (ctx) => {
  const response = await fetch(
    `http://user-service:3001${ctx.path}`,
    {
      method: ctx.method,
      headers: ctx.headers,
      body: ctx.method !== 'GET' ? JSON.stringify(ctx.body) : undefined,
    }
  );

  const data = await response.json();
  ctx.status(response.status);
  ctx.json(data);
});

// Proxy to order service
app.all('/orders/*', async (ctx) => {
  const response = await fetch(
    `http://order-service:3002${ctx.path}`,
    {
      method: ctx.method,
      headers: ctx.headers,
      body: ctx.method !== 'GET' ? JSON.stringify(ctx.body) : undefined,
    }
  );

  const data = await response.json();
  ctx.status(response.status);
  ctx.json(data);
});

app.listen(3000);
```

### Event-Driven Architecture

```typescript
import { EventEmitter } from 'events';
import Redis from 'ioredis';

const events = new EventEmitter();
const redis = new Redis(process.env.REDIS_URL);
const redisPub = new Redis(process.env.REDIS_URL);

// Subscribe to events
redis.subscribe('user:created', 'order:placed');

redis.on('message', (channel, message) => {
  const data = JSON.parse(message);
  events.emit(channel, data);
});

// User service
app.post('/users', async (ctx) => {
  const user = await createUser(ctx.body);

  // Publish event
  await redisPub.publish('user:created', JSON.stringify(user));

  ctx.json(user, 201);
});

// Email service (separate process)
events.on('user:created', async (user) => {
  await sendWelcomeEmail(user.email);
  console.log(`Welcome email sent to ${user.email}`);
});

// Analytics service (separate process)
events.on('user:created', async (user) => {
  await trackUserRegistration(user);
  console.log(`User registration tracked: ${user.id}`);
});
```

### CQRS (Command Query Responsibility Segregation)

```typescript
// Write model (commands)
class UserCommandService {
  async createUser(data: CreateUserDTO) {
    const user = await db.insert('users', data);

    // Update read model
    await redis.set(`user:${user.id}`, JSON.stringify(user));
    await redis.sadd('users:all', user.id);

    return user;
  }

  async updateUser(id: string, data: UpdateUserDTO) {
    const user = await db.update('users', id, data);

    // Update read model
    await redis.set(`user:${id}`, JSON.stringify(user));

    return user;
  }
}

// Read model (queries)
class UserQueryService {
  async getUser(id: string) {
    // Try cache first
    const cached = await redis.get(`user:${id}`);
    if (cached) {
      return JSON.parse(cached);
    }

    // Fallback to database
    const user = await db.findById('users', id);
    await redis.set(`user:${id}`, JSON.stringify(user));

    return user;
  }

  async getAllUsers() {
    const userIds = await redis.smembers('users:all');

    const users = await Promise.all(
      userIds.map(id => this.getUser(id))
    );

    return users;
  }
}

// Use in routes
const userCommands = new UserCommandService();
const userQueries = new UserQueryService();

app.post('/users', async (ctx) => {
  const user = await userCommands.createUser(ctx.body);
  ctx.json(user, 201);
});

app.get('/users/:id', async (ctx) => {
  const user = await userQueries.getUser(ctx.params.id);
  ctx.json(user);
});
```

---

## Performance Monitoring

### Application Metrics

```typescript
import { Counter, Histogram, Gauge } from 'prom-client';

// Request counter
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

// Request duration
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

// Active connections
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
});

// Metrics middleware
app.use(async (ctx, next) => {
  const start = Date.now();
  activeConnections.inc();

  try {
    await next();

    const duration = (Date.now() - start) / 1000;
    httpRequestsTotal.labels(ctx.method, ctx.path, String(ctx.statusCode)).inc();
    httpRequestDuration.labels(ctx.method, ctx.path, String(ctx.statusCode)).observe(duration);
  } finally {
    activeConnections.dec();
  }
});
```

---

## Capacity Planning

### Calculating Capacity

```typescript
// Example calculations

const requestsPerSecond = 1000;
const avgResponseTime = 0.05; // 50ms
const concurrentConnections = requestsPerSecond * avgResponseTime; // 50

// Memory per request
const memoryPerRequest = 10 * 1024; // 10 KB
const totalMemoryForRequests = concurrentConnections * memoryPerRequest; // 500 KB

// Number of instances needed
const instanceCapacity = 100; // requests/second per instance
const instancesNeeded = Math.ceil(requestsPerSecond / instanceCapacity); // 10

console.log(`
Capacity Planning:
- Requests/second: ${requestsPerSecond}
- Concurrent connections: ${concurrentConnections}
- Memory needed: ${(totalMemoryForRequests / 1024 / 1024).toFixed(2)} MB
- Instances needed: ${instancesNeeded}
`);
```

---

## See Also

- [Production Setup](../deployment/production-setup.md)
- [Docker Deployment](../deployment/docker.md)
- [Cloud Deployment](../deployment/cloud-deployment.md)
- [Performance Optimization](../performance/optimization.md)
- [Production Observability](../deployment/production-observability.md)
