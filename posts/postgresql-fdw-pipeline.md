<!--
#postgresql #fdw #datapipeline #sql #database #etl #datasync
-->

# PostgreSQL Foreign Data Wrappers (FDW)

## Introduction

Sync data between PostgreSQL databases using Foreign Data Wrappers. Direct cross-database queries without ETL tools.

## The Problem

Manual exports and imports don't provide real-time access and are hard to maintain.

```sql
-- Manual approach - inefficient
COPY (SELECT * FROM source_table) TO '/tmp/export.csv';
COPY target_table FROM '/tmp/export.csv';
```

## The Solution

Use Foreign Data Wrappers to query remote databases directly.

**Setup:**
```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Create foreign server
CREATE SERVER source_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'source-host', port '5432', dbname 'source_database');

-- Create user mapping
CREATE USER MAPPING FOR CURRENT_USER
SERVER source_server
OPTIONS (user 'source_user', password 'source_password');

-- Create foreign table
CREATE FOREIGN TABLE source_properties (
    id INT,
    property_id VARCHAR,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION
)
SERVER source_server
OPTIONS (schema_name 'data', table_name 'properties');
```

**Query foreign table:**
```sql
-- Direct query
SELECT * FROM source_properties WHERE latitude > 40;

-- Join with local tables
SELECT 
    l.name,
    f.property_id
FROM local_locations l
JOIN source_properties f ON ST_DWithin(l.coords, f.coords, 1000);
```

**Sync data:**
```sql
-- Basic sync
TRUNCATE TABLE data.properties;
INSERT INTO data.properties SELECT * FROM source_properties;

-- Batch sync for large datasets
DO $$
DECLARE
    batch_size INTEGER := 5000;
    offset_val INTEGER := 0;
    total_rows INTEGER;
BEGIN
    SELECT COUNT(*) INTO total_rows FROM source_properties;
    TRUNCATE TABLE data.properties;

    WHILE offset_val < total_rows LOOP
        INSERT INTO data.properties
        SELECT * FROM source_properties
        LIMIT batch_size OFFSET offset_val;
        offset_val := offset_val + batch_size;
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

**Update foreign table:**
```sql
-- Refresh foreign table definition
DROP FOREIGN TABLE source_properties;
CREATE FOREIGN TABLE source_properties (...)
SERVER source_server
OPTIONS (schema_name 'data', table_name 'properties');
```

**Cleanup:**
```sql
-- Drop foreign table
DROP FOREIGN TABLE source_properties;

-- Drop user mapping
DROP USER MAPPING FOR CURRENT_USER SERVER source_server;

-- Drop server
DROP SERVER source_server;
```

## Benefits

- Real-time access - Query remote data directly
- No ETL tools - Built into PostgreSQL
- Flexible - Join local and remote tables
- Efficient - Database-level operations

Next: [nextjs-api-routes.md](./nextjs-api-routes.md) | [building-geojson-apis.md](./building-geojson-apis.md)
