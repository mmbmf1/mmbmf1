<!--
#postgresql #nextjs #api #database #queries #typescript #fullstack
-->

# Basic Query Patterns and Filtering

## Introduction

Built efficient query patterns for retrieving and filtering data from PostgreSQL in Next.js APIs. This approach uses parameterized queries with dynamic filtering to create flexible, performant endpoints.

## The Problem

When building API endpoints, you need to query data with various filters and conditions. The typical approaches involve separate endpoints for each filter combination or fetching all data and filtering in JavaScript, which leads to API bloat or performance issues.

```typescript
// Inefficient approach - fetch everything, filter in JavaScript
const result = await query('SELECT * FROM users');
const filtered = result.rows.filter(user => user.active && user.role === 'admin');
```

This works for small datasets, but becomes slow as data grows and doesn't leverage database indexes.

## The Solution

Instead of fetching everything, we build dynamic queries with parameterized filters that execute at the database level. The architecture flows from query parameters through dynamic query building to efficient database execution.

### Architecture Overview

Query Parameters → Dynamic Query Builder → Parameterized Query → Database → Filtered Results

- **Query parameters**: Filters from request
- **Dynamic query builder**: Constructs SQL with conditions
- **Parameterized query**: Safe SQL with placeholders
- **Database execution**: Uses indexes efficiently
- **Filtered results**: Only requested data returned

### Implementation

**Basic filtering:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { active, role, search } = req.query;
  
  let sql = 'SELECT * FROM users WHERE 1=1';
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
  
  sql += ' ORDER BY created_at DESC';
  
  const result = await query(sql, params);
  res.json(result.rows);
}
```

**Filtering with type safety:**
```typescript
interface UserFilters {
  active?: boolean;
  role?: string;
  search?: string;
  minAge?: number;
  maxAge?: number;
}

function buildUserQuery(filters: UserFilters) {
  let sql = 'SELECT * FROM users WHERE 1=1';
  const params: any[] = [];
  let paramCount = 0;
  
  if (filters.active !== undefined) {
    paramCount++;
    sql += ` AND active = $${paramCount}`;
    params.push(filters.active);
  }
  
  if (filters.role) {
    paramCount++;
    sql += ` AND role = $${paramCount}`;
    params.push(filters.role);
  }
  
  if (filters.search) {
    paramCount++;
    sql += ` AND (name ILIKE $${paramCount} OR email ILIKE $${paramCount})`;
    params.push(`%${filters.search}%`);
  }
  
  if (filters.minAge !== undefined) {
    paramCount++;
    sql += ` AND age >= $${paramCount}`;
    params.push(filters.minAge);
  }
  
  if (filters.maxAge !== undefined) {
    paramCount++;
    sql += ` AND age <= $${paramCount}`;
    params.push(filters.maxAge);
  }
  
  return { sql, params };
}

export default async function handler(req, res) {
  const filters: UserFilters = {
    active: req.query.active === 'true' ? true : req.query.active === 'false' ? false : undefined,
    role: req.query.role as string,
    search: req.query.search as string,
    minAge: req.query.minAge ? parseInt(req.query.minAge as string) : undefined,
    maxAge: req.query.maxAge ? parseInt(req.query.maxAge as string) : undefined,
  };
  
  const { sql, params } = buildUserQuery(filters);
  const result = await query(sql + ' ORDER BY created_at DESC', params);
  res.json(result.rows);
}
```

**App Router example:**
```typescript
// app/api/users/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  
  const active = searchParams.get('active');
  const role = searchParams.get('role');
  const search = searchParams.get('search');
  
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
  
  if (search) {
    paramCount++;
    sql += ` AND (name ILIKE $${paramCount} OR email ILIKE $${paramCount})`;
    params.push(`%${search}%`);
  }
  
  sql += ' ORDER BY created_at DESC';
  
  const result = await query(sql, params);
  return NextResponse.json(result.rows);
}
```

### Query Patterns

- **Equality filters** - Exact matches (`active = true`)
- **Range filters** - Numeric ranges (`age BETWEEN 18 AND 65`)
- **Text search** - Pattern matching (`ILIKE '%search%'`)
- **Multiple conditions** - Combine filters with AND/OR
- **Sorting** - ORDER BY for consistent results

## Benefits

This approach provides flexible querying that executes efficiently at the database level. We get parameterized queries, proper index usage, and reduced data transfer. This pattern works well for:

- **Performance** - Database handles filtering efficiently
- **Flexibility** - Support multiple filter combinations
- **Security** - Parameterized queries prevent SQL injection
- **Scalability** - Works well as data grows

The clean separation between filter building and query execution means endpoints are flexible and performant while maintaining security.

This builds on error handling (see [error-handling-patterns.md](./error-handling-patterns.md)). Next, see how to implement pagination (see [pagination-strategies.md](./pagination-strategies.md)).
