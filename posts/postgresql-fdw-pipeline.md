<!--
#postgresql #fdw #datapipeline #sql #database #etl #datasync
-->

# PostgreSQL Foreign Data Wrappers (FDW)

## Introduction

Sync data between PostgreSQL databases using Foreign Data Wrappers. Direct cross-database queries without ETL tools. FDWs let you query remote databases as if they were local tables, making data synchronization straightforward. Perfect for keeping multiple databases in sync or building data pipelines.

## The Problem

Manual exports and imports may not provide real-time access and can be hard to maintain. Exporting data to CSV files and importing them into another database creates delays and requires manual intervention. As data changes frequently, keeping databases in sync becomes a constant chore. This approach doesn't scale well and introduces opportunities for errors.

```sql
COPY (SELECT * FROM source_table) TO '/tmp/export.csv';
COPY target_table FROM '/tmp/export.csv';
```

## The Solution

Use Foreign Data Wrappers to query remote databases directly. FDWs create a bridge between databases, letting you query remote tables as if they were local. You can join local and remote tables, sync data efficiently, and even perform real-time queries across database boundaries. All without external ETL tools or manual file transfers.

**Setup:**
```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER source_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'source-host', port '5432', dbname 'source_database');

CREATE USER MAPPING FOR CURRENT_USER
SERVER source_server
OPTIONS (user 'source_user', password 'source_password');

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
SELECT * FROM source_properties WHERE latitude > 40;

SELECT l.name, f.property_id
FROM local_locations l
JOIN source_properties f ON ST_DWithin(l.coords, f.coords, 1000);
```

**Sync data:**
```sql
TRUNCATE TABLE data.properties;
INSERT INTO data.properties SELECT * FROM source_properties;
```

**Batch sync:**
```sql
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

**Cleanup:**
```sql
DROP FOREIGN TABLE source_properties;
DROP USER MAPPING FOR CURRENT_USER SERVER source_server;
DROP SERVER source_server;
```

## Benefits

- **Real-time access** - Query remote data directly without exporting and importing files. Changes in the source database are immediately queryable from your local database.
- **No ETL tools** - Built into PostgreSQL. No need for separate ETL pipelines or external tools - FDWs are a native PostgreSQL feature.
- **Flexible** - Join local and remote tables seamlessly. You can combine data from multiple sources in a single query, enabling powerful cross-database operations.
- **Efficient** - Database-level operations. PostgreSQL handles the networking and data transfer efficiently, making cross-database queries fast and reliable.

Next: [nextjs-api-routes.md](./nextjs-api-routes.md) | [building-geojson-apis.md](./building-geojson-apis.md)
