<!--
#nextjs #postgresql #database #typescript #connectionpooling #fullstack #api
-->

# Connecting Next.js to PostgreSQL

## Introduction

Type-safe PostgreSQL connection for Next.js API routes. Connection pooling, environment configuration, works with Docker and production. This approach ensures your database connections are managed efficiently and your code stays clean. Perfect foundation for building robust API endpoints.

For detailed coverage, see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md) for connection pooling and [nextjs-api-routes.md](./nextjs-api-routes.md) for using the connection in API routes.

## The Problem

Creating new connections per request hits limits and leaks connections. Every API request that creates a new database connection adds overhead and consumes resources. Under load, you'll quickly hit PostgreSQL's connection limit, causing requests to fail. Even worse, if you forget to close connections properly, they accumulate over time until your database becomes unresponsive.

```typescript
const client = new Client({ /* config */ });
await client.connect();
const result = await client.query('SELECT * FROM users');
await client.end(); // Easy to forget
```

## The Solution

Use a shared database client with connection pooling. Create one connection pool when your application starts, and reuse those connections across all API requests. The pool automatically manages connection lifecycle, handles failures, and scales based on demand. Your code stays simple - just import and query.

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
});

export async function query(text: string, params?: any[]) {
  return pool.query(text, params);
}
```

**Usage in API routes:**
```typescript
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  res.json(result.rows);
}
```

**Environment configuration (.env.local):**
```bash
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB=your_database
```

## Benefits

- **Performance** - Connection pooling reduces overhead by reusing existing connections instead of creating new ones for every request. This dramatically improves response times.
- **Reliability** - Automatic connection management prevents leaks and handles connection failures gracefully. The pool replaces broken connections automatically.
- **Simplicity** - No manual connect/disconnect needed. Import the query function and use it - the pool handles everything behind the scenes.
- **Flexibility** - Same code works for Docker and production. Just change environment variables, and your connection adapts automatically.

With your database connected, you can now build API endpoints. For geospatial applications, see [postgresql-nextjs-geojson-pipeline.md](./postgresql-nextjs-geojson-pipeline.md) for PostGIS integration patterns.
