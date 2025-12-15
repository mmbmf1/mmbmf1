<!--
#postgis #nextjs #geojson #database #api #geospatial #postgresql #typescript
-->

# Building a Production-Ready GeoJSON Pipeline with PostGIS + Next.js

## Introduction

Geospatial data pipeline using PostGIS with Next.js API routes. Database-level spatial processing, not JavaScript calculations. This approach leverages PostgreSQL's spatial capabilities to handle complex geographic operations efficiently. Your Next.js API simply formats the results, keeping your application code clean and performant.

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

Use PostGIS database functions to generate GeoJSON at the database level. Create PostgreSQL functions that handle the GeoJSON transformation, keeping your API code simple. For existing PostGIS data, use `ST_AsGeoJSON` and `jsonb_build_object` to construct complete FeatureCollection objects directly in SQL queries.

**API endpoint:**
```typescript
// pages/api/geojson/points.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
  const { data, latField, lngField, properties } = req.body;
  
  const result = await query(`
    SELECT generate_geojson_points($1, $2, $3, $4) as geojson
  `, [JSON.stringify(data), latField, lngField, properties]);
  
  res.json(JSON.parse(result.rows[0].geojson));
}
```

**Database function:**
```sql
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
        
        features := features || jsonb_build_array(
            jsonb_build_object(
                'type', 'Feature',
                'geometry', ST_AsGeoJSON(point)::JSONB,
                'properties', (
                    SELECT jsonb_object_agg(key, value)
                    FROM jsonb_each(item)
                    WHERE key = ANY(properties)
                )
            )
        );
    END LOOP;
    
    RETURN jsonb_build_object('type', 'FeatureCollection', 'features', features);
END;
$$ LANGUAGE plpgsql;
```

**Query existing PostGIS data:**
```typescript
// pages/api/locations/geojson.ts
import { query } from '@/lib/db';

export default async function handler(req, res) {
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
  
  res.json(result.rows[0].geojson);
}
```

## Benefits

- **Mapping applications** - Real-time geospatial data processing. Your API can serve GeoJSON data efficiently for mapping libraries and GIS applications.
- **Data visualization** - Efficient coordinate transformations. PostgreSQL handles all the spatial calculations and formatting, keeping your API endpoints fast and simple.
- **Performance** - Database handles spatial operations. PostGIS processes spatial queries efficiently using indexes, generating GeoJSON without transferring raw coordinate data.
- **Standards** - GeoJSON is widely supported. The GeoJSON format works seamlessly with mapping libraries, GIS tools, and other geospatial applications.

This builds on [nextjs-api-routes.md](./nextjs-api-routes.md) and [postgis-setup-basics.md](./postgis-setup-basics.md). See [building-geojson-apis.md](./building-geojson-apis.md) for more examples.
