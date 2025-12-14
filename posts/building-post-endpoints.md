<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building POST Endpoints in Next.js

## Introduction

POST endpoints with parameterized queries. Safe inserts, handles conflicts, returns created records.

## The Problem

String concatenation leads to SQL injection. Missing conflict handling causes poor errors.

```typescript
// Unsafe - SQL injection risk
const sql = `INSERT INTO users (email, name) VALUES ('${email}', '${name}')`;
```

## The Solution

Use parameterized queries with proper error handling.

**Basic insert:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { email, name } = req.body;
  
  if (!email || !name) {
    return res.status(400).json({ error: 'Email and name are required' });
  }
  
  try {
    const result = await query(
      `INSERT INTO users (email, name) 
       VALUES ($1, $2) 
       RETURNING *`,
      [email, name]
    );
    
    res.status(201).json(result.rows[0]);
  } catch (error: any) {
    if (error.code === '23505') { // Unique violation
      return res.status(409).json({ error: 'Email already exists' });
    }
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to create user' });
  }
}
```

**Insert with conflict handling:**
```typescript
// Upsert pattern
const result = await query(
  `INSERT INTO users (email, name) 
   VALUES ($1, $2) 
   ON CONFLICT (email) DO UPDATE 
   SET name = EXCLUDED.name, updated_at = NOW()
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
  const body = await request.json();
  const { email, name } = body;
  
  if (!email || !name) {
    return NextResponse.json(
      { error: 'Email and name are required' },
      { status: 400 }
    );
  }
  
  try {
    const result = await query(
      `INSERT INTO users (email, name) 
       VALUES ($1, $2) 
       RETURNING *`,
      [email, name]
    );
    
    return NextResponse.json(result.rows[0], { status: 201 });
  } catch (error: any) {
    if (error.code === '23505') {
      return NextResponse.json(
        { error: 'Email already exists' },
        { status: 409 }
      );
    }
    return NextResponse.json(
      { error: 'Failed to create user' },
      { status: 500 }
    );
  }
}
```

**Multiple inserts:**
```typescript
// Batch insert
const values = users.map((_, i) => `($${i * 2 + 1}, $${i * 2 + 2})`).join(', ');
const params = users.flatMap(u => [u.email, u.name]);

const result = await query(
  `INSERT INTO users (email, name) 
   VALUES ${values} 
   RETURNING *`,
  params
);
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
