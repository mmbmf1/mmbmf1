<!--
#postgresql #nextjs #connectionpooling #database #typescript #performance #fullstack
-->

# PostgreSQL Connection Pooling in Next.js

<!-- ![Connection Pooling Architecture](images/connection-pooling.png) -->

## Introduction

Shared connection pool for PostgreSQL in Next.js. Reuses connections across requests, prevents leaks, improves performance. Instead of creating a new connection for every API request, the pool maintains a set of reusable connections. This dramatically reduces overhead and ensures your application scales efficiently.

## The Problem

Creating new connections per request can hit limits and potentially leak connections if cleanup is forgotten. Each new connection has overhead - authentication, memory allocation, and network setup. Under load, you might hit PostgreSQL's connection limit, causing requests to fail. Even worse, forgetting to close connections means they stay open indefinitely, slowly consuming resources until your database becomes unresponsive.

```typescript
const client = new Client({ /* config */ });
await client.connect();
const result = await client.query('SELECT * FROM users');
await client.end(); // Easy to forget
```

## The Solution

Use a singleton connection pool initialized once. The pool creates a set of reusable connections when your application starts and manages them automatically. When you need a connection, the pool provides one from its available set. When you're done, the connection returns to the pool instead of being destroyed, ready for the next request.

```typescript
// lib/db.ts
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'postgres',
  database: process.env.DB || 'your_database',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export async function query(text: string, params?: any[]) {
  return pool.query(text, params);
}

export default pool;
```

**Usage in API routes:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  res.json(result.rows);
}
```

**Environment variables (.env.local):**
```bash
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB=your_database
```

**How it works:**
- Pool created once when module loads
- Connections reused across requests
- Auto-scales up to `max` limit
- Failed connections replaced automatically
- Idle connections timeout and close

## Benefits

- **Performance** - Reuses connections instead of creating new ones for every request. This eliminates connection overhead and dramatically improves response times.
- **Reliability** - Auto-manages connections, handling failures gracefully and replacing broken connections automatically. You don't have to worry about connection state.
- **Scalability** - Handles concurrent requests efficiently by sharing a pool of connections. The pool grows and shrinks based on demand, up to your configured maximum.
- **Simplicity** - No manual connect/disconnect needed. Just import and use - the pool handles all the complexity behind the scenes.

Next: [nextjs-api-routes.md](./nextjs-api-routes.md)
