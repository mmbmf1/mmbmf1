<!--
#docker #postgresql #localdevelopment #database #containers #dockercompose #devops
-->

# Setting Up Docker for Local PostgreSQL Development

## Introduction

Set up a Docker-based PostgreSQL development environment for local API development. This approach keeps the database containerized and isolated from system PostgreSQL installations, making it easy to start, stop, and reset without affecting your system.

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

Instead of installing PostgreSQL directly, we containerized the database using Docker Compose. The architecture flows from docker-compose configuration to a running PostgreSQL container accessible on localhost.

### Architecture Overview

docker-compose.yml → PostgreSQL Container → localhost:5432

- **docker-compose.yml**: Defines the database service configuration
- **PostgreSQL container**: Isolated database instance
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
      POSTGRES_DB: your_database
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

**Note**: For specialized extensions, use alternative images like `postgis/postgis:15-3.3` for geospatial data, `timescale/timescaledb` for time-series data, or other PostgreSQL variants as needed. The standard `postgres` image works for most use cases.

### Starting the Database

```bash
# Start database container
docker-compose up -d

# Verify it's running
docker ps | grep postgres_dev_db

# View logs
docker-compose logs -f db

# Stop database
docker-compose down
```

The database is now accessible at `localhost:5432` with username `postgres` and password `postgres`.

## Benefits

This approach provides isolated database environments that are easy to reset and share. We get consistent PostgreSQL instances without system-level conflicts. This pattern works well for:

- **Team consistency** - Everyone runs the same database setup
- **Easy cleanup** - Reset with one command when things go wrong
- **No system conflicts** - Containerized database doesn't interfere with system PostgreSQL
- **Quick setup** - One command to get a running database

The clean separation between container configuration and your application means database setup is version-controlled and reproducible while maintaining full PostgreSQL capabilities.

Once your database is running, you can set up initialization scripts (see [database-initialization-scripts.md](./database-initialization-scripts.md)) to automatically load schema and data, or configure data persistence (see [docker-data-persistence.md](./docker-data-persistence.md)). Then connect your Next.js application (see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)).
