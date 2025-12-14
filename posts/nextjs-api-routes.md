<!--
#nextjs #postgresql #api #typescript #fullstack #database #restapi
-->

# Using PostgreSQL in Next.js API Routes

## Introduction

Built Next.js API routes that use the PostgreSQL connection pool to handle database queries. This approach provides clean database access in API endpoints without managing connections manually.

## The Problem

When building API routes in Next.js, you need to query the database efficiently. The typical approaches involve importing connection code in every route or recreating database clients, which leads to code duplication and inconsistent patterns.

```typescript
// Inconsistent approach - connection logic in every route
export default async function handler(req, res) {
  const pool = new Pool({ /* config */ });
  const result = await pool.query('SELECT * FROM users');
  // ...
}
```

This works, but duplicates connection setup in every route and doesn't leverage connection pooling.

## The Solution

Instead of managing connections in each route, we import the shared database client and use it directly. The architecture flows from the connection pool through API routes to database queries.

### Architecture Overview

Connection Pool → API Route → Database Query → Response

- **Connection pool**: Shared database client (from [postgresql-connection-pooling.md](./postgresql-connection-pooling.md))
- **API route**: Next.js API endpoint handler
- **Database query**: Uses pooled connection
- **Response**: Returns data to client

### Implementation

```typescript
// pages/api/users.ts (Pages Router)
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  res.json(result.rows);
}
```

```typescript
// app/api/users/route.ts (App Router)
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET() {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  return NextResponse.json(result.rows);
}
```

### Environment Configuration

Connection details come from environment variables, making it easy to switch between local Docker and production:

```bash
# .env.local
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB=your_database
```

The same code works for both environments—just change the environment variables.

### Error Handling

```typescript
import { query } from '@/lib/db';

export default async function handler(req, res) {
  try {
    const result = await query('SELECT * FROM users');
    res.json(result.rows);
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Database query failed' });
  }
}
```

## Benefits

This approach provides clean database access in API routes with automatic connection management. We get consistent patterns, efficient connection reuse, and simple error handling. This pattern works well for:

- **Consistency** - Same database access pattern across all routes
- **Performance** - Connection pooling handles efficiency
- **Simplicity** - Import and use, no connection management
- **Flexibility** - Works with Docker local dev and production

The clean separation between connection management and route logic means API routes focus on business logic while the connection pool handles database efficiency.

This builds on connection pooling (see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)). Next, see how to build specific endpoint types (see [building-get-endpoints.md](./building-get-endpoints.md)).
