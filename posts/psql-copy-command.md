<!--
#postgresql #csv #import #database #etl #data
-->

# CSV Import with psql \copy

## Introduction

Import large CSV files directly into PostgreSQL using `\copy`. Fast, reliable, no external tools needed. This native PostgreSQL command handles large imports efficiently and works seamlessly with your existing database setup. Much faster than GUI tools or custom import scripts.

## The Problem

GUI tools and custom scripts can be slow for large CSV imports. Looking for a simpler approach. GUI database tools often struggle with large files, timing out or consuming excessive memory. Custom import scripts add complexity and maintenance overhead. You need something fast, reliable, and built into PostgreSQL itself.

## The Solution

Use psql's `\copy` command for direct database-level CSV import. This native PostgreSQL command reads CSV files directly and inserts data efficiently. It handles headers, delimiters, and data types automatically, making imports straightforward. Works seamlessly with Docker containers and local PostgreSQL installations.

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

- **Fast** - Direct database import without intermediate processing. PostgreSQL reads and inserts data efficiently, handling large files without memory issues.
- **Reliable** - Native PostgreSQL support means no external dependencies or compatibility issues. Works consistently across different environments.
- **Simple** - One command to import entire CSV files. No complex scripts or GUI tools needed - just a straightforward command-line operation.
- **Flexible** - Works with any CSV format. You can specify delimiters, handle headers, select specific columns, and even export data using the same command.

Next: [postgresql-fdw-pipeline.md](./postgresql-fdw-pipeline.md) | [nextjs-api-routes.md](./nextjs-api-routes.md)
