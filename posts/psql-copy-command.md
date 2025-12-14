<!--
#postgresql #csv #import #database #etl #data
-->

# CSV Import with psql \copy

## Introduction

Import large CSV files directly into PostgreSQL using `\copy`. Fast, reliable, no external tools needed.

## The Problem

GUI tools and custom scripts can be slow for large CSV imports. Looking for a simpler approach.

## The Solution

Use psql's `\copy` command for direct database-level CSV import.

**Basic import:**
```sql
CREATE TABLE staging_table (
    id SERIAL PRIMARY KEY,
    column1 VARCHAR,
    column2 INTEGER,
    column3 TIMESTAMP
);

\copy staging_table FROM '/path/to/file.csv' WITH (FORMAT csv, HEADER true);
SELECT COUNT(*) FROM staging_table;
```

**Command line:**
```bash
psql -d your_database -c "\copy staging_table FROM '/path/to/file.csv' WITH (FORMAT csv, HEADER true);"
```

**Options:**
```sql
\copy table FROM 'file.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',');
\copy table (col1, col2, col3) FROM 'file.csv' WITH (FORMAT csv, HEADER true);
\copy table TO '/path/to/output.csv' WITH (FORMAT csv, HEADER true);
```

**Docker container:**
```bash
docker cp file.csv container_name:/tmp/file.csv
docker exec -i container_name psql -U postgres -d your_database -c "\copy staging_table FROM '/tmp/file.csv' WITH (FORMAT csv, HEADER true);"
```

**Common issues:**
- File path must be accessible to PostgreSQL server
- Use `COPY` (server-side) or `\copy` (client-side)
- Ensure table structure matches CSV columns

## Benefits

- Fast - Direct database import
- Reliable - Native PostgreSQL support
- Simple - One command
- Flexible - Works with any CSV format

Next: [postgresql-fdw-pipeline.md](./postgresql-fdw-pipeline.md) | [nextjs-api-routes.md](./nextjs-api-routes.md)
