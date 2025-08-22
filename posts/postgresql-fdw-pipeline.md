<!--
#postgresql #fdw #dataPipeline #sql #database #etl #dataSync #postgres #foreignDataWrapper #dataEngineering #automation #batchProcessing #json #performance #scalability #cron #dataIntegration #realTimeData #localCopy #remoteDatabase
-->

# Building a PostgreSQL FDW Data Pipeline

<!-- #sql/fdw -->

## Introduction

Recently built a data synchronization system using PostgreSQL Foreign Data Wrappers (FDW) to pull data from one database into another for analysis. This approach eliminates the need for traditional ETL tools while providing real-time access to cross-database data.

## Architecture Overview

Source Database → Foreign Server → Foreign Tables → Target Database

- **Source Database:** Database containing the original data
- **Foreign Server:** Connection to source PostgreSQL instance
- **Foreign Tables:** Mappings to source tables in target database
- **Target Database:** Database for analysis and processing

## Getting Started

### 1. Explore the Source Database

```sql
-- List available databases
\l

-- Connect to source database
\c source_database

-- List schemas
\dn

-- List tables in a schema
\dt schema_name.*
```

### 2. Set Up Foreign Server

```sql
-- Create foreign server connection
CREATE SERVER source_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'source-host', port '5432', dbname 'source_database');

-- Create user mapping
CREATE USER MAPPING FOR CURRENT_USER
SERVER source_server
OPTIONS (user 'source_user', password 'source_password');
```

## Foreign Tables Setup

### Example Structure

```sql
CREATE FOREIGN TABLE source_properties (
    id INT,
    property_id VARCHAR,
    -- ... other columns
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION
)
SERVER source_server
OPTIONS (schema_name 'data', table_name 'properties');
```

### Test the Connection

```sql
-- Verify foreign table works
SELECT COUNT(*) FROM source_properties;

-- Check data structure
\d source_properties

-- Sample some data
SELECT * FROM source_properties LIMIT 5;
```

## Data Synchronization

### Basic Sync Pattern

```sql
TRUNCATE TABLE data.properties;
INSERT INTO data.properties SELECT * FROM source_properties;
```

### Batch Processing (For large datasets)

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

### Monitor Progress

```sql
-- Check sync status
SELECT COUNT(*) FROM data.properties;
SELECT COUNT(*) FROM source_properties;

-- Compare row counts
SELECT
    'target' as source, COUNT(*) as count FROM data.properties
UNION ALL
SELECT
    'source' as source, COUNT(*) as count FROM source_properties;
```

## Key Learnings

### 1. FDW Limitations

- **No Constraints:** Foreign tables don't support PRIMARY KEY, FOREIGN KEY, or UNIQUE constraints
- **No Indexes:** Indexes must be created on the source server
- **No Sequences:** Default values with sequences need special handling

### 2. Performance Considerations

- **Batch Processing:** Large datasets benefit from batch inserts
- **Network Latency:** Monitor connection performance
- **Memory Usage:** Consider disabling triggers during bulk operations

### 3. JSON Data Handling

```sql
-- JSON columns work seamlessly with FDW
SELECT * FROM source_services WHERE service_data ? 'key';
SELECT account_id, service_data->>'service_type' FROM source_services;
```

## Troubleshooting

### Common Issues

```sql
-- Check foreign server status
SELECT * FROM pg_foreign_server;

-- Verify user mappings
SELECT * FROM pg_user_mappings;

-- Test connection
SELECT * FROM source_properties LIMIT 1;
```

## Use Cases

This pattern works for various scenarios:

- **Development → Production** data sync
- **Analytics → Operational** database connections
- **Multi-region** database synchronization
- **Legacy → Modern** system migrations
- **Cloud → On-prem** data pipelines

## Benefits Achieved

- **Real-time Access:** Direct querying of cross-database data
- **Performance:** Fast queries on target database
- **No ETL Tools:** Native PostgreSQL solution
- **JSON Support:** Full JSON column functionality
- **Scalable:** Easy to add more tables
- **Flexible:** Works for any two PostgreSQL databases

<!--
future cron job section
## Automation with pg_cron

### Setting Up pg_cron

```sql
-- Enable pg_cron extension
CREATE EXTENSION pg_cron;

-- Grant usage to your user
GRANT USAGE ON SCHEMA cron TO your_username;
```

### Scheduling Data Sync

```sql
-- Sync properties table every hour
SELECT cron.schedule('sync-properties', '0 * * * *',
    'TRUNCATE TABLE data.properties; INSERT INTO data.properties SELECT * FROM source_properties;');

-- Sync subscribers table every 2 hours
SELECT cron.schedule('sync-subscribers', '0 */2 * * *',
    'TRUNCATE TABLE data.subscribers; INSERT INTO data.subscribers SELECT * FROM source_subscribers;');

-- Sync services table daily at 2 AM
SELECT cron.schedule('sync-services', '0 2 * * *',
    'TRUNCATE TABLE data.services; INSERT INTO data.services SELECT * FROM source_services;');
```

### Managing Cron Jobs

```sql
-- List all scheduled jobs
SELECT * FROM cron.job;

-- Check job run history
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;

-- Remove a job
SELECT cron.unschedule('sync-properties');
```

### Error Handling

```sql
-- Check for failed jobs
SELECT
    jobid,
    job_pid,
    database,
    username,
    command,
    return_message,
    start_time,
    end_time
FROM cron.job_run_details
WHERE return_message != 'COMMAND OK'
ORDER BY start_time DESC;
```

### Performance Monitoring

```sql
-- Monitor sync duration
SELECT
    jobid,
    command,
    start_time,
    end_time,
    EXTRACT(EPOCH FROM (end_time - start_time)) as duration_seconds
FROM cron.job_run_details
WHERE return_message = 'COMMAND OK'
ORDER BY start_time DESC;
```
-->
