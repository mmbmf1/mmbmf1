<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building PUT/PATCH Endpoints in Next.js

## Introduction

Update endpoints with dynamic query building. Supports partial updates (PATCH) and full replacement (PUT).

## The Problem

Updating all fields even when only one changed is inefficient. String concatenation is unsafe.

```typescript
// Inefficient - updates all fields
const sql = `UPDATE users SET email='${email}', name='${name}', role='${role}' WHERE id=${id}`;
```

## The Solution

Build dynamic queries that only update provided fields.

**PATCH - Partial updates:**
```typescript
// pages/api/users/[id].ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  if (req.method !== 'PATCH') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { id } = req.query;
  const { email, name, active } = req.body;
  
  const updates: string[] = [];
  const params: any[] = [];
  let paramCount = 0;
  
  if (email !== undefined) {
    paramCount++;
    updates.push(`email = $${paramCount}`);
    params.push(email);
  }
  
  if (name !== undefined) {
    paramCount++;
    updates.push(`name = $${paramCount}`);
    params.push(name);
  }
  
  if (active !== undefined) {
    paramCount++;
    updates.push(`active = $${paramCount}`);
    params.push(active);
  }
  
  if (updates.length === 0) {
    return res.status(400).json({ error: 'No fields to update' });
  }
  
  paramCount++;
  updates.push(`updated_at = NOW()`);
  params.push(id);
  
  try {
    const result = await query(
      `UPDATE users 
       SET ${updates.join(', ')} 
       WHERE id = $${paramCount} 
       RETURNING *`,
      params
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(result.rows[0]);
  } catch (error: any) {
    if (error.code === '23505') {
      return res.status(409).json({ error: 'Email already exists' });
    }
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to update user' });
  }
}
```

**PUT - Full replacement:**
```typescript
// PUT requires all fields
const { email, name, role, active } = req.body;

if (!email || !name || role === undefined || active === undefined) {
  return res.status(400).json({ error: 'All fields required for PUT' });
}

const result = await query(
  `UPDATE users 
   SET email = $1, name = $2, role = $3, active = $4, updated_at = NOW()
   WHERE id = $5 
   RETURNING *`,
  [email, name, role, active, id]
);
```

**App Router:**
```typescript
// app/api/users/[id]/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function PATCH(
  request: Request,
  { params }: { params: { id: string } }
) {
  const body = await request.json();
  const updates: string[] = [];
  const params_list: any[] = [];
  let paramCount = 0;
  
  Object.entries(body).forEach(([key, value]) => {
    if (value !== undefined) {
      paramCount++;
      updates.push(`${key} = $${paramCount}`);
      params_list.push(value);
    }
  });
  
  if (updates.length === 0) {
    return NextResponse.json(
      { error: 'No fields to update' },
      { status: 400 }
    );
  }
  
  paramCount++;
  params_list.push(params.id);
  
  const result = await query(
    `UPDATE users 
     SET ${updates.join(', ')}, updated_at = NOW()
     WHERE id = $${paramCount} 
     RETURNING *`,
    params_list
  );
  
  if (result.rows.length === 0) {
    return NextResponse.json(
      { error: 'User not found' },
      { status: 404 }
    );
  }
  
  return NextResponse.json(result.rows[0]);
}
```

**Best practices:**
- PATCH for partial updates (only provided fields)
- PUT for full replacement (require all fields)
- Use parameterized queries
- Return 404 for missing records
- Update `updated_at` automatically
- Use `RETURNING *` to return updated record

## Benefits

- Flexibility - Support both PATCH and PUT
- Efficiency - Only update fields that changed
- Security - Parameterized queries prevent SQL injection
- User experience - Clear error messages

Next: [building-delete-endpoints.md](./building-delete-endpoints.md)
