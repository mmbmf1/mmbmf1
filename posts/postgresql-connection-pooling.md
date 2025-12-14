<!--
#postgresql #nextjs #connectionpooling #database #typescript #performance #fullstack
-->

# PostgreSQL Connection Pooling in Next.js

## Introduction

Set up connection pooling for PostgreSQL in Next.js API routes to efficiently manage database connections across requests. This approach reuses connections instead of creating new ones for each request, improving performance and preventing connection exhaustion.

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

Instead of creating connections per request, we use a shared connection pool that's initialized once and reused across requests. The architecture flows from environment configuration through a singleton database client to Next.js API routes.

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
  max: 20, // Maximum pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export async function query(text: string, params?: any[]) {
  return pool.query(text, params);
}

export default pool;
```

The pool manages connections automatically—creating new ones as needed, reusing idle connections, and cleaning up when requests complete. No manual connection management required.

**Error handling**: The pool automatically handles connection failures by removing failed connections and creating new ones. For application-level error handling, wrap queries in try-catch blocks.

### How Connection Pooling Works

- **Pool creation**: Initialized once when module loads
- **Connection reuse**: Idle connections are reused for new queries
- **Automatic scaling**: Creates new connections up to `max` limit
- **Cleanup**: Idle connections timeout and close automatically
- **Error handling**: Failed connections are removed and replaced

## Benefits

This approach provides efficient database access with automatic connection management. We get connection pooling, proper resource cleanup, and better performance without manual connection handling. This pattern works well for:

- **Performance** - Connection pooling reduces overhead and connection churn
- **Reliability** - Automatic connection management prevents leaks
- **Scalability** - Handles concurrent requests efficiently
- **Simplicity** - No manual connect/disconnect in every route

The clean separation between connection management and API logic means database access is efficient and maintainable while Next.js handles the HTTP layer.

This builds on Docker setup (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)). Next, see how to use this connection in API routes (see [nextjs-api-routes.md](./nextjs-api-routes.md)).
