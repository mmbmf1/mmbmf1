# psql \copy: CSV import that actually works

## Introduction

Had an 83MB CSV with 195k+ rows that needed to go into PostgreSQL. Didn't know psql had a `\copy` command, so I wanted to learn how to use it.

## The Problem

When working with large CSV files, you need to get data into PostgreSQL for analysis. The typical approaches involve GUI tools or complex setup that can be slow and unreliable.

```sql
-- Manual approach - inefficient and error-prone
-- Using pgAdmin or other GUI tools
-- Or writing custom import scripts
```

This works for small files, but becomes problematic with larger datasets.

## The Solution

Instead of external tools, we use psql's `\copy` command that provides direct database-level CSV import capabilities.

```sql
\copy staging_table FROM '/path/to/file.csv' WITH (FORMAT csv, HEADER true);
```

That's it. No GUI, no complex setup, just works.

### Implementation

```sql
-- Create table structure to match CSV
CREATE TABLE staging_table (
    id SERIAL PRIMARY KEY,
    column1 VARCHAR,
    column2 INTEGER,
    column3 TIMESTAMP
);

-- Import data with one command
\copy staging_table FROM '/path/to/file.csv' WITH (FORMAT csv, HEADER true);

-- Verify import
SELECT COUNT(*) FROM staging_table;
```

### Scripting the Workflow

```bash
# Create table, import data, run queries - all in one go
psql -d your_db -c "CREATE TABLE staging_table (...);"
psql -d your_db -c "\copy staging_table FROM 'file.csv' WITH (FORMAT csv, HEADER true);"
psql -d your_db -c "SELECT COUNT(*) FROM staging_table;"
```

## Benefits

This approach uses PostgreSQL's native CSV processing capabilities. We get efficient large file handling and direct database integration without additional tools. This pattern works well for:

- **Data analysis projects** - Quick CSV to database conversion
- **ETL pipelines** - Reliable data import processes  
- **Development workflows** - Fast data setup for testing
- **Data migration** - Moving CSV data into production systems

The clean integration with PostgreSQL means the heavy lifting happens at the database level while maintaining full SQL query capabilities.
