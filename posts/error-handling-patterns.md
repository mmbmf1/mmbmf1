<!--
#nextjs #postgresql #api #errorhandling #typescript #fullstack #database
-->

# Error Handling Patterns in Next.js APIs

## Introduction

Consistent error handling across API routes. Clear error responses, proper HTTP status codes, secure error messages.

## The Problem

Generic error messages can expose database internals and create a confusing experience for API consumers.

```typescript
try {
  const result = await query('SELECT * FROM users WHERE id = $1', [id]);
} catch (error) {
  res.json({ error: error.message }); // Exposes database details
}
```

## The Solution

Use error handling utilities with consistent error formats.

**Error handling utility:**
```typescript
// lib/errors.ts
export class ApiError extends Error {
  constructor(public statusCode: number, public message: string, public code?: string) {
    super(message);
    this.name = 'ApiError';
  }
}

export function handleDatabaseError(error: any): ApiError {
  switch (error.code) {
    case '23505': return new ApiError(409, 'Resource already exists', 'CONFLICT');
    case '23503': return new ApiError(409, 'Cannot delete: related records exist', 'CONSTRAINT_VIOLATION');
    case '23502': return new ApiError(400, 'Required field is missing', 'VALIDATION_ERROR');
    default: return new ApiError(500, 'Database operation failed', 'DATABASE_ERROR');
  }
}
```

**Using error handling:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';
import { handleDatabaseError, ApiError } from '@/lib/errors';

export default async function handler(req, res) {
  try {
    const { id } = req.query;
    const result = await query('SELECT * FROM users WHERE id = $1', [id]);
    
    if (result.rows.length === 0) {
      throw new ApiError(404, 'User not found', 'NOT_FOUND');
    }
    
    res.json(result.rows[0]);
  } catch (error: any) {
    if (error instanceof ApiError) {
      return res.status(error.statusCode).json({ error: error.message, code: error.code });
    }
    
    if (error.code && (error.code.startsWith('23') || error.code.startsWith('42'))) {
      const apiError = handleDatabaseError(error);
      return res.status(apiError.statusCode).json({ error: apiError.message, code: apiError.code });
    }
    
    res.status(500).json({ error: 'An unexpected error occurred', code: 'INTERNAL_ERROR' });
  }
}
```

**App Router:**
```typescript
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { handleDatabaseError, ApiError } from '@/lib/errors';
import { NextResponse } from 'next/server';

export async function GET(request: Request, { params }: { params: { id: string } }) {
  try {
    const result = await query('SELECT * FROM users WHERE id = $1', [params.id]);
    
    if (result.rows.length === 0) {
      return NextResponse.json({ error: 'User not found', code: 'NOT_FOUND' }, { status: 404 });
    }
    
    return NextResponse.json(result.rows[0]);
  } catch (error: any) {
    if (error.code && (error.code.startsWith('23') || error.code.startsWith('42'))) {
      const apiError = handleDatabaseError(error);
      return NextResponse.json({ error: apiError.message, code: apiError.code }, { status: apiError.statusCode });
    }
    return NextResponse.json({ error: 'An unexpected error occurred', code: 'INTERNAL_ERROR' }, { status: 500 });
  }
}
```

**Common HTTP status codes:**
- `400` - Bad Request (validation errors)
- `404` - Not Found (resource doesn't exist)
- `409` - Conflict (unique constraint violations)
- `500` - Internal Server Error (unexpected errors)

## Benefits

- User experience - Clear, actionable error messages
- Security - Don't expose database internals
- Consistency - Standardized error format
- Debugging - Proper logging while hiding details

Next: [basic-query-patterns.md](./basic-query-patterns.md)
