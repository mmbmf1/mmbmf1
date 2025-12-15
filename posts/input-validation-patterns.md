<!--
#nextjs #postgresql #api #validation #typescript #fullstack #database
-->

# Input Validation Patterns in Next.js APIs

## Introduction

Validate request data before database operations. Clear error messages, prevents invalid data. Catching errors early saves database resources and provides better feedback to API consumers. Essential for building APIs that are both secure and user-friendly.

## The Problem

Minimal validation can lead to unclear errors and unnecessary database load. When invalid data reaches your database, PostgreSQL returns cryptic error codes that don't help API consumers understand what went wrong. Worse, invalid data might pass initial checks but cause problems later, wasting database resources on operations that are doomed to fail.

```typescript
const { email, name } = req.body;
const result = await query('INSERT INTO users (email, name) VALUES ($1, $2)', [email, name]);
```

## The Solution

Validate in the API layer before database operations. Check data types, formats, and constraints before sending queries to PostgreSQL. This catches errors early and provides clear, actionable feedback to API consumers. Use validation libraries like Zod for type-safe validation that integrates seamlessly with TypeScript.

**Manual validation:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

function validateUserInput(body: any) {
  const errors: string[] = [];
  
  if (!body.email || typeof body.email !== 'string') {
    errors.push('Email is required');
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(body.email)) {
    errors.push('Invalid email format');
  }
  
  if (!body.name || typeof body.name !== 'string' || body.name.length < 2) {
    errors.push('Name must be at least 2 characters');
  }
  
  return { isValid: errors.length === 0, errors };
}

export default async function handler(req, res) {
  const validation = validateUserInput(req.body);
  
  if (!validation.isValid) {
    return res.status(400).json({ error: 'Validation failed', details: validation.errors });
  }
  
  const { email, name } = req.body;
  const result = await query('INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *', [email, name]);
  res.status(201).json(result.rows[0]);
}
```

**Using Zod:**
```typescript
import { z } from 'zod';
import { query } from '@/lib/db';

const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
  age: z.number().int().min(0).max(150).optional(),
});

export default async function handler(req, res) {
  const validation = userSchema.safeParse(req.body);
  
  if (!validation.success) {
    return res.status(400).json({ error: 'Validation failed', details: validation.error.errors });
  }
  
  const { email, name, age } = validation.data;
  const result = await query('INSERT INTO users (email, name, age) VALUES ($1, $2, $3) RETURNING *', [email, name, age]);
  res.status(201).json(result.rows[0]);
}
```

**App Router with Zod:**
```typescript
// app/api/users/route.ts
import { z } from 'zod';
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
});

export async function POST(request: Request) {
  const body = await request.json();
  const validation = userSchema.safeParse(body);
  
  if (!validation.success) {
    return NextResponse.json({ error: 'Validation failed', details: validation.error.errors }, { status: 400 });
  }
  
  const { email, name } = validation.data;
  const result = await query('INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *', [email, name]);
  return NextResponse.json(result.rows[0], { status: 201 });
}
```

**Validation patterns:**
- Type checking - Verify data types
- Format validation - Email, URL, date formats
- Range validation - Number bounds
- Required fields - Check presence
- String length - Min/max lengths

## Benefits

- **User experience** - Clear, actionable errors that tell API consumers exactly what's wrong and how to fix it. No more guessing about cryptic database error codes.
- **Performance** - Catch errors before database queries. Invalid data is rejected immediately, saving database resources and improving response times.
- **Data integrity** - Only valid data reaches database. Your database constraints become a safety net rather than the primary validation mechanism.
- **Maintainability** - Centralized validation logic that's easy to update and test. Changes to validation rules happen in one place, not scattered across your codebase.

Next: [error-handling-patterns.md](./error-handling-patterns.md)
