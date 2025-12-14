<!--
#postgresql #nextjs #api #performance #database #optimization #indexing
-->

# Query Performance Optimization

## Introduction

Optimized PostgreSQL queries in Next.js APIs to improve response times and handle larger datasets efficiently. This approach uses indexes, query analysis, and efficient patterns to reduce database load and improve user experience.

## The Problem

When building API endpoints, queries can become slow as data grows. The typical approaches involve fetching more data than needed or not using indexes, which leads to slow responses and poor scalability.

```typescript
// Inefficient query - no indexes, fetches unnecessary data
const result = await query('SELECT * FROM users WHERE email = $1', [email]);
// Then filters in JavaScript
const activeUsers = result.rows.filter(u => u.active);
```

This works for small datasets, but becomes slow as tables grow and doesn't leverage database optimization.

## The Solution

Instead of relying on JavaScript filtering, we optimize queries at the database level using indexes, efficient query patterns, and query analysis. The architecture flows from query design through index usage to optimized execution.

### Architecture Overview

Query Design → Index Analysis → Optimized Query → Fast Execution

- **Query design**: Structure queries for efficiency
- **Index analysis**: Identify needed indexes
- **Optimized query**: Use indexes and efficient patterns
- **Fast execution**: Database handles optimization

### Implementation

**Adding indexes:**
```sql
-- Create indexes for common query patterns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Composite index for multiple conditions
CREATE INDEX idx_users_active_created ON users(active, created_at DESC);
```

**Efficient query patterns:**
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

**Using EXPLAIN to analyze queries:**
```typescript
// Analyze query performance
const explainResult = await query(
  'EXPLAIN ANALYZE SELECT * FROM users WHERE active = $1 ORDER BY created_at DESC LIMIT 20',
  [true]
);

console.log(explainResult.rows);
// Look for "Seq Scan" (bad) vs "Index Scan" (good)
```

**Selecting only needed columns:**
```typescript
// Instead of SELECT *
const result = await query(
  'SELECT id, email, name FROM users WHERE active = $1',
  [true]
);

// Reduces data transfer and memory usage
```

**Using JOINs efficiently:**
```typescript
// Efficient JOIN with indexed foreign keys
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

**Connection pooling considerations:**
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

### Optimization Techniques

- **Indexes** - Create indexes on frequently queried columns
- **Select specific columns** - Don't use SELECT * in production
- **Limit results** - Always use LIMIT for list endpoints
- **Efficient WHERE clauses** - Filter on indexed columns
- **Avoid N+1 queries** - Use JOINs instead of multiple queries
- **Connection pooling** - Reuse connections efficiently

## Benefits

This approach provides optimized queries that scale well as data grows. We get faster response times, reduced database load, and better user experience. This pattern works well for:

- **Performance** - Faster query execution
- **Scalability** - Handles growing datasets efficiently
- **Resource usage** - Reduced database load
- **User experience** - Faster API responses

The clean separation between query design and execution means endpoints are optimized while maintaining readability and maintainability.

This builds on pagination (see [pagination-strategies.md](./pagination-strategies.md)). Next, see how to import data with CSV files (see [psql-copy-command.md](./psql-copy-command.md)).
