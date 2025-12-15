<!--
#postgis #nextjs #geojson #api #geospatial #postgresql #typescript
-->

# Building GeoJSON APIs with PostGIS

## Introduction

Generate GeoJSON from PostGIS geometry data in Next.js APIs. Database-level spatial processing, not JavaScript calculations. PostgreSQL handles the heavy lifting of spatial operations, and your API simply formats the results. This approach scales much better than processing coordinates in your application code.

## The Problem

Processing coordinates in JavaScript can be slow for large datasets. When you fetch raw coordinate data and transform it to GeoJSON in your application, you're doing work that PostgreSQL could handle much more efficiently. Large datasets mean transferring unnecessary data over the network and processing it in memory, which becomes a bottleneck as data grows.

```javascript
const features = data.map(point => ({
  type: "Feature",
  geometry: { type: "Point", coordinates: [point.lng, point.lat] },
  properties: { name: point.name }
}));
```

## The Solution

Use PostGIS `ST_AsGeoJSON` to generate GeoJSON at the database level. PostgreSQL's `ST_AsGeoJSON` function converts geometry data directly to GeoJSON format, and `jsonb_build_object` lets you construct complete FeatureCollection objects in SQL. This means your API just returns the formatted JSON without any JavaScript processing.

**Query existing PostGIS data:**
```typescript
// pages/api/locations/geojson.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { center, radius } = req.query;
  
  let sql = `
    SELECT jsonb_build_object(
      'type', 'FeatureCollection',
      'features', jsonb_agg(
        jsonb_build_object(
          'type', 'Feature',
          'geometry', ST_AsGeoJSON(coordinates)::JSONB,
          'properties', jsonb_build_object('id', id, 'name', name)
        )
      )
    ) as geojson
    FROM locations WHERE 1=1
  `;
  
  const params: any[] = [];
  let paramCount = 0;
  
  if (center) {
    const [lng, lat] = center.split(',').map(parseFloat);
    paramCount++;
    sql += ` AND ST_DWithin(coordinates, ST_SetSRID(ST_MakePoint($${paramCount}, $${paramCount + 1}), 4326), $${paramCount + 2})`;
    params.push(lng, lat, radius || 10000);
    paramCount += 2;
  }
  
  const result = await query(sql, params);
  res.json(result.rows[0].geojson);
}
```

**App Router:**
```typescript
// app/api/locations/geojson/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET() {
  const result = await query(`
    SELECT jsonb_build_object(
      'type', 'FeatureCollection',
      'features', jsonb_agg(
        jsonb_build_object(
          'type', 'Feature',
          'geometry', ST_AsGeoJSON(coordinates)::JSONB,
          'properties', jsonb_build_object('id', id, 'name', name)
        )
      )
    ) as geojson
    FROM locations
  `);
  
  return NextResponse.json(result.rows[0].geojson);
}
```

**Filter by bounds:**
```typescript
const { minLng, minLat, maxLng, maxLat } = req.query;

const result = await query(`
  SELECT jsonb_build_object(
    'type', 'FeatureCollection',
    'features', jsonb_agg(
      jsonb_build_object(
        'type', 'Feature',
        'geometry', ST_AsGeoJSON(coordinates)::JSONB,
        'properties', jsonb_build_object('id', id, 'name', name)
      )
    )
  ) as geojson
  FROM locations
  WHERE ST_Within(coordinates, ST_MakeEnvelope($1, $2, $3, $4, 4326))
`, [minLng, minLat, maxLng, maxLat]);
```

## Benefits

- **Performance** - Database handles spatial operations. PostGIS processes spatial queries efficiently using indexes, and generates GeoJSON without transferring raw coordinate data.
- **Standards** - GeoJSON is widely supported. The GeoJSON format works seamlessly with mapping libraries, GIS tools, and other geospatial applications.
- **Efficiency** - No JavaScript processing needed. PostgreSQL does all the formatting work, keeping your API code simple and fast.
- **Functionality** - Full PostGIS spatial capabilities. You can filter by bounds, radius, or any spatial relationship while generating GeoJSON in the same query.

Next: [pgvector-setup.md](./pgvector-setup.md)
