<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building DELETE Endpoints in Next.js

## Introduction

DELETE endpoints with existence checks and constraint handling. Safe deletion with proper error responses. These patterns ensure you never delete records that don't exist and handle foreign key constraints gracefully. Provides clear feedback to API consumers about what happened with their deletion request.

## The Problem

Deleting without checking if records exist can lead to confusing responses. Missing foreign key constraint handling can cause unclear errors. If you delete a record that doesn't exist, PostgreSQL silently succeeds, leaving API consumers confused about whether anything actually happened. When foreign key constraints prevent deletion, PostgreSQL returns cryptic error codes that don't help users understand why deletion failed.

```typescript
const sql = `DELETE FROM users WHERE id = ${id}`;
```

## The Solution

Check existence first, handle constraints, use parameterized queries. Verify the record exists before attempting deletion, giving clear 404 responses when appropriate. Catch foreign key constraint violations and return meaningful error messages that explain why deletion isn't possible. Always use parameterized queries to prevent SQL injection.

**Basic delete:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { id } = req.query;
  
  const checkResult = await query('SELECT id FROM users WHERE id = $1', [id]);
  
  if (checkResult.rows.length === 0) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  try {
    await query('DELETE FROM users WHERE id = $1', [id]);
    res.status(204).send();
  } catch (error: any) {
    if (error.code === '23503') {
      return res.status(409).json({ error: 'Cannot delete user with associated records' });
    }
    res.status(500).json({ error: 'Failed to delete user' });
  }
}
```

**Delete with RETURNING:**
```typescript
const result = await query('DELETE FROM users WHERE id = $1 RETURNING *', [id]);

if (result.rows.length === 0) {
  return res.status(404).json({ error: 'User not found' });
}

res.json({ message: 'User deleted', user: result.rows[0] });
```

**Soft delete:**
```typescript
const result = await query(
  `UPDATE users SET deleted_at = NOW(), active = false 
   WHERE id = $1 AND deleted_at IS NULL RETURNING *`,
  [id]
);

if (result.rows.length === 0) {
  return res.status(404).json({ error: 'User not found or already deleted' });
}

res.json({ message: 'User deleted', user: result.rows[0] });
```

**App Router:**
```typescript
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function DELETE(request: Request, { params }: { params: { id: string } }) {
  const checkResult = await query('SELECT id FROM users WHERE id = $1', [params.id]);
  
  if (checkResult.rows.length === 0) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 });
  }
  
  try {
    await query('DELETE FROM users WHERE id = $1', [params.id]);
    return new NextResponse(null, { status: 204 });
  } catch (error: any) {
    if (error.code === '23503') {
      return NextResponse.json({ error: 'Cannot delete user with associated records' }, { status: 409 });
    }
    return NextResponse.json({ error: 'Failed to delete user' }, { status: 500 });
  }
}
```

**Best practices:**
- Check existence before deletion
- Use parameterized queries
- Handle foreign key violations (code `23503`)
- Consider soft deletes for audit trails
- Return 204 for success, 404 for not found

## Benefits

- **Safety** - Verifies existence before deletion. API consumers get clear feedback about whether a record exists, preventing confusion about deletion results.
- **Reliability** - Handles constraints gracefully. When foreign key relationships prevent deletion, your API explains why clearly instead of returning cryptic database errors.
- **User experience** - Clear error messages that help API consumers understand what happened. Proper HTTP status codes make error handling straightforward for client applications.
- **Flexibility** - Supports hard and soft deletes. You can implement permanent deletion or soft deletion patterns depending on your application's needs.

Next: [input-validation-patterns.md](./input-validation-patterns.md)
