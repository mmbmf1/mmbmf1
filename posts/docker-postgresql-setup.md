<!--
#docker #postgresql #localdevelopment #database #containers #dockercompose #devops
-->

# Setting Up Docker for Local PostgreSQL Development

![Docker PostgreSQL Setup](./images/docker-postgresql-setup.png)

## Introduction

Docker-based PostgreSQL setup for local development. Isolated, easy to reset, consistent across teams.

## The Problem

Installing PostgreSQL directly conflicts with system installations and makes cleanup difficult.

```bash
# Manual approach - conflicts with system PostgreSQL
brew install postgresql
initdb /usr/local/var/postgres
pg_ctl start
createdb your_database
```

## The Solution

Use Docker Compose to run PostgreSQL in a container.

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
# Start
docker-compose up -d

# Check status
docker ps | grep postgres_dev_db

# View logs
docker-compose logs -f db

# Stop
docker-compose down

# Reset (removes data)
docker-compose down -v && docker-compose up -d
```

**Connection:**
- Host: `localhost`
- Port: `5432`
- User: `postgres`
- Password: `postgres`
- Database: `your_database`

## Benefits

- Team consistency - Same setup everywhere
- Easy cleanup - Reset with one command
- No system conflicts - Isolated container
- Quick setup - One command to start

Next: [database-initialization-scripts.md](./database-initialization-scripts.md) | [docker-data-persistence.md](./docker-data-persistence.md) | [postgresql-connection-pooling.md](./postgresql-connection-pooling.md)
