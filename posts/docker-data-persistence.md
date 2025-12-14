<!--
#docker #postgresql #volumes #datapersistence #localdevelopment #devops
-->

# Data Persistence with Docker Volumes

## Introduction

Configured Docker volumes for PostgreSQL data persistence so database contents survive container restarts and removals. This approach keeps data separate from container lifecycle while maintaining easy reset capabilities.

## The Problem

When you stop or remove a Docker container, all data inside it is lost by default. For databases, you need data to persist across container restarts, but you also want the ability to reset everything when needed. The typical approaches involve managing data directories manually or losing data on every container restart.

```bash
# Problem: Data lost when container stops
docker-compose down
docker-compose up -d
# All data is gone
```

This works for stateless containers, but databases need persistence while maintaining the ability to reset.

## The Solution

Instead of storing data inside the container, we use Docker volumes to store database data outside the container filesystem. The architecture flows from container configuration through volume mapping to persistent data storage.

### Architecture Overview

Container → Volume Mapping → Persistent Storage → Data Survives Restarts

- **Volume declaration**: Named volume in docker-compose.yml
- **Volume mapping**: Container directory mapped to volume
- **Persistent storage**: Data stored on host filesystem
- **Lifecycle independence**: Data survives container removal

### Implementation

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

The `postgres_data` volume stores all database files in Docker's managed storage, separate from the container.

### Data Lifecycle

**Data persists:**
```bash
# Stop container - data stays
docker-compose down

# Start again - data is still there
docker-compose up -d

# Restart container - data persists
docker-compose restart db
```

**Data deleted:**
```bash
# Remove volumes - deletes all data
docker-compose down -v

# Start fresh - initialization scripts run again
docker-compose up -d
```

### Volume Management

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect posts_postgres_data

# Remove specific volume
docker volume rm posts_postgres_data

# Remove all unused volumes
docker volume prune
```

## Benefits

This approach provides controlled data persistence that balances durability with flexibility. We get data that survives restarts while maintaining easy reset capabilities. This pattern works well for:

- **Development workflows** - Data persists between coding sessions
- **Testing scenarios** - Reset to clean state when needed
- **Team consistency** - Same data persistence behavior everywhere
- **Easy cleanup** - One command to reset everything

The clean separation between container lifecycle and data storage means you can restart, update, or reset containers without losing data unless explicitly intended.

This works with Docker setup (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)) and initialization scripts (see [database-initialization-scripts.md](./database-initialization-scripts.md)). Once data is configured, connect your application (see [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)).
