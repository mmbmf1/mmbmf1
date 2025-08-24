<!--
#postgis #nextjs #geojson #database #api #geospatial #postgresql #typescript #performance #spatial #mapping #webapi #databasefunctions #fullstack #modernweb #spatialdata #geoutils
-->

# Building a Production-Ready GeoJSON Pipeline with PostGIS + Next.js

## Introduction

Built a geospatial data pipeline using PostgreSQL's PostGIS extension with Next.js API routes to generate GeoJSON from coordinate data. This approach uses database-level spatial processing instead of JavaScript calculations.

## The Problem

When building mapping applications, you need to convert coordinate data into GeoJSON format. The typical approach processes this data in JavaScript, which works for small datasets but becomes inefficient as you scale to thousands of points.

```javascript
// JavaScript approach - inefficient for large datasets
const features = data.map(point => ({
  type: "Feature",
  geometry: {
    type: "Point", 
    coordinates: [point.lng, point.lat]
  },
  properties: { name: point.name }
}));
```

This works, but doesn't use the spatial processing capabilities that databases provide.

## The Solution

Instead of processing data in JavaScript, we built custom PostGIS database functions that handle the spatial operations at the database level. The architecture flows from coordinate data through Next.js API routes to PostGIS functions, producing standardized GeoJSON responses.

### API Design

```typescript
// Clean, type-safe API endpoints
POST /api/geojson/points
{
  "data": [
    {"name": "NYC", "lat": 40.7128, "lng": -74.006, "population": 8336817}
  ],
  "latField": "lat",
  "lngField": "lng", 
  "properties": ["name", "population"]
}

// Returns GeoJSON FeatureCollection
{
  "type": "FeatureCollection",
  "features": [...]
}
```

### Implementation

```typescript
// Next.js API route implementation
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { data, latField, lngField, properties } = req.body;
  
  const result = await db.query(`
    SELECT generate_geojson_points($1, $2, $3, $4) as geojson
  `, [JSON.stringify(data), latField, lngField, properties]);
  
  res.json(JSON.parse(result.rows[0].geojson));
}
```

## Benefits

This approach uses PostGIS spatial indexes and geodetic calculations. We get coordinate system handling and geometry validation without additional code. This pattern works well for:

- **Mapping applications** - Real-time geospatial data processing
- **Data visualization projects** - Efficient coordinate transformations  
- **Real-time applications** - Processing location data streams

The clean separation between the API layer and database processing means the heavy lifting happens in PostGIS while Next.js handles the HTTP interface.