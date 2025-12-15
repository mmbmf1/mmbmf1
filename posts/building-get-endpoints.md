<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building GET Endpoints in Next.js

## Introduction

GET endpoints with parameterized queries. Safe, efficient, handles filtering and errors. These patterns protect against SQL injection while keeping your code readable and maintainable. Perfect for building APIs that retrieve data with various filters and search capabilities.

## The Problem

String concatenation in queries can lead to SQL injection vulnerabilities. When you build SQL queries by concatenating user input directly into strings, malicious users can inject their own SQL code. This is one of the most common and dangerous security vulnerabilities in web applications. Even if you think your input is safe, edge cases and unexpected input formats can create vulnerabilities.

```typescript
const result = await query(`SELECT * FROM users WHERE id = ${req.query.id}`);
```

## The Solution

Use parameterized queries with proper error handling. PostgreSQL treats parameters as data, not SQL code, completely preventing injection attacks. The database engine separates the query structure from the data values, making it impossible for user input to be interpreted as SQL commands. This approach is both safer and more efficient than string concatenation.

**Single record:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { id } = req.query;
  const result = await query('SELECT id, email, name FROM users WHERE id = $1', [id]);
  
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
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request, { params }: { params: { id: string } }) {
  const result = await query('SELECT id, email, name FROM users WHERE id = $1', [params.id]);
  
  if (result.rows.length === 0) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 });
  }
  
  return NextResponse.json(result.rows[0]);
}
```

**Best practices:**
- Always use parameterized queries (`$1`, `$2`, etc.)
- Validate input types and ranges
- Return 404 for missing resources
- Use LIMIT to prevent large responses
- Select specific columns, not `SELECT *`

## Benefits

- **Security** - Prevents SQL injection by design. Parameterized queries make it impossible for user input to be executed as SQL code.
- **Performance** - Efficient queries that PostgreSQL can optimize and cache. The query planner can reuse execution plans for similar queries with different parameters.
- **Consistency** - Standard patterns that work the same way across all your endpoints. Once you learn the pattern, you can apply it everywhere.
- **Maintainability** - Clear code that's easy to read and understand. Other developers can quickly see what data is being queried and how.

Next: [building-post-endpoints.md](./building-post-endpoints.md)
