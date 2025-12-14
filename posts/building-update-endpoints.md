<!--
#nextjs #postgresql #api #restapi #typescript #fullstack #database
-->

# Building PUT/PATCH Endpoints in Next.js

## Introduction

Built update endpoints in Next.js that modify existing records in PostgreSQL safely. This approach uses parameterized queries with partial updates, handling both PUT (full replacement) and PATCH (partial update) patterns.

## The Problem

When building update endpoints, you need to modify existing records safely. The typical approaches involve updating all fields even when only one changed, or using string concatenation, which leads to unnecessary updates and SQL injection vulnerabilities.

```typescript
// Inefficient approach - updates all fields
const sql = `UPDATE users SET email='${email}', name='${name}', role='${role}' WHERE id=${id}`;
```

This works, but updates all fields even when only one changed and is vulnerable to SQL injection.

## The Solution

Instead of updating everything, we use dynamic query building with parameterized queries that only update provided fields. The architecture flows from request body through field detection to conditional updates.

### Architecture Overview

Request Body → Field Detection → Dynamic Update Query → Database → Updated Record

- **Request body**: JSON with fields to update
- **Field detection**: Identify which fields are provided
- **Dynamic update**: Build query with only provided fields
- **Database update**: Execute with parameters
- **Updated record**: Return updated data

### Implementation

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
  
  // Build dynamic update query
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

**App Router example:**
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

### Best Practices

- **PATCH for partial updates** - Only update provided fields
- **PUT for full replacement** - Require all fields
- **Use parameterized queries** - Prevents SQL injection
- **Handle not found** - Return 404 for missing records
- **Update timestamps** - Set updated_at automatically
- **Return updated record** - Use RETURNING clause

## Benefits

This approach provides flexible update endpoints that handle both partial and full updates safely. We get SQL injection protection, efficient updates, and proper HTTP status codes. This pattern works well for:

- **Flexibility** - Support both PATCH and PUT patterns
- **Efficiency** - Only update fields that changed
- **Security** - Parameterized queries prevent SQL injection
- **User experience** - Clear error messages for conflicts

The clean separation between request handling and database operations means update endpoints are secure and maintainable while supporting flexible update patterns.

This builds on POST endpoints (see [building-post-endpoints.md](./building-post-endpoints.md)). Next, see how to delete data with DELETE endpoints (see [building-delete-endpoints.md](./building-delete-endpoints.md)).
