<!--
#postgis #postgresql #geospatial #docker #database #localdevelopment
-->

# PostGIS Setup and Geospatial Basics

![PostGIS Spatial Queries](./images/postgis-queries.png)

## Introduction

PostGIS extension for PostgreSQL. Store geometry data, perform spatial queries, calculate distances in the database.

## The Problem

Separate lat/lng columns don't support spatial queries or efficient indexing.

```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION
);
```

## The Solution

Use PostGIS geometry types and spatial functions.

**Docker setup:**
```yaml
# docker-compose.yml
services:
  db:
    image: postgis/postgis:15-3.3
    container_name: postgres_dev_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: your_database
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

**Enable PostGIS:**
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
SELECT PostGIS_version();
```

**Create table with geometry:**
```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    coordinates GEOMETRY(POINT, 4326)
);

CREATE INDEX idx_locations_coordinates ON locations USING GIST (coordinates);
```

**Insert spatial data:**
```sql
INSERT INTO locations (name, coordinates) VALUES
    ('New York', ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326)),
    ('Los Angeles', ST_SetSRID(ST_MakePoint(-118.2437, 34.0522), 4326));
```

**Spatial queries:**
```sql
-- Points within radius
SELECT name, ST_AsText(coordinates) as location
FROM locations
WHERE ST_DWithin(coordinates, ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326), 100000);

-- Distance between points
SELECT a.name, b.name, ST_Distance(a.coordinates, b.coordinates) / 1000 as distance_km
FROM locations a, locations b
WHERE a.id < b.id;

-- Convert to GeoJSON
SELECT name, ST_AsGeoJSON(coordinates) as geojson FROM locations;
```

**PostGIS functions:**
- `ST_MakePoint(lng, lat)` - Create point
- `ST_SetSRID(geom, srid)` - Set coordinate system
- `ST_Distance(geom1, geom2)` - Calculate distance
- `ST_DWithin(geom1, geom2, distance)` - Within distance check
- `ST_AsGeoJSON(geom)` - Convert to GeoJSON

## Benefits

- Performance - Spatial indexes for fast queries
- Functionality - Rich spatial operations
- Standards - Standard coordinate systems
- Efficiency - Database handles calculations

Next: [building-geojson-apis.md](./building-geojson-apis.md)
