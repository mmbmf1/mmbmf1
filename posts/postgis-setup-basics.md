<!--
#postgis #postgresql #geospatial #docker #database #localdevelopment
-->

# PostGIS Setup and Geospatial Basics

## Introduction

Set up PostGIS extension in PostgreSQL for geospatial data processing. This approach enables spatial queries, coordinate transformations, and geometry operations directly in the database.

## The Problem

When working with location data, you need to store coordinates and perform spatial operations. The typical approaches involve storing lat/lng as separate columns and calculating distances in application code, which is inefficient and doesn't leverage database capabilities.

```sql
-- Basic approach - no spatial capabilities
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION
);

-- Distance calculation in application code
-- const distance = calculateDistance(lat1, lng1, lat2, lng2);
```

This works, but doesn't support spatial queries, coordinate systems, or efficient spatial indexing.

## The Solution

Instead of separate lat/lng columns, we use PostGIS geometry types and spatial functions. The architecture flows from PostGIS-enabled database through geometry storage to spatial queries.

### Architecture Overview

PostGIS Extension → Geometry Columns → Spatial Indexes → Spatial Queries

- **PostGIS extension**: Enables spatial capabilities
- **Geometry columns**: Store spatial data (POINT, POLYGON, etc.)
- **Spatial indexes**: GIST indexes for fast spatial queries
- **Spatial queries**: Distance, intersection, containment operations

### Implementation

**Docker setup with PostGIS:**
```yaml
# docker-compose.yml
services:
  db:
    image: postgis/postgis:15-3.3  # PostGIS-enabled image
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

**Enable PostGIS in database:**
```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Verify installation
SELECT PostGIS_version();
```

**Create table with geometry:**
```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    coordinates GEOMETRY(POINT, 4326)  -- WGS84 coordinate system
);

-- Create spatial index
CREATE INDEX idx_locations_coordinates ON locations USING GIST (coordinates);
```

**Insert spatial data:**
```sql
-- Insert point (longitude, latitude)
INSERT INTO locations (name, coordinates) VALUES
    ('New York', ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326)),
    ('Los Angeles', ST_SetSRID(ST_MakePoint(-118.2437, 34.0522), 4326));
```

**Basic spatial queries:**
```sql
-- Find points within radius
SELECT name, ST_AsText(coordinates) as location
FROM locations
WHERE ST_DWithin(
    coordinates,
    ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326),
    100000  -- 100km in meters
);

-- Calculate distance between points
SELECT 
    a.name as location1,
    b.name as location2,
    ST_Distance(a.coordinates, b.coordinates) / 1000 as distance_km
FROM locations a, locations b
WHERE a.id < b.id;
```

### PostGIS Functions

- **ST_MakePoint(lng, lat)** - Create a point geometry
- **ST_SetSRID(geom, srid)** - Set coordinate system
- **ST_Distance(geom1, geom2)** - Calculate distance
- **ST_DWithin(geom1, geom2, distance)** - Check if within distance
- **ST_AsText(geom)** - Convert geometry to text
- **ST_AsGeoJSON(geom)** - Convert to GeoJSON

## Benefits

This approach provides powerful spatial capabilities directly in the database. We get efficient spatial queries, coordinate system handling, and geometry operations without application code. This pattern works well for:

- **Performance** - Spatial indexes enable fast queries
- **Functionality** - Rich set of spatial operations
- **Standards** - Supports standard coordinate systems
- **Efficiency** - Database handles spatial calculations

The clean separation between spatial data storage and queries means geospatial operations are efficient and maintainable.

This builds on Docker setup (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)). Next, see how to build GeoJSON APIs with PostGIS (see [building-geojson-apis.md](./building-geojson-apis.md)).
