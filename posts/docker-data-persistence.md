<!--
#docker #postgresql #volumes #datapersistence #localdevelopment #devops
-->

# Data Persistence with Docker Volumes

## Introduction

Docker volumes keep PostgreSQL data separate from containers. Data survives restarts, easy to reset when needed. Volumes are stored on your host machine, completely independent of the container lifecycle. This gives you the best of both worlds: isolated development environments with persistent data.

## The Problem

Container data is lost when containers stop or are removed. By default, everything inside a Docker container is ephemeral - when the container stops, all data disappears. This is fine for stateless services, but databases need their data to persist across restarts. Without volumes, you'd lose all your data every time you restart Docker or update your container.

```bash
docker-compose down
docker-compose up -d
# All data gone
```

## The Solution

Use named volumes to persist data outside containers. Volumes are Docker-managed storage areas that exist independently of containers. When you mount a volume to your PostgreSQL container's data directory, all database files are stored in the volume, not in the container itself. This means your data survives container restarts, updates, and even complete container removal.

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

- **Data persists** - Survives restarts, updates, and container removal. Your database data is safe as long as the volume exists, independent of container lifecycle.
- **Easy reset** - Remove volumes to start fresh. When you need to reset your database, just delete the volume and restart - you get a completely clean database.
- **Team consistency** - Same persistence behavior for everyone. All developers experience the same data persistence patterns, eliminating environment-specific issues.
- **Easy cleanup** - One command to reset. `docker-compose down -v` removes everything, giving you a fresh start whenever you need it.

Next: [database-initialization-scripts.md](./database-initialization-scripts.md) | [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)
