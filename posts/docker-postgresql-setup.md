<!--
#docker #postgresql #localdevelopment #database #containers #dockercompose #devops
-->

# Setting Up Docker for Local PostgreSQL Development

## Introduction

Set up a Docker-based PostgreSQL development environment for local API development. This approach keeps the database containerized and isolated from system PostgreSQL installations while providing persistent data storage. This is the foundation for connecting applications (see [nextjs-postgresql-connection.md](./nextjs-postgresql-connection.md)) and importing data (see [psql-copy-command.md](./psql-copy-command.md)).

## The Problem

When developing locally, you need a PostgreSQL database that matches production but doesn't interfere with system databases. The typical approaches involve installing PostgreSQL directly on your machine or manually managing database instances, which can conflict with existing installations and make cleanup difficult.

```bash
# Manual approach - can conflict with system PostgreSQL
brew install postgresql
initdb /usr/local/var/postgres
pg_ctl start
createdb your_database
```

This works, but ties database setup to your local machine and makes it harder to share consistent environments across the team.

## The Solution

Instead of installing PostgreSQL directly, we containerized the database using Docker Compose with an initialization system that loads schema and mock data automatically. The architecture flows from docker-compose configuration through initialization scripts to a running PostgreSQL container.

### Architecture Overview

docker-compose.yml → PostgreSQL Container → Initialization Scripts → Persistent Volume

- **docker-compose.yml**: Defines the database service with PostgreSQL image
- **db-init directory**: SQL files that run automatically on first container start
- **Persistent volumes**: Data survives container restarts
- **Port mapping**: Database accessible on localhost:5432

### Implementation

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15
    container_name: postgres_dev_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db-init:/docker-entrypoint-initdb.d
```

**Note**: For specialized extensions, use alternative images like `postgis/postgis:15-3.3` for geospatial data, `timescale/timescaledb` for time-series data, or other PostgreSQL variants as needed. The standard `postgres` image works for most use cases.

The `db-init` directory contains numbered SQL files (01-24) that PostgreSQL runs automatically on first container start. Files execute in alphabetical order, which is why they're numbered sequentially.

**Initialization flow:**
1. `01_create_databases.sql` - Creates databases
2. `02-11` - Creates schemas and core tables
3. `12_init_client_data.sql` - Inserts mock/test data
4. `13-24` - Additional tables and reference data

### Data Persistence

Docker volumes store database data outside the container. This means data survives `docker-compose down` but gets deleted with `docker-compose down -v`. The volume approach provides persistence without tying data to the container lifecycle.

```bash
# Start database
docker-compose up -d

# Stop (data persists)
docker-compose down

# Reset everything (data deleted)
docker-compose down -v && docker-compose up -d
```

## Benefits

This approach provides isolated database environments that are easy to reset and share. We get consistent PostgreSQL instances without system-level conflicts. This pattern works well for:

- **Team consistency** - Everyone runs the same database setup
- **Easy cleanup** - Reset with one command when things go wrong
- **No system conflicts** - Containerized database doesn't interfere with system PostgreSQL
- **Development workflows** - Mock data loads automatically on first start

The clean separation between container configuration and initialization scripts means database setup is version-controlled and reproducible while maintaining full PostgreSQL capabilities.

Once your database is running, you can connect to it from Next.js API routes (see [nextjs-postgresql-connection.md](./nextjs-postgresql-connection.md)) or import data using tools like `psql \copy` (see [psql-copy-command.md](./psql-copy-command.md)).
