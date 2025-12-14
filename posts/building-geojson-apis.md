<!--
#postgis #nextjs #geojson #api #geospatial #postgresql #typescript
-->

# Building GeoJSON APIs with PostGIS

## Introduction

Generate GeoJSON from PostGIS geometry data in Next.js APIs. Database-level spatial processing, not JavaScript calculations.

## The Problem

Processing coordinates in JavaScript can be slow for large datasets.

```javascript
const features = data.map(point => ({
  type: "Feature",
  geometry: { type: "Point", coordinates: [point.lng, point.lat] },
  properties: { name: point.name }
}));
```

## The Solution

Use PostGIS `ST_AsGeoJSON` to generate GeoJSON at the database level.

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

- Performance - Database handles spatial operations
- Standards - GeoJSON is widely supported
- Efficiency - No JavaScript processing needed
- Functionality - Full PostGIS spatial capabilities

Next: [pgvector-setup.md](./pgvector-setup.md)
