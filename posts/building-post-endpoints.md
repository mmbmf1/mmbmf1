<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building POST Endpoints in Next.js

## Introduction

POST endpoints with parameterized queries. Safe inserts, handles conflicts, returns created records.

## The Problem

String concatenation in queries can lead to SQL injection vulnerabilities. Missing conflict handling can result in unclear error messages.

```typescript
const sql = `INSERT INTO users (email, name) VALUES ('${email}', '${name}')`;
```

## The Solution

Use parameterized queries with proper error handling.

**Basic insert:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { email, name } = req.body;
  
  try {
    const result = await query(
      'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
      [email, name]
    );
    res.status(201).json(result.rows[0]);
  } catch (error: any) {
    if (error.code === '23505') {
      return res.status(409).json({ error: 'Email already exists' });
    }
    res.status(500).json({ error: 'Failed to create user' });
  }
}
```

**Insert with conflict handling:**
```typescript
const result = await query(
  `INSERT INTO users (email, name) VALUES ($1, $2)
   ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name, updated_at = NOW()
   RETURNING *`,
  [email, name]
);
```

**App Router:**
```typescript
// app/api/users/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const { email, name } = await request.json();
  
  try {
    const result = await query(
      'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
      [email, name]
    );
    return NextResponse.json(result.rows[0], { status: 201 });
  } catch (error: any) {
    if (error.code === '23505') {
      return NextResponse.json({ error: 'Email already exists' }, { status: 409 });
    }
    return NextResponse.json({ error: 'Failed to create user' }, { status: 500 });
  }
}
```

**Best practices:**
- Validate input before database operations
- Use parameterized queries
- Handle unique constraint violations (code `23505`)
- Return created record with `RETURNING *`
- Use 201 for created, 409 for conflicts

## Benefits

- Security - Prevents SQL injection
- Reliability - Handles conflicts gracefully
- User experience - Clear error messages
- Maintainability - Consistent patterns

Next: [building-update-endpoints.md](./building-update-endpoints.md)
