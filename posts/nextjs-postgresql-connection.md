<!--
#nextjs #postgresql #database #typescript #connectionpooling #fullstack #api #localdevelopment #docker #databaseclient
-->

# Connecting Next.js to PostgreSQL

## Introduction

Set up a type-safe PostgreSQL connection for Next.js API routes that handles connection pooling and environment configuration. This approach provides a clean database client interface that works seamlessly with Docker-based local development (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)) and production environments.

For detailed coverage, see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md) for connection pooling and [nextjs-api-routes.md](./nextjs-api-routes.md) for using the connection in API routes.

## The Problem

When building Next.js APIs that need database access, you need to establish PostgreSQL connections efficiently. The typical approaches involve creating new connections for each request or manually managing connection pools, which leads to connection leaks and poor performance.

```typescript
// Manual approach - creates new connection per request, easy to leak
const client = new Client({ /* config */ });
await client.connect();
const result = await client.query('SELECT * FROM users');
await client.end(); // Easy to forget
```

This works, but creates a new connection for every request, doesn't handle connection pooling, and makes it easy to forget cleanup. Connection limits get hit quickly under load.

## The Solution

Instead of creating connections per request, we built a shared database client with connection pooling that's initialized once and reused across requests. The architecture flows from environment configuration through a singleton database client to Next.js API routes.

### Architecture Overview

Environment Variables → Database Client Singleton → Connection Pool → API Routes

- **Environment variables**: Database connection configuration
- **Database client**: Singleton instance with connection pooling
- **Connection pool**: Reuses connections efficiently
- **API routes**: Clean database access without connection management

### Implementation

```typescript
// lib/db.ts - Shared connection pool
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

The pool manages connections automatically—creating new ones as needed, reusing idle connections, and cleaning up when requests complete. No manual connection management required.

### Usage in API Routes

```typescript
// API route - clean database access
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  res.json(result.rows);
}
```

### Environment Configuration

Connection details come from environment variables, making it easy to switch between local Docker and production:

```bash
# .env.local - same code works for Docker local dev and production
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB=your_database
```

The same code works for both environments—just change the environment variables.

## Benefits

This approach provides efficient database access with automatic connection management. We get connection pooling, proper resource cleanup, and type safety without manual connection handling. This pattern works well for:

- **Performance** - Connection pooling reduces overhead and connection churn
- **Reliability** - Automatic connection management prevents leaks
- **Simplicity** - No manual connect/disconnect in every route
- **Flexibility** - Same code works for Docker local dev and production
- **Type safety** - TypeScript integration for better developer experience

The clean separation between connection management and API logic means database access is efficient and maintainable while Next.js handles the HTTP layer. Connection pooling handles the heavy lifting of managing database connections, allowing API routes to focus on business logic.

With your database connected, you can now build API endpoints that process data efficiently. For geospatial applications, see [postgresql-nextjs-geojson-pipeline.md](./postgresql-nextjs-geojson-pipeline.md) for PostGIS integration patterns.
