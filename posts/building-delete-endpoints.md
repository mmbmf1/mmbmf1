<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building DELETE Endpoints in Next.js

## Introduction

DELETE endpoints with existence checks and constraint handling. Safe deletion with proper error responses.

## The Problem

Deleting without checking if records exist can lead to confusing responses. Missing foreign key constraint handling can cause unclear errors.

```typescript
const sql = `DELETE FROM users WHERE id = ${id}`;
```

## The Solution

Check existence first, handle constraints, use parameterized queries.

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

- Safety - Verifies existence before deletion
- Reliability - Handles constraints gracefully
- User experience - Clear error messages
- Flexibility - Supports hard and soft deletes

Next: [input-validation-patterns.md](./input-validation-patterns.md)
