<!--
#postgresql #nextjs #api #pagination #database #typescript #fullstack
-->

# Pagination Strategies for API Endpoints

## Introduction

Pagination with LIMIT and OFFSET. Handle large datasets efficiently with navigation metadata.

## The Problem

Returning all data can be slow and consume excessive bandwidth for large datasets.

```typescript
const result = await query('SELECT * FROM users');
res.json(result.rows); // Could be thousands of records
```

## The Solution

Use LIMIT and OFFSET with metadata for navigation.

**Offset-based pagination:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const page = Math.max(1, parseInt(req.query.page as string) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(req.query.limit as string) || 20));
  const offset = (page - 1) * limit;
  
  const [countResult, dataResult] = await Promise.all([
    query('SELECT COUNT(*) FROM users'),
    query('SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2', [limit, offset])
  ]);
  
  const total = parseInt(countResult.rows[0].count);
  const totalPages = Math.ceil(total / limit);
  
  res.json({
    data: dataResult.rows,
    pagination: {
      page,
      limit,
      total,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1
    }
  });
}
```

**Cursor-based pagination:**
```typescript
const limit = Math.min(100, Math.max(1, parseInt(req.query.limit as string) || 20));
const cursor = req.query.cursor as string;

let sql = 'SELECT * FROM users';
const params: any[] = [limit + 1];

if (cursor) {
  sql += ' WHERE id > $2';
  params.push(parseInt(cursor));
}

sql += ' ORDER BY id ASC LIMIT $1';

const result = await query(sql, params);
const hasNext = result.rows.length > limit;
const data = hasNext ? result.rows.slice(0, -1) : result.rows;
const nextCursor = hasNext ? data[data.length - 1].id : null;

res.json({
  data,
  pagination: { limit, hasNext, nextCursor }
});
```

**App Router:**
```typescript
// app/api/users/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '20');
  const offset = (page - 1) * limit;
  
  const [countResult, dataResult] = await Promise.all([
    query('SELECT COUNT(*) FROM users'),
    query('SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2', [limit, offset])
  ]);
  
  const total = parseInt(countResult.rows[0].count);
  const totalPages = Math.ceil(total / limit);
  
  return NextResponse.json({
    data: dataResult.rows,
    pagination: { page, limit, total, totalPages, hasNext: page < totalPages, hasPrev: page > 1 }
  });
}
```

**Pagination strategies:**
- **Offset-based** - Page numbers, supports jumping to pages
- **Cursor-based** - Last ID, consistent results

## Benefits

- Performance - Only fetch requested page
- User experience - Manageable result sets
- Scalability - Works well as data grows
- Flexibility - Support different strategies

Next: [query-performance-optimization.md](./query-performance-optimization.md)
