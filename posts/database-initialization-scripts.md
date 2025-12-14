<!--
#postgresql #docker #database #initialization #sql #localdevelopment #devops
-->

# Database Initialization Scripts in Docker

## Introduction

Set up automatic database initialization for Docker PostgreSQL containers using SQL scripts that run on first container start. This approach ensures consistent schema and seed data across all development environments without manual setup steps.

## The Problem

When starting a new database container, you need to create schemas, tables, and seed data. The typical approaches involve manually running SQL scripts or connecting to the database after it starts, which is error-prone and doesn't scale across team members.

```bash
# Manual approach - requires manual steps every time
docker-compose up -d
psql -h localhost -U postgres -d your_database -f schema.sql
psql -h localhost -U postgres -d your_database -f seed_data.sql
```

This works, but requires remembering to run scripts in the correct order and doesn't happen automatically when containers are reset.

## The Solution

Instead of manual script execution, we use PostgreSQL's built-in initialization directory that automatically runs SQL files when a container first starts. The architecture flows from SQL files in a mounted directory through PostgreSQL's initialization process to a fully configured database.

### Architecture Overview

SQL Files → /docker-entrypoint-initdb.d → PostgreSQL Initialization → Configured Database

- **db-init directory**: Contains numbered SQL files
- **docker-entrypoint-initdb.d**: PostgreSQL's automatic initialization directory
- **Alphabetical execution**: Files run in order based on filename
- **One-time execution**: Scripts only run on first container start

### Implementation

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15
    volumes:
      - ./db-init:/docker-entrypoint-initdb.d
```

The `db-init` directory contains numbered SQL files that PostgreSQL runs automatically on first container start. Files execute in alphabetical order, which is why they're numbered sequentially.

**Example structure:**
```
db-init/
├── 01_create_schemas.sql
├── 02_create_tables.sql
├── 03_insert_reference_data.sql
└── 04_insert_test_data.sql
```

**Example initialization file:**
```sql
-- 01_create_schemas.sql
-- Note: Scripts run in the default database context
CREATE SCHEMA IF NOT EXISTS public;
CREATE SCHEMA IF NOT EXISTS app;

-- 02_create_tables.sql
CREATE TABLE app.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 03_insert_reference_data.sql
INSERT INTO app.users (email, name) VALUES
    ('admin@example.com', 'Admin User'),
    ('user@example.com', 'Regular User');
```

**Important**: Initialization scripts run in the context of the database specified by `POSTGRES_DB`. To create multiple databases, you'll need to use a shell script wrapper or connect to `postgres` database first.

### Initialization Flow

1. Container starts for the first time
2. PostgreSQL detects files in `/docker-entrypoint-initdb.d`
3. Files execute in alphabetical order
4. Database is ready with schema and data

**Important**: Initialization only happens on the **first start**. After that, data persists in volumes. To reinitialize, remove volumes: `docker-compose down -v && docker-compose up -d`

## Benefits

This approach provides automatic database setup that's consistent across all environments. We get version-controlled schema and seed data that runs automatically without manual intervention. This pattern works well for:

- **Team consistency** - Everyone gets the same database setup automatically
- **Version control** - SQL files are tracked in git
- **Reproducibility** - Fresh database setup is one command away
- **Development workflows** - Reset and reinitialize easily

The clean separation between container configuration and initialization scripts means database setup is automated and reproducible while maintaining full control over schema and data.

This works with the Docker setup (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)) and data persistence (see [docker-data-persistence.md](./docker-data-persistence.md)). Once initialized, connect your application (see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)).
