<!--
#postgis #nextjs #geojson #api #geospatial #postgresql #typescript
-->

# Building GeoJSON APIs with PostGIS

## Introduction

Built Next.js API routes that generate GeoJSON from PostGIS geometry data. This approach uses database-level spatial processing instead of JavaScript calculations, providing efficient geospatial data APIs.

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

This works, but doesn't use the spatial processing capabilities that databases provide and becomes slow with large datasets.

## The Solution

Instead of processing data in JavaScript, we use PostGIS functions to generate GeoJSON at the database level. The architecture flows from PostGIS geometry data through ST_AsGeoJSON functions to standardized GeoJSON responses.

### Architecture Overview

PostGIS Data → ST_AsGeoJSON → Next.js API → GeoJSON Response

- **PostGIS data**: Geometry columns in database
- **ST_AsGeoJSON**: PostGIS function converts to GeoJSON
- **Next.js API**: Formats response
- **GeoJSON response**: Standard geospatial format

### Implementation

**Database function for GeoJSON:**
```sql
-- Create function to generate GeoJSON FeatureCollection
CREATE OR REPLACE FUNCTION generate_geojson_points(
    data_json JSONB,
    lat_field TEXT,
    lng_field TEXT,
    properties TEXT[]
) RETURNS JSONB AS $$
DECLARE
    features JSONB := '[]'::JSONB;
    item JSONB;
    point GEOMETRY;
    feature JSONB;
BEGIN
    FOR item IN SELECT * FROM jsonb_array_elements(data_json)
    LOOP
        point := ST_SetSRID(
            ST_MakePoint(
                (item->>lng_field)::DOUBLE PRECISION,
                (item->>lat_field)::DOUBLE PRECISION
            ),
            4326
        );
        
        feature := jsonb_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(point)::JSONB,
            'properties', (
                SELECT jsonb_object_agg(key, value)
                FROM jsonb_each(item)
                WHERE key = ANY(properties)
            )
        );
        
        features := features || jsonb_build_array(feature);
    END LOOP;
    
    RETURN jsonb_build_object(
        'type', 'FeatureCollection',
        'features', features
    );
END;
$$ LANGUAGE plpgsql;
```

**Next.js API route:**
```typescript
// pages/api/geojson/points.ts
import { query } from '@/lib/db';
import { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { data, latField, lngField, properties } = req.body;
  
  try {
    const result = await query(
      `SELECT generate_geojson_points($1, $2, $3, $4) as geojson`,
      [
        JSON.stringify(data),
        latField,
        lngField,
        properties
      ]
    );
    
    res.json(JSON.parse(result.rows[0].geojson));
  } catch (error) {
    console.error('Database error:', error);
    res.status(500).json({ error: 'Failed to generate GeoJSON' });
  }
}
```

**Querying existing PostGIS data:**
```typescript
// pages/api/locations/geojson.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { bounds, radius, center } = req.query;
  
  let sql = `
    SELECT 
      jsonb_build_object(
        'type', 'FeatureCollection',
        'features', jsonb_agg(
          jsonb_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(coordinates)::JSONB,
            'properties', jsonb_build_object(
              'id', id,
              'name', name
            )
          )
        )
      ) as geojson
    FROM locations
    WHERE 1=1
  `;
  
  const params: any[] = [];
  let paramCount = 0;
  
  if (center) {
    const [lng, lat] = center.split(',').map(parseFloat);
    paramCount++;
    sql += ` AND ST_DWithin(
      coordinates,
      ST_SetSRID(ST_MakePoint($${paramCount}, $${paramCount + 1}), 4326),
      $${paramCount + 2}
    )`;
    params.push(lng, lat, radius || 10000);
    paramCount += 2;
  }
  
  const result = await query(sql, params);
  res.json(result.rows[0].geojson);
}
```

**App Router example:**
```typescript
// app/api/locations/geojson/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const bounds = searchParams.get('bounds');
  
  const result = await query(`
    SELECT 
      jsonb_build_object(
        'type', 'FeatureCollection',
        'features', jsonb_agg(
          jsonb_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(coordinates)::JSONB,
            'properties', jsonb_build_object(
              'id', id,
              'name', name
            )
          )
        )
      ) as geojson
    FROM locations
  `);
  
  return NextResponse.json(result.rows[0].geojson);
}
```

## Benefits

This approach uses PostGIS spatial indexes and geodetic calculations efficiently. We get coordinate system handling and geometry validation without additional code. This pattern works well for:

- **Mapping applications** - Real-time geospatial data processing
- **Data visualization** - Efficient coordinate transformations
- **Performance** - Database handles spatial operations
- **Standards** - GeoJSON is widely supported

The clean separation between the API layer and database processing means the heavy lifting happens in PostGIS while Next.js handles the HTTP interface.

This builds on PostGIS setup (see [postgis-setup-basics.md](./postgis-setup-basics.md)) and API routes (see [nextjs-api-routes.md](./nextjs-api-routes.md)). For other specialized extensions, see how to set up vector search with pgvector (see [pgvector-setup.md](./pgvector-setup.md)).
