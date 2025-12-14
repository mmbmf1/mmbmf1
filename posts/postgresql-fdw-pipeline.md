<!--
#postgresql #fdw #dataPipeline #sql #database #etl #dataSync #postgres #foreignDataWrapper #dataEngineering #automation #batchProcessing #json #performance #scalability #cron #dataIntegration #realTimeData #localCopy #remoteDatabase
-->

# Building a PostgreSQL FDW Data Pipeline

<!-- #sql/fdw -->

## Introduction

Built a data synchronization system using PostgreSQL Foreign Data Wrappers (FDW) to pull data from one database into another for analysis. This approach eliminates the need for traditional ETL tools while providing real-time access to cross-database data. Works with any PostgreSQL instances, including Docker-based local development (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)).

## The Problem

When working with multiple databases, you need to synchronize data between them for analysis and reporting. Traditional approaches involve complex ETL pipelines, scheduled jobs, or manual data exports that are difficult to maintain and don't provide real-time access.

Here's what that looks like:

```sql
-- Manual approach - inefficient and error-prone
-- Export data from source
COPY (SELECT * FROM source_table) TO '/tmp/export.csv';

-- Import to target
COPY target_table FROM '/tmp/export.csv';

-- Repeat manually or with complex cron jobs
```

This works, but doesn't leverage PostgreSQL's built-in cross-database capabilities.

## The Solution

Instead of manual exports and imports, we built a data pipeline using PostgreSQL Foreign Data Wrappers (FDW) that provides direct access to remote database tables. The architecture flows from source database through foreign server connections to local foreign tables, enabling real-time cross-database queries.

### Architecture Overview

Source Database → Foreign Server → Foreign Tables → Target Database

- **Source Database:** Database containing the original data
- **Foreign Server:** Connection to source PostgreSQL instance  
- **Foreign Tables:** Mappings to source tables in target database
- **Target Database:** Database for analysis and processing

### Implementation

```sql
-- Create foreign server connection
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

### Data Synchronization

```sql
-- Basic sync pattern
TRUNCATE TABLE data.properties;
INSERT INTO data.properties SELECT * FROM source_properties;

-- Batch processing for large datasets
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

## Benefits

This approach provides direct database-level access to cross-database data. We get real-time synchronization and query capabilities without additional infrastructure. This pattern works well for:

- **Development → Production** data sync
- **Analytics → Operational** database connections  
- **Multi-region** database synchronization
- **Legacy → Modern** system migrations
- **Cloud → On-prem** data pipelines

The clean separation between foreign server connections and local processing means the heavy lifting happens in PostgreSQL while maintaining full SQL query capabilities.

Once data is synchronized, you can query it through Next.js API routes (see [nextjs-postgresql-connection.md](./nextjs-postgresql-connection.md)) or process it for specific use cases like geospatial data (see [postgresql-nextjs-geojson-pipeline.md](./postgresql-nextjs-geojson-pipeline.md)).
