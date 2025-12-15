<!--
#postgis #postgresql #geospatial #docker #database #localdevelopment
-->

# PostGIS Setup and Geospatial Basics

<!-- ![PostGIS Spatial Queries](images/postgis-queries.png) -->

## Introduction

PostGIS extension for PostgreSQL. Store geometry data, perform spatial queries, calculate distances in the database. Perfect for applications that need to work with maps, locations, or geographic data. PostGIS brings powerful spatial capabilities directly into your PostgreSQL database.

## The Problem

Separate lat/lng columns don't support spatial queries or efficient spatial indexing. Calculating distances between points requires complex trigonometry in your application code. Finding points within a radius or performing other spatial operations becomes slow and error-prone. You can't leverage spatial indexes that make these operations fast.

```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION
);
```

## The Solution

Use PostGIS geometry types and spatial functions. Store locations as geometry objects instead of separate latitude and longitude columns. PostGIS provides powerful spatial functions for distance calculations, area measurements, and spatial relationships. Spatial indexes (GIST) make these operations incredibly fast, even with millions of points.

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

- **Performance** - Spatial indexes for fast queries. GIST indexes make spatial operations like "find all points within radius" incredibly fast, even with millions of records.
- **Functionality** - Rich spatial operations. Calculate distances, areas, intersections, and more without writing complex application code.
- **Standards** - Standard coordinate systems. PostGIS supports all standard coordinate reference systems, ensuring compatibility with mapping tools and other geospatial systems.
- **Efficiency** - Database handles calculations. Let PostgreSQL do the heavy lifting instead of processing coordinates in your application code.

Next: [building-geojson-apis.md](./building-geojson-apis.md)
