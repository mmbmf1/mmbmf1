<!--
#postgresql #docker #database #initialization #sql #localdevelopment #devops
-->

# Database Initialization Scripts in Docker

## Introduction

Auto-run SQL scripts on first container start. Consistent schema and seed data without manual steps.

## The Problem

Manual script execution can be error-prone and doesn't scale well across team members.

```bash
docker-compose up -d
psql -h localhost -U postgres -d your_database -f schema.sql
psql -h localhost -U postgres -d your_database -f seed_data.sql
```

## The Solution

Mount SQL files to `/docker-entrypoint-initdb.d`. Files run alphabetically on first start only.

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15
    volumes:
      - ./db-init:/docker-entrypoint-initdb.d
```

**Directory structure:**
```
db-init/
├── 01_create_schemas.sql
├── 02_create_tables.sql
├── 03_insert_reference_data.sql
└── 04_insert_test_data.sql
```

**Example SQL files:**
```sql
-- 01_create_schemas.sql
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

**Important:**
- Scripts run only on first container start
- Files execute in alphabetical order (number them)
- To reinitialize: `docker-compose down -v && docker-compose up -d`
- Scripts run in the `POSTGRES_DB` database context

## Benefits

- Team consistency - Same setup automatically
- Version control - SQL files in git
- Reproducibility - Fresh setup in one command
- Easy reset - Remove volumes and restart

Next: [docker-data-persistence.md](./docker-data-persistence.md) | [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)
