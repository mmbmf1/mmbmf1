<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building POST Endpoints in Next.js

## Introduction

Built POST endpoints in Next.js that create new records in PostgreSQL safely. This approach uses parameterized queries and proper error handling to insert data while preventing SQL injection and handling conflicts.

## The Problem

When building POST endpoints, you need to insert data into the database safely. The typical approaches involve string concatenation or not handling unique constraint violations, which leads to SQL injection vulnerabilities and poor error messages.

```typescript
// Unsafe approach - SQL injection risk
const sql = `INSERT INTO users (email, name) VALUES ('${email}', '${name}')`;
```

This works for simple cases, but is vulnerable to SQL injection and doesn't handle errors gracefully.

## The Solution

Instead of string concatenation, we use parameterized queries with proper error handling and conflict resolution. The architecture flows from request body through validation and parameterized inserts to success responses.

### Architecture Overview

Request Body → Validation → Parameterized Insert → Database → Success Response

- **Request body**: JSON data from client
- **Validation**: Check required fields and types
- **Parameterized insert**: Safe SQL with placeholders
- **Database insert**: Executes with parameters
- **Success response**: Returns created record

### Implementation

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
// Handle duplicate emails gracefully
const result = await query(
  `INSERT INTO users (email, name) 
   VALUES ($1, $2) 
   ON CONFLICT (email) DO UPDATE 
   SET name = EXCLUDED.name, updated_at = NOW()
   RETURNING *`,
  [email, name]
);
```

**App Router example:**
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

### Best Practices

- **Validate input** - Check required fields and data types
- **Use parameterized queries** - Prevents SQL injection
- **Handle conflicts** - Check for unique constraint violations
- **Return created record** - Use RETURNING clause
- **Proper status codes** - 201 for created, 409 for conflicts
- **Error messages** - Provide helpful error responses

## Benefits

This approach provides safe, robust POST endpoints that handle validation and errors properly. We get SQL injection protection, proper HTTP status codes, and clear error messages. This pattern works well for:

- **Security** - Parameterized queries prevent SQL injection
- **Reliability** - Proper error handling for edge cases
- **User experience** - Clear error messages for conflicts
- **Maintainability** - Consistent patterns across endpoints

The clean separation between request handling and database operations means endpoints are secure and maintainable while providing good user feedback.

This builds on GET endpoints (see [building-get-endpoints.md](./building-get-endpoints.md)). Next, see how to update data with PUT/PATCH endpoints (see [building-update-endpoints.md](./building-update-endpoints.md)).
