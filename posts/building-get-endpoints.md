<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building GET Endpoints in Next.js

## Introduction

GET endpoints with parameterized queries. Safe, efficient, handles filtering and errors.

## The Problem

String concatenation in queries leads to SQL injection.

```typescript
// Unsafe - SQL injection risk
const result = await query(`SELECT * FROM users WHERE id = ${req.query.id}`);
```

## The Solution

Use parameterized queries with proper error handling.

**Single record:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  try {
    const { id } = req.query;
    
    const result = await query(
      'SELECT id, email, name FROM users WHERE id = $1',
      [id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(result.rows[0]);
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to fetch user' });
  }
}
```

**List with filtering:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  try {
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
    
    sql += ' ORDER BY created_at DESC LIMIT 50';
    
    const result = await query(sql, params);
    res.json(result.rows);
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to fetch users' });
  }
}
```

**App Router:**
```typescript
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const result = await query(
      'SELECT id, email, name FROM users WHERE id = $1',
      [params.id]
    );
    
    if (result.rows.length === 0) {
      return NextResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    }
    
    return NextResponse.json(result.rows[0]);
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch user' },
      { status: 500 }
    );
  }
}
```

**Best practices:**
- Always use parameterized queries (`$1`, `$2`, etc.)
- Validate input types and ranges
- Return 404 for missing resources
- Use LIMIT to prevent large responses
- Select specific columns, not `SELECT *`

## Benefits

- Security - Prevents SQL injection
- Performance - Efficient queries
- Consistency - Standard patterns
- Maintainability - Clear code

Next: [building-post-endpoints.md](./building-post-endpoints.md)
