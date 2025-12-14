<!--
#nextjs #postgresql #api #validation #typescript #fullstack #database
-->

# Input Validation Patterns in Next.js APIs

## Introduction

Validate request data before database operations. Clear error messages, prevents invalid data.

## The Problem

Minimal validation leads to unclear errors and unnecessary database load.

```typescript
const { email, name } = req.body;
const result = await query('INSERT INTO users (email, name) VALUES ($1, $2)', [email, name]);
```

## The Solution

Validate in the API layer before database operations.

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

- User experience - Clear, actionable errors
- Performance - Catch errors before database queries
- Data integrity - Only valid data reaches database
- Maintainability - Centralized validation logic

Next: [error-handling-patterns.md](./error-handling-patterns.md)
