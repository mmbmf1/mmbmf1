<!--
#postgresql #nextjs #api #database #queries #typescript #fullstack
-->

# Basic Query Patterns and Filtering

## Introduction

Dynamic query building with parameterized filters. Efficient database-level filtering, not JavaScript filtering. Let PostgreSQL do what it does best: filter and sort data efficiently using indexes. This approach scales much better than fetching everything and filtering in your application code.

## The Problem

Fetching everything and filtering in JavaScript can be slow for large datasets and may not leverage database indexes effectively. When you fetch all records and filter in your application code, you're transferring unnecessary data over the network and doing work that PostgreSQL could do much more efficiently. Database indexes can't help if you're not using them in your queries.

```typescript
const result = await query('SELECT * FROM users');
const filtered = result.rows.filter(user => user.active && user.role === 'admin');
```

## The Solution

Build dynamic queries with parameterized filters. Construct your WHERE clause dynamically based on the filters provided, but always use parameterized queries to maintain security. This lets PostgreSQL use indexes effectively while keeping your queries safe from SQL injection.

**Basic filtering:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { active, role, search } = req.query;
  
  let sql = 'SELECT id, email, name FROM users WHERE 1=1';
  const params: any[] = [];
  let paramCount = 0;
  
  if (active !== undefined) {
    paramCount++;
    sql += ` AND active = $${paramCount}`;
    params.push(active === 'true');
  }
  
  if (role) {
    paramCount++;
    sql += ` AND role = $${paramCount}`;
    params.push(role);
  }
  
  if (search) {
    paramCount++;
    sql += ` AND (name ILIKE $${paramCount} OR email ILIKE $${paramCount})`;
    params.push(`%${search}%`);
  }
  
  const result = await query(sql + ' ORDER BY created_at DESC LIMIT 50', params);
  res.json(result.rows);
}
```

**App Router:**
```typescript
// app/api/users/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const active = searchParams.get('active');
  const role = searchParams.get('role');
  
  let sql = 'SELECT * FROM users WHERE 1=1';
  const params: any[] = [];
  let paramCount = 0;
  
  if (active !== null) {
    paramCount++;
    sql += ` AND active = $${paramCount}`;
    params.push(active === 'true');
  }
  
  if (role) {
    paramCount++;
    sql += ` AND role = $${paramCount}`;
    params.push(role);
  }
  
  const result = await query(sql + ' ORDER BY created_at DESC', params);
  return NextResponse.json(result.rows);
}
```

**Query patterns:**
- Equality filters - `active = true`
- Range filters - `age BETWEEN 18 AND 65`
- Text search - `ILIKE '%search%'`
- Multiple conditions - Combine with AND/OR
- Sorting - `ORDER BY created_at DESC`

## Benefits

- **Performance** - Database handles filtering efficiently using indexes and optimized query plans. PostgreSQL is much faster at filtering than JavaScript, especially as data grows.
- **Flexibility** - Support multiple filter combinations. Clients can combine different filters, and your API builds the appropriate query dynamically.
- **Security** - Parameterized queries prevent SQL injection. Even though you're building queries dynamically, parameters are safely separated from SQL structure.
- **Scalability** - Works well as data grows. Database-level filtering scales much better than fetching everything and filtering in memory.

Next: [pagination-strategies.md](./pagination-strategies.md)
