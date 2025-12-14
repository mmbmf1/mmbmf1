<!--
#docker #postgresql #volumes #datapersistence #localdevelopment #devops
-->

# Data Persistence with Docker Volumes

## Introduction

Docker volumes keep PostgreSQL data separate from containers. Data survives restarts, easy to reset when needed.

## The Problem

Container data is lost when containers stop or are removed.

```bash
# Data lost when container stops
docker-compose down
docker-compose up -d
# All data gone
```

## The Solution

Use named volumes to persist data outside containers.

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

**Data lifecycle:**
```bash
# Data persists through restarts
docker-compose down
docker-compose up -d
# Data still there

# Data persists through container removal
docker-compose down
docker-compose rm db
docker-compose up -d
# Data still there

# Delete data (removes volumes)
docker-compose down -v
docker-compose up -d
# Fresh database, initialization scripts run again
```

**Volume management:**
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

**Volume location:**
- Linux: `/var/lib/docker/volumes/`
- macOS/Windows: Managed by Docker Desktop

**Backup volume:**
```bash
# Backup
docker run --rm -v posts_postgres_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/postgres_backup.tar.gz /data

# Restore
docker run --rm -v posts_postgres_data:/data -v $(pwd):/backup \
  alpine tar xzf /backup/postgres_backup.tar.gz -C /
```

## Benefits

- Data persists - Survives restarts
- Easy reset - Remove volumes to start fresh
- Team consistency - Same persistence behavior
- Easy cleanup - One command to reset

Next: [database-initialization-scripts.md](./database-initialization-scripts.md) | [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)
