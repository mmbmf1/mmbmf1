<!--
#nextjs #postgresql #database #typescript #connectionpooling #fullstack #api
-->

# Connecting Next.js to PostgreSQL

## Introduction

Type-safe PostgreSQL connection for Next.js API routes. Connection pooling, environment configuration, works with Docker and production.

For detailed coverage, see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md) for connection pooling and [nextjs-api-routes.md](./nextjs-api-routes.md) for using the connection in API routes.

## The Problem

Creating new connections per request hits limits and leaks connections.

```typescript
const client = new Client({ /* config */ });
await client.connect();
const result = await client.query('SELECT * FROM users');
await client.end(); // Easy to forget
```

## The Solution

Use a shared database client with connection pooling.

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

- Performance - Connection pooling reduces overhead
- Reliability - Automatic connection management prevents leaks
- Simplicity - No manual connect/disconnect
- Flexibility - Same code works for Docker and production

With your database connected, you can now build API endpoints. For geospatial applications, see [postgresql-nextjs-geojson-pipeline.md](./postgresql-nextjs-geojson-pipeline.md) for PostGIS integration patterns.
