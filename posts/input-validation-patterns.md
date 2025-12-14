<!--
#nextjs #postgresql #api #validation #typescript #fullstack #database
-->

# Input Validation Patterns in Next.js APIs

## Introduction

Implemented input validation for Next.js API routes to ensure data integrity and provide clear error messages. This approach validates request data before database operations, preventing invalid data and improving user experience.

## The Problem

When building API endpoints, you need to validate incoming data before processing. The typical approaches involve minimal validation or validating in the database, which leads to unclear error messages and unnecessary database load.

```typescript
// Minimal validation - unclear errors
const { email, name } = req.body;
const result = await query('INSERT INTO users (email, name) VALUES ($1, $2)', [email, name]);
```

This works, but database errors are cryptic and don't provide helpful feedback to API consumers.

## The Solution

Instead of relying on database validation, we validate input in the API layer before database operations. The architecture flows from request body through validation functions to either error responses or database operations.

### Architecture Overview

Request Body → Validation Functions → Error Response OR Database Operation

- **Request body**: Incoming JSON data
- **Validation functions**: Check data types, formats, constraints
- **Error response**: Return validation errors with details
- **Database operation**: Proceed if validation passes

### Implementation

**Basic validation:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

function validateUserInput(body: any) {
  const errors: string[] = [];
  
  if (!body.email || typeof body.email !== 'string') {
    errors.push('Email is required and must be a string');
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(body.email)) {
    errors.push('Email must be a valid email address');
  }
  
  if (!body.name || typeof body.name !== 'string') {
    errors.push('Name is required and must be a string');
  } else if (body.name.length < 2) {
    errors.push('Name must be at least 2 characters');
  }
  
  if (body.age !== undefined) {
    if (typeof body.age !== 'number') {
      errors.push('Age must be a number');
    } else if (body.age < 0 || body.age > 150) {
      errors.push('Age must be between 0 and 150');
    }
  }
  
  return {
    isValid: errors.length === 0,
    errors
  };
}

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const validation = validateUserInput(req.body);
  
  if (!validation.isValid) {
    return res.status(400).json({ 
      error: 'Validation failed',
      details: validation.errors
    });
  }
  
  const { email, name, age } = req.body;
  
  try {
    const result = await query(
      'INSERT INTO users (email, name, age) VALUES ($1, $2, $3) RETURNING *',
      [email, name, age]
    );
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to create user' });
  }
}
```

**Using Zod for validation:**
```typescript
import { z } from 'zod';
import { query } from '@/lib/db';

const userSchema = z.object({
  email: z.string().email('Invalid email address'),
  name: z.string().min(2, 'Name must be at least 2 characters'),
  age: z.number().int().min(0).max(150).optional(),
});

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const validation = userSchema.safeParse(req.body);
  
  if (!validation.success) {
    return res.status(400).json({
      error: 'Validation failed',
      details: validation.error.errors
    });
  }
  
  const { email, name, age } = validation.data;
  
  try {
    const result = await query(
      'INSERT INTO users (email, name, age) VALUES ($1, $2, $3) RETURNING *',
      [email, name, age]
    );
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to create user' });
  }
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
  age: z.number().int().min(0).max(150).optional(),
});

export async function POST(request: Request) {
  const body = await request.json();
  
  const validation = userSchema.safeParse(body);
  
  if (!validation.success) {
    return NextResponse.json(
      {
        error: 'Validation failed',
        details: validation.error.errors
      },
      { status: 400 }
    );
  }
  
  const { email, name, age } = validation.data;
  
  try {
    const result = await query(
      'INSERT INTO users (email, name, age) VALUES ($1, $2, $3) RETURNING *',
      [email, name, age]
    );
    
    return NextResponse.json(result.rows[0], { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to create user' },
      { status: 500 }
    );
  }
}
```

### Validation Patterns

- **Type checking** - Verify data types match expectations
- **Format validation** - Check email, URL, date formats
- **Range validation** - Ensure numbers are within bounds
- **Required fields** - Check for presence of required data
- **String length** - Validate minimum/maximum lengths
- **Custom rules** - Business logic validation

## Benefits

This approach provides clear validation that catches errors before database operations. We get better error messages, reduced database load, and improved user experience. This pattern works well for:

- **User experience** - Clear, actionable error messages
- **Performance** - Catch errors before database queries
- **Data integrity** - Ensure only valid data reaches database
- **Maintainability** - Centralized validation logic

The clean separation between validation and database operations means endpoints are more robust and provide better feedback to API consumers.

This builds on POST endpoints (see [building-post-endpoints.md](./building-post-endpoints.md)). Next, see how to handle errors consistently (see [error-handling-patterns.md](./error-handling-patterns.md)).
