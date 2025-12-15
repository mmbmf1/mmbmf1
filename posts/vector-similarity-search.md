<!--
#pgvector #postgresql #vectors #similarity #semanticsearch #database #nextjs
-->

# Vector Similarity Search Patterns

## Introduction

Vector similarity search queries with pgvector. Find semantically similar content using cosine similarity and vector indexes. This enables powerful features like finding similar documents, recommendations, and semantic search. All processed efficiently in the database without moving large amounts of data to your application.

## The Problem

Application-level similarity calculation can be slow and may not scale well for large datasets. Fetching all embeddings and calculating similarities in JavaScript means transferring massive amounts of data over the network. As your dataset grows, this approach becomes completely impractical. Vector operations are computationally expensive, and doing them in your application code doesn't leverage database optimizations.

```typescript
const queryEmbedding = await generateEmbedding(query);
const allDocs = await query('SELECT id, content, embedding FROM documents');
const similarities = allDocs.map(doc => ({
  ...doc,
  similarity: cosineSimilarity(queryEmbedding, doc.embedding)
}));
const results = similarities.sort((a, b) => b.similarity - a.similarity).slice(0, 10);
```

## The Solution

Use pgvector similarity operators at the database level. Let PostgreSQL calculate similarities using optimized vector operations and indexes. The `<=>` operator calculates cosine distance efficiently, and vector indexes make these operations fast even with millions of vectors. You can combine vector search with traditional keyword search for hybrid approaches.

**Basic similarity search:**
```typescript
// pages/api/search.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';

export default async function handler(req, res) {
  const { q, limit = 10 } = req.query;
  
  const queryEmbedding = await generateEmbedding(q as string);
  
  const result = await query(
    `SELECT id, content, 1 - (embedding <=> $1::vector) as similarity
     FROM documents
     WHERE embedding IS NOT NULL
     ORDER BY embedding <=> $1::vector
     LIMIT $2`,
    [JSON.stringify(queryEmbedding), parseInt(limit as string)]
  );
  
  res.json({ query: q, results: result.rows });
}
```

**Similarity threshold:**
```typescript
const result = await query(
  `SELECT id, content, 1 - (embedding <=> $1::vector) as similarity
   FROM documents
   WHERE embedding IS NOT NULL AND (embedding <=> $1::vector) < $2
   ORDER BY embedding <=> $1::vector LIMIT $3`,
  [JSON.stringify(queryEmbedding), 0.3, limit]
);
```

**Hybrid search (vector + keyword):**
```typescript
const result = await query(
  `SELECT id, content,
    1 - (embedding <=> $1::vector) as similarity,
    ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) as keyword_rank
   FROM documents
   WHERE embedding IS NOT NULL AND content ILIKE $3
   ORDER BY (1 - (embedding <=> $1::vector)) * 0.7 + 
            ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) * 0.3 DESC
   LIMIT $4`,
  [JSON.stringify(queryEmbedding), keywordQuery, `%${keywordQuery}%`, limit]
);
```

**App Router:**
```typescript
// app/api/search/route.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const q = searchParams.get('q');
  const limit = parseInt(searchParams.get('limit') || '10');
  
  const queryEmbedding = await generateEmbedding(q!);
  
  const result = await query(
    `SELECT id, content, 1 - (embedding <=> $1::vector) as similarity
     FROM documents
     ORDER BY embedding <=> $1::vector LIMIT $2`,
    [JSON.stringify(queryEmbedding), limit]
  );
  
  return NextResponse.json({ query: q, results: result.rows });
}
```

**Similarity metrics:**
- Cosine similarity - 0-1, higher is more similar
- Cosine distance - 0-2, lower is more similar
- L2 distance - Euclidean distance
- Inner product - Dot product

## Benefits

- **Semantic search** - Find content by meaning. Vector similarity search understands context and relationships, not just exact keyword matches.
- **Performance** - Database-level operations. PostgreSQL handles vector calculations efficiently using specialized indexes and optimized operators.
- **Flexibility** - Combine with keyword search. You can blend semantic search with traditional full-text search for the best of both worlds.
- **Scalability** - Works with large datasets. Vector indexes make similarity search fast even with millions of documents, enabling real-time semantic search at scale.

Next: [building-rag-api.md](./building-rag-api.md)
