<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building GET Endpoints in Next.js

## Introduction

Built GET endpoints in Next.js that retrieve data from PostgreSQL efficiently. This approach uses parameterized queries and proper response formatting to create safe, performant read operations.

## The Problem

When building GET endpoints, you need to retrieve data from the database safely and efficiently. The typical approaches involve string concatenation for queries or fetching all data without filtering, which leads to SQL injection vulnerabilities and performance issues.

```typescript
// Unsafe approach - SQL injection risk
const result = await query(`SELECT * FROM users WHERE id = ${req.query.id}`);
```

This works for simple cases, but is vulnerable to SQL injection and doesn't scale to complex queries.

## The Solution

Instead of string concatenation, we use parameterized queries with proper filtering and response formatting. The architecture flows from request parameters through parameterized queries to formatted responses.

### Architecture Overview

Request Parameters → Parameterized Query → Database → Formatted Response

- **Request parameters**: Query string or route parameters
- **Parameterized queries**: Safe SQL with placeholders
- **Database query**: Executes with parameters
- **Formatted response**: JSON response with data

### Implementation

**Single record by ID:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { id } = req.query;
  
  const result = await query(
    'SELECT * FROM users WHERE id = $1',
    [id]
  );
  
  if (result.rows.length === 0) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  res.json(result.rows[0]);
}
```

**List with filtering:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { active, role } = req.query;
  
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
  
  const result = await query(sql, params);
  res.json(result.rows);
}
```

**App Router example:**
```typescript
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const result = await query(
    'SELECT * FROM users WHERE id = $1',
    [params.id]
  );
  
  if (result.rows.length === 0) {
    return NextResponse.json(
      { error: 'User not found' },
      { status: 404 }
    );
  }
  
  return NextResponse.json(result.rows[0]);
}
```

### Best Practices

- **Always use parameterized queries** - Prevents SQL injection
- **Validate input** - Check parameter types and ranges
- **Handle not found** - Return 404 for missing resources
- **Limit results** - Use LIMIT to prevent large responses
- **Select specific columns** - Don't use SELECT * in production

## Benefits

This approach provides safe, efficient GET endpoints that handle filtering and error cases properly. We get SQL injection protection, proper HTTP status codes, and clean response formatting. This pattern works well for:

- **Security** - Parameterized queries prevent SQL injection
- **Performance** - Efficient queries with proper filtering
- **Consistency** - Standard patterns across endpoints
- **Maintainability** - Clear, readable query construction

The clean separation between request handling and database queries means endpoints are secure and performant while maintaining readable code.

This builds on API routes (see [nextjs-api-routes.md](./nextjs-api-routes.md)). Next, see how to create data with POST endpoints (see [building-post-endpoints.md](./building-post-endpoints.md)).
