<!--
#postgresql #nextjs #api #performance #database #optimization #indexing
-->

# Query Performance Optimization

## Introduction

Optimize PostgreSQL queries with indexes, efficient patterns, and query analysis. Faster responses, better scalability. Understanding how PostgreSQL executes your queries helps you identify bottlenecks and optimize effectively. Essential knowledge for building APIs that perform well as your data grows.

## The Problem

Queries can become slow as data grows without proper indexes and optimization. As your database grows from thousands to millions of records, queries that were fast can suddenly become painfully slow. Without indexes, PostgreSQL has to scan entire tables, which becomes exponentially slower as data increases. Poor query patterns can also waste resources even when indexes exist.

```typescript
const result = await query('SELECT * FROM users WHERE email = $1', [email]);
const activeUsers = result.rows.filter(u => u.active);
```

## The Solution

Optimize at the database level with indexes and efficient patterns. Create indexes on columns you frequently query, use EXPLAIN ANALYZE to understand query execution plans, and write queries that leverage indexes effectively. Select only the columns you need, use LIMIT to cap result sizes, and structure JOINs efficiently.

**Adding indexes:**
```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Composite index
CREATE INDEX idx_users_active_created ON users(active, created_at DESC);
```

**Efficient queries:**
```typescript
// pages/api/users.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { active, role } = req.query;
  
  let sql = 'SELECT id, email, name FROM users WHERE 1=1';
  const params: any[] = [];
  let paramCount = 0;
  
  if (active !== undefined) {
    paramCount++;
    sql += ` AND active = $${paramCount}`;
    params.push(active === 'true');
  }
  
  sql += ' ORDER BY created_at DESC LIMIT 20';
  
  const result = await query(sql, params);
  res.json(result.rows);
}
```

**Query analysis:**
```typescript
const explainResult = await query(
  'EXPLAIN ANALYZE SELECT * FROM users WHERE active = $1 ORDER BY created_at DESC LIMIT 20',
  [true]
);
// Look for "Seq Scan" (full table scan) vs "Index Scan" (uses index)
```

**Select specific columns:**
```typescript
const result = await query('SELECT id, email, name FROM users WHERE active = $1', [true]);
```

**Efficient JOINs:**
```typescript
const result = await query(`
  SELECT u.id, u.email, p.title
  FROM users u
  INNER JOIN posts p ON p.user_id = u.id
  WHERE u.active = $1
  ORDER BY p.created_at DESC LIMIT 20
`, [true]);
```

**Connection pool configuration:**
```typescript
// lib/db.ts
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  statement_timeout: 5000,
});
```

**Optimization techniques:**
- Indexes - Create on frequently queried columns
- Select specific columns - Don't use `SELECT *`
- Limit results - Always use LIMIT
- Efficient WHERE clauses - Filter on indexed columns
- Avoid N+1 queries - Use JOINs instead of multiple queries

## Benefits

- **Performance** - Faster query execution through proper indexing and query optimization. Well-indexed queries can be orders of magnitude faster than full table scans.
- **Scalability** - Handles growing datasets efficiently. Performance stays consistent as your data grows from thousands to millions of records.
- **Resource usage** - Reduced database load. Efficient queries use less CPU, memory, and I/O, allowing your database to handle more concurrent requests.
- **User experience** - Faster API responses. Optimized queries mean snappy API endpoints that keep users engaged.

Next: [psql-copy-command.md](./psql-copy-command.md)
