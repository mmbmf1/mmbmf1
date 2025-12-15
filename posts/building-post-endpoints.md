<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building POST Endpoints in Next.js

## Introduction

POST endpoints with parameterized queries. Safe inserts, handles conflicts, returns created records. These patterns ensure data integrity while providing clear feedback to API consumers. Essential for any application that needs to create new records safely and reliably.

## The Problem

String concatenation in queries can lead to SQL injection vulnerabilities. Missing conflict handling can result in unclear error messages. When you build INSERT queries by concatenating user input, malicious users can inject SQL code. Even worse, when unique constraint violations occur, PostgreSQL returns cryptic error codes that don't help API consumers understand what went wrong.

```typescript
const sql = `INSERT INTO users (email, name) VALUES ('${email}', '${name}')`;
```

## The Solution

Use parameterized queries with proper error handling. Parameterized queries prevent SQL injection completely, and proper error handling translates PostgreSQL error codes into clear, user-friendly messages. Catch constraint violations and return appropriate HTTP status codes that API consumers can handle gracefully.

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

- **Security** - Prevents SQL injection by design. Parameterized queries make it impossible for user input to be executed as SQL code, protecting your database from malicious attacks.
- **Reliability** - Handles conflicts gracefully. When duplicate records are attempted, your API returns clear error messages instead of crashing or returning cryptic database errors.
- **User experience** - Clear error messages that help API consumers understand what went wrong and how to fix it. Proper HTTP status codes make error handling straightforward.
- **Maintainability** - Consistent patterns across all your POST endpoints. Once you establish the pattern, every endpoint follows the same reliable approach.

Next: [building-update-endpoints.md](./building-update-endpoints.md)
