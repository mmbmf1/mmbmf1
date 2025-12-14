<!--
#nextjs #postgresql #api #typescript #fullstack #database #restapi
-->

# Using PostgreSQL in Next.js API Routes

## Introduction

Use the shared database connection pool in Next.js API routes. Clean database access without connection management.

## The Problem

Managing connections in every route can lead to duplication and inconsistent patterns.

```typescript
export default async function handler(req, res) {
  const pool = new Pool({ /* config */ });
  const result = await pool.query('SELECT * FROM users');
}
```

## The Solution

Import the shared database client and use it directly.

**Pages Router:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  res.json(result.rows);
}
```

**App Router:**
```typescript
// app/api/users/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET() {
  const result = await query('SELECT * FROM users WHERE active = $1', [true]);
  return NextResponse.json(result.rows);
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

**Query with parameters:**
```typescript
const { id, active } = req.query;
const result = await query('SELECT * FROM users WHERE id = $1 AND active = $2', [id, active === 'true']);
```

**Parallel queries:**
```typescript
const [userResult, postsResult] = await Promise.all([
  query('SELECT * FROM users WHERE id = $1', [id]),
  query('SELECT * FROM posts WHERE user_id = $1', [id])
]);
```

**Transaction:**
```typescript
import pool from '@/lib/db';

const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT INTO users (email) VALUES ($1)', [email]);
  await client.query('INSERT INTO profiles (user_id) VALUES ($1)', [userId]);
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
```

## Benefits

- Consistency - Same pattern everywhere
- Performance - Connection pooling handles efficiency
- Simplicity - Import and use
- Flexibility - Works with Docker and production

Next: [building-get-endpoints.md](./building-get-endpoints.md)
