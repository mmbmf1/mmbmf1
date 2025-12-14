<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building DELETE Endpoints in Next.js

## Introduction

Built DELETE endpoints in Next.js that remove records from PostgreSQL safely. This approach uses parameterized queries and proper error handling to delete data while checking for existence and handling foreign key constraints.

## The Problem

When building DELETE endpoints, you need to remove records safely. The typical approaches involve not checking if records exist or ignoring foreign key constraints, which leads to confusing error responses and orphaned data.

```typescript
// Problematic approach - no existence check
const sql = `DELETE FROM users WHERE id = ${id}`;
```

This works, but doesn't verify the record exists and can fail silently or cause foreign key violations.

## The Solution

Instead of blind deletion, we use parameterized queries with existence checks and proper error handling for constraints. The architecture flows from route parameters through existence verification to safe deletion.

### Architecture Overview

Route Parameters → Existence Check → Parameterized Delete → Database → Success Response

- **Route parameters**: ID from URL
- **Existence check**: Verify record exists before deletion
- **Parameterized delete**: Safe SQL with placeholders
- **Database delete**: Execute with parameters
- **Success response**: Confirm deletion

### Implementation

**Basic delete:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  if (req.method !== 'DELETE') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { id } = req.query;
  
  try {
    // Check if record exists first
    const checkResult = await query(
      'SELECT id FROM users WHERE id = $1',
      [id]
    );
    
    if (checkResult.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    // Delete the record
    await query('DELETE FROM users WHERE id = $1', [id]);
    
    res.status(204).send(); // No content
  } catch (error: any) {
    if (error.code === '23503') { // Foreign key violation
      return res.status(409).json({ 
        error: 'Cannot delete user with associated records' 
      });
    }
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to delete user' });
  }
}
```

**Delete with RETURNING (return deleted record):**
```typescript
const result = await query(
  'DELETE FROM users WHERE id = $1 RETURNING *',
  [id]
);

if (result.rows.length === 0) {
  return res.status(404).json({ error: 'User not found' });
}

res.json({ message: 'User deleted', user: result.rows[0] });
```

**Soft delete (mark as deleted instead of removing):**
```typescript
// Instead of DELETE, update deleted_at timestamp
const result = await query(
  `UPDATE users 
   SET deleted_at = NOW(), active = false 
   WHERE id = $1 AND deleted_at IS NULL 
   RETURNING *`,
  [id]
);

if (result.rows.length === 0) {
  return res.status(404).json({ error: 'User not found or already deleted' });
}

res.json({ message: 'User deleted', user: result.rows[0] });
```

**App Router example:**
```typescript
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    // Check existence
    const checkResult = await query(
      'SELECT id FROM users WHERE id = $1',
      [params.id]
    );
    
    if (checkResult.rows.length === 0) {
      return NextResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    }
    
    // Delete
    await query('DELETE FROM users WHERE id = $1', [params.id]);
    
    return new NextResponse(null, { status: 204 });
  } catch (error: any) {
    if (error.code === '23503') {
      return NextResponse.json(
        { error: 'Cannot delete user with associated records' },
        { status: 409 }
      );
    }
    return NextResponse.json(
      { error: 'Failed to delete user' },
      { status: 500 }
    );
  }
}
```

### Best Practices

- **Check existence first** - Verify record exists before deletion
- **Use parameterized queries** - Prevents SQL injection
- **Handle foreign keys** - Check for constraint violations
- **Consider soft deletes** - Mark as deleted instead of removing
- **Proper status codes** - 204 for success, 404 for not found
- **Return deleted data** - Use RETURNING if client needs confirmation

## Benefits

This approach provides safe DELETE endpoints that handle existence checks and constraints properly. We get SQL injection protection, proper HTTP status codes, and clear error messages. This pattern works well for:

- **Safety** - Verifies existence before deletion
- **Reliability** - Handles foreign key constraints gracefully
- **User experience** - Clear error messages for conflicts
- **Flexibility** - Supports both hard and soft deletes

The clean separation between request handling and database operations means delete endpoints are secure and maintainable while providing good user feedback.

This builds on update endpoints (see [building-update-endpoints.md](./building-update-endpoints.md)). Next, see how to validate input properly (see [input-validation-patterns.md](./input-validation-patterns.md)).
