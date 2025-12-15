<!--
#postgresql #nextjs #api #pagination #database #typescript #fullstack
-->

# Pagination Strategies for API Endpoints

## Introduction

Pagination with LIMIT and OFFSET. Handle large datasets efficiently with navigation metadata. Essential for any API that might return more data than a client can reasonably handle in one request. Provides a smooth browsing experience while keeping response sizes manageable.

## The Problem

Returning all data can be slow and consume excessive bandwidth for large datasets. When you have thousands or millions of records, fetching everything in one request becomes impractical. Response times increase, memory usage spikes, and network transfer becomes a bottleneck. Mobile clients especially struggle with large payloads.

```typescript
const result = await query('SELECT * FROM users');
res.json(result.rows); // Could be thousands of records
```

## The Solution

Use LIMIT and OFFSET with metadata for navigation. Break large result sets into manageable pages, returning only the records requested along with metadata that helps clients navigate through pages. Include information like total count, current page, and whether more pages exist.

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

- **Performance** - Only fetch requested page. Database queries return small, fast result sets instead of transferring massive amounts of data.
- **User experience** - Manageable result sets that load quickly and don't overwhelm clients. Users can navigate through data at their own pace.
- **Scalability** - Works well as data grows. Pagination performance stays consistent whether you have thousands or millions of records.
- **Flexibility** - Support different strategies. Offset-based pagination for page numbers, cursor-based for consistent results even as data changes.

Next: [query-performance-optimization.md](./query-performance-optimization.md)
