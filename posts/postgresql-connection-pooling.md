<!--
#postgresql #nextjs #connectionpooling #database #typescript #performance #fullstack
-->

# PostgreSQL Connection Pooling in Next.js

![Connection Pooling Architecture](./images/connection-pooling.png)

## Introduction

Shared connection pool for PostgreSQL in Next.js. Reuses connections across requests, prevents leaks, improves performance.

## The Problem

Creating new connections per request can hit limits and potentially leak connections if cleanup is forgotten.

```typescript
const client = new Client({ /* config */ });
await client.connect();
const result = await client.query('SELECT * FROM users');
await client.end(); // Easy to forget
```

## The Solution

Use a singleton connection pool initialized once.

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

- Performance - Reuses connections
- Reliability - Auto-manages connections
- Scalability - Handles concurrent requests
- Simplicity - No manual connect/disconnect

Next: [nextjs-api-routes.md](./nextjs-api-routes.md)
