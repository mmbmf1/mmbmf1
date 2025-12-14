<!--
#postgresql #nextjs #api #pagination #database #typescript #fullstack
-->

# Pagination Strategies for API Endpoints

## Introduction

Implemented pagination for Next.js API endpoints to handle large datasets efficiently. This approach uses LIMIT and OFFSET to return data in manageable chunks while providing metadata for navigation.

## The Problem

When building API endpoints, you need to handle large result sets. The typical approaches involve returning all data or using inefficient pagination, which leads to slow responses or poor user experience.

```typescript
// Problematic approach - returns everything
const result = await query('SELECT * FROM users');
res.json(result.rows); // Could be thousands of records
```

This works for small datasets, but becomes slow and consumes excessive bandwidth as data grows.

## The Solution

Instead of returning all data, we use LIMIT and OFFSET with metadata to paginate results. The architecture flows from pagination parameters through query construction to paginated responses with navigation info.

### Architecture Overview

Pagination Parameters → Query with LIMIT/OFFSET → Database → Paginated Response

- **Pagination parameters**: Page number or offset/limit
- **Query construction**: Add LIMIT and OFFSET clauses
- **Database execution**: Returns only requested page
- **Response metadata**: Include pagination info for navigation

### Implementation

**Offset-based pagination:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  try {
    const page = Math.max(1, parseInt(req.query.page as string) || 1);
    const limit = Math.min(100, Math.max(1, parseInt(req.query.limit as string) || 20));
    const offset = (page - 1) * limit;
    
    // Get total count for metadata
    const countResult = await query('SELECT COUNT(*) FROM users');
    const total = parseInt(countResult.rows[0].count);
    const totalPages = Math.ceil(total / limit);
    
    // Get paginated results
    const result = await query(
      'SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2',
      [limit, offset]
    );
    
    res.json({
      data: result.rows,
      pagination: {
        page,
        limit,
        total,
        totalPages,
        hasNext: page < totalPages,
        hasPrev: page > 1
      }
    });
  } catch (error) {
    console.error('Pagination error:', error);
    res.status(500).json({ error: 'Failed to fetch users' });
  }
}
```

**Cursor-based pagination (better for large datasets):**
```typescript
export default async function handler(req, res) {
  try {
    const limit = Math.min(100, Math.max(1, parseInt(req.query.limit as string) || 20));
    const cursor = req.query.cursor as string; // Last ID from previous page
    
    let sql = 'SELECT * FROM users';
    const params: any[] = [limit + 1]; // Fetch one extra to check for next page
    
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
      pagination: {
        limit,
        hasNext,
        nextCursor
      }
    });
  } catch (error) {
    console.error('Pagination error:', error);
    res.status(500).json({ error: 'Failed to fetch users' });
  }
}
```

**App Router with offset pagination:**
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
    query(
      'SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2',
      [limit, offset]
    )
  ]);
  
  const total = parseInt(countResult.rows[0].count);
  const totalPages = Math.ceil(total / limit);
  
  return NextResponse.json({
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

### Pagination Strategies

**Offset-based (page numbers):**
- Pros: Simple, supports jumping to specific pages
- Cons: Slower on large offsets, can skip/duplicate records if data changes
- Best for: Small to medium datasets, user-facing pagination

**Cursor-based (last ID):**
- Pros: Consistent results, faster on large datasets
- Cons: No jumping to specific pages, requires ordered column
- Best for: Large datasets, infinite scroll, real-time data

## Benefits

This approach provides efficient pagination that handles large datasets gracefully. We get reduced response sizes, better performance, and clear navigation metadata. This pattern works well for:

- **Performance** - Only fetch requested page of data
- **User experience** - Manageable result sets
- **Scalability** - Works well as data grows
- **Flexibility** - Support different pagination strategies

The clean separation between pagination logic and data fetching means endpoints are efficient and provide good user experience.

This builds on query patterns (see [basic-query-patterns.md](./basic-query-patterns.md)). Next, see how to optimize query performance (see [query-performance-optimization.md](./query-performance-optimization.md)).
