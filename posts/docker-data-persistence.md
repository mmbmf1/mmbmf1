<!--
#docker #postgresql #volumes #datapersistence #localdevelopment #devops
-->

# Data Persistence with Docker Volumes

## Introduction

Docker volumes keep PostgreSQL data separate from containers. Data survives restarts, easy to reset when needed.

## The Problem

Container data is lost when containers stop or are removed.

```bash
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
# Data persists
docker-compose down
docker-compose up -d
# Data still there

# Delete data
docker-compose down -v
docker-compose up -d
# Fresh database, initialization scripts run again
```

**Volume management:**
```bash
docker volume ls
docker volume inspect posts_postgres_data
docker volume rm posts_postgres_data
docker volume prune
```

**Backup volume:**
```bash
docker run --rm -v posts_postgres_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/postgres_backup.tar.gz /data
```

## Benefits

- Data persists - Survives restarts
- Easy reset - Remove volumes to start fresh
- Team consistency - Same persistence behavior
- Easy cleanup - One command to reset

Next: [database-initialization-scripts.md](./database-initialization-scripts.md) | [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)
