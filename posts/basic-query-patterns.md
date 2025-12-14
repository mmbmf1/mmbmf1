<!--
#postgresql #nextjs #api #database #queries #typescript #fullstack
-->

# Basic Query Patterns and Filtering

## Introduction

Dynamic query building with parameterized filters. Efficient database-level filtering, not JavaScript filtering.

## The Problem

Fetching everything and filtering in JavaScript is slow and doesn't use indexes.

```typescript
const result = await query('SELECT * FROM users');
const filtered = result.rows.filter(user => user.active && user.role === 'admin');
```

## The Solution

Build dynamic queries with parameterized filters.

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

- Performance - Database handles filtering efficiently
- Flexibility - Support multiple filter combinations
- Security - Parameterized queries prevent SQL injection
- Scalability - Works well as data grows

Next: [pagination-strategies.md](./pagination-strategies.md)
