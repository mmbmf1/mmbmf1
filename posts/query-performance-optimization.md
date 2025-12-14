<!--
#postgresql #nextjs #api #performance #database #optimization #indexing
-->

# Query Performance Optimization

## Introduction

Optimize PostgreSQL queries with indexes, efficient patterns, and query analysis. Faster responses, better scalability.

## The Problem

Queries become slow as data grows without proper indexes and optimization.

```typescript
// Inefficient - no indexes, fetches unnecessary data
const result = await query('SELECT * FROM users WHERE email = $1', [email]);
const activeUsers = result.rows.filter(u => u.active);
```

## The Solution

Optimize at the database level with indexes and efficient patterns.

**Adding indexes:**
```sql
-- Single column indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Composite index for multiple conditions
CREATE INDEX idx_users_active_created ON users(active, created_at DESC);
```

**Efficient queries:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { active, role } = req.query;
  
  // Use indexed columns in WHERE clause
  let sql = 'SELECT id, email, name, created_at FROM users WHERE 1=1';
  const params: any[] = [];
  let paramCount = 0;
  
  // Filter on indexed column first
  if (active !== undefined) {
    paramCount++;
    sql += ` AND active = $${paramCount}`;
    params.push(active === 'true');
  }
  
  if (role) {
    paramCount++;
    sql += ` AND role = $${paramCount}`;
    params.push(role);
  }
  
  // Use indexed column for sorting
  sql += ' ORDER BY created_at DESC LIMIT 20';
  
  const result = await query(sql, params);
  res.json(result.rows);
}
```

**Query analysis:**
```typescript
// Analyze query performance
const explainResult = await query(
  'EXPLAIN ANALYZE SELECT * FROM users WHERE active = $1 ORDER BY created_at DESC LIMIT 20',
  [true]
);

console.log(explainResult.rows);
// Look for "Seq Scan" (bad) vs "Index Scan" (good)
```

**Select specific columns:**
```typescript
// Instead of SELECT *
const result = await query(
  'SELECT id, email, name FROM users WHERE active = $1',
  [true]
);
```

**Efficient JOINs:**
```typescript
// Use indexed foreign keys
const result = await query(`
  SELECT 
    u.id, u.email, u.name,
    p.title, p.created_at as post_created
  FROM users u
  INNER JOIN posts p ON p.user_id = u.id
  WHERE u.active = $1
  ORDER BY p.created_at DESC
  LIMIT 20
`, [true]);
```

**Connection pool configuration:**
```typescript
// lib/db.ts - Configure pool for performance
import { Pool } from 'pg';

const pool = new Pool({
  // ... connection config
  max: 20, // Adjust based on load
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  statement_timeout: 5000, // Kill slow queries
});
```

**Optimization techniques:**
- Indexes - Create on frequently queried columns
- Select specific columns - Don't use `SELECT *`
- Limit results - Always use LIMIT
- Efficient WHERE clauses - Filter on indexed columns
- Avoid N+1 queries - Use JOINs instead of multiple queries
- Connection pooling - Reuse connections efficiently

## Benefits

- Performance - Faster query execution
- Scalability - Handles growing datasets efficiently
- Resource usage - Reduced database load
- User experience - Faster API responses

Next: [psql-copy-command.md](./psql-copy-command.md)
