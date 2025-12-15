<!--
#docker #postgresql #localdevelopment #database #containers #dockercompose #devops
-->

# Setting Up Docker for Local PostgreSQL Development

<!-- ![Docker PostgreSQL Setup](images/docker-postgresql-setup.png) -->

## Introduction

Docker-based PostgreSQL setup for local development. Isolated, easy to reset, consistent across teams. Perfect for developers who want a clean database environment without modifying their system. Works seamlessly with Docker Compose for simple container management.

## The Problem

Installing PostgreSQL directly can conflict with system installations and make cleanup more difficult. Different team members might have different versions installed, leading to inconsistent development environments. When you need to reset or remove PostgreSQL, it can leave behind configuration files and data directories that are hard to track down.

```bash
brew install postgresql
initdb /usr/local/var/postgres
pg_ctl start
createdb your_database
```

## The Solution

Use Docker Compose to run PostgreSQL in a container. This keeps your database completely isolated from your system and makes it easy to start, stop, and reset. The configuration is version-controlled in a simple YAML file that anyone on your team can use to get the exact same setup.

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

**Specialized images:**
- `postgis/postgis:15-3.3` - Geospatial data
- `ankane/pgvector:latest` - Vector embeddings
- `timescale/timescaledb` - Time-series data

**Commands:**
```bash
docker-compose up -d
docker ps | grep postgres_dev_db
docker-compose logs -f db
docker-compose down
docker-compose down -v && docker-compose up -d
```

**Connection:**
- Host: `localhost`
- Port: `5432`
- User: `postgres`
- Password: `postgres`
- Database: `your_database`

## Benefits

- **Team consistency** - Same setup everywhere. Everyone runs the same PostgreSQL version with identical configuration, eliminating "works on my machine" issues.
- **Easy cleanup** - Reset with one command. When you need a fresh database, just remove the volume and restart. No manual cleanup of system files needed.
- **No system conflicts** - Isolated container. Your Docker PostgreSQL instance won't interfere with any system-level PostgreSQL installations or other services.
- **Quick setup** - One command to start. New team members can be up and running in seconds, not minutes or hours.

Next: [database-initialization-scripts.md](./database-initialization-scripts.md) | [docker-data-persistence.md](./docker-data-persistence.md) | [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)
