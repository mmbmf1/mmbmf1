<!--
#nextjs #postgresql #api #typescript #fullstack #database #restapi
-->

# Using PostgreSQL in Next.js API Routes

## Introduction

Use the shared database connection pool in Next.js API routes. Clean database access without connection management.

## The Problem

Managing connections in every route leads to duplication and inconsistent patterns.

```typescript
// Inconsistent - connection logic in every route
export default async function handler(req, res) {
  const pool = new Pool({ /* config */ });
  const result = await pool.query('SELECT * FROM users');
  // ...
}
```

## The Solution

Import the shared database client and use it directly.

**Pages Router:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  try {
    const result = await query('SELECT * FROM users WHERE active = $1', [true]);
    res.json(result.rows);
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Database query failed' });
  }
}
```

**App Router:**
```typescript
// app/api/users/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET() {
  try {
    const result = await query('SELECT * FROM users WHERE active = $1', [true]);
    return NextResponse.json(result.rows);
  } catch (error) {
    console.error('Database error:', error);
    return NextResponse.json(
      { error: 'Database query failed' },
      { status: 500 }
    );
  }
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
// Dynamic queries
const { id, active } = req.query;

const result = await query(
  'SELECT * FROM users WHERE id = $1 AND active = $2',
  [id, active === 'true']
);
```

**Multiple queries:**
```typescript
// Sequential queries
const userResult = await query('SELECT * FROM users WHERE id = $1', [id]);
const postsResult = await query('SELECT * FROM posts WHERE user_id = $1', [id]);

// Parallel queries
const [userResult, postsResult] = await Promise.all([
  query('SELECT * FROM users WHERE id = $1', [id]),
  query('SELECT * FROM posts WHERE user_id = $1', [id])
]);
```

**Transaction example:**
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
