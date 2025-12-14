<!--
#pgvector #postgresql #vectors #similarity #semanticsearch #database #nextjs
-->

# Vector Similarity Search Patterns

## Introduction

Built vector similarity search queries using pgvector to find semantically similar content. This approach uses cosine similarity and vector indexes to enable fast semantic search directly in PostgreSQL.

## The Problem

When building semantic search features, you need to find documents similar to a query based on meaning rather than keywords. The typical approaches involve using separate vector databases or calculating similarities in application code, which adds complexity and doesn't leverage PostgreSQL's capabilities.

```typescript
// Application-level approach - inefficient
const queryEmbedding = await generateEmbedding(query);
const allDocs = await query('SELECT id, content, embedding FROM documents');
const similarities = allDocs.map(doc => ({
  ...doc,
  similarity: cosineSimilarity(queryEmbedding, doc.embedding)
}));
const results = similarities.sort((a, b) => b.similarity - a.similarity).slice(0, 10);
```

This works, but requires fetching all documents and calculating similarities in JavaScript, which is slow and doesn't scale.

## The Solution

Instead of application-level similarity calculation, we use pgvector's similarity operators and indexes to perform efficient vector searches at the database level. The architecture flows from query embeddings through vector similarity operators to ranked results.

### Architecture Overview

Query Embedding → Vector Similarity Operator → Vector Index → Ranked Results

- **Query embedding**: Vector representation of search query
- **Similarity operator**: pgvector <=> operator for cosine distance
- **Vector index**: HNSW index for fast approximate search
- **Ranked results**: Documents ordered by similarity

### Implementation

**Basic similarity search:**
```typescript
// pages/api/search.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';

export default async function handler(req, res) {
  const { q, limit = 10 } = req.query;
  
  if (!q) {
    return res.status(400).json({ error: 'Query parameter q is required' });
  }
  
  try {
    // Generate embedding for query
    const queryEmbedding = await generateEmbedding(q as string);
    
    const result = await query(
      `SELECT 
        id,
        content,
        1 - (embedding <=> $1::vector) as similarity
      FROM documents
      WHERE embedding IS NOT NULL
      ORDER BY embedding <=> $1::vector
      LIMIT $2`,
      [JSON.stringify(queryEmbedding), parseInt(limit as string)]
    );
    
    res.json({
      query: q,
      results: result.rows.map(row => ({
        id: row.id,
        content: row.content,
        similarity: parseFloat(row.similarity)
      }))
    });
  } catch (error) {
    console.error('Search error:', error);
    res.status(500).json({ error: 'Failed to perform search' });
  }
}
```

**Similarity search with threshold:**
```typescript
// Only return results above similarity threshold
const result = await query(
  `SELECT 
    id,
    content,
    1 - (embedding <=> $1::vector) as similarity
  FROM documents
  WHERE embedding IS NOT NULL
    AND (embedding <=> $1::vector) < $2  -- Cosine distance threshold
  ORDER BY embedding <=> $1::vector
  LIMIT $3`,
  [JSON.stringify(queryEmbedding), 0.3, limit] // 0.3 distance = ~0.7 similarity
);
```

**Hybrid search (vector + keyword):**
```typescript
// Combine semantic search with keyword filtering
const result = await query(
  `SELECT 
    id,
    content,
    1 - (embedding <=> $1::vector) as similarity,
    ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) as keyword_rank
  FROM documents
  WHERE embedding IS NOT NULL
    AND content ILIKE $3
  ORDER BY 
    (1 - (embedding <=> $1::vector)) * 0.7 + 
    ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) * 0.3 DESC
  LIMIT $4`,
  [JSON.stringify(queryEmbedding), keywordQuery, `%${keywordQuery}%`, limit]
);
```

**App Router example:**
```typescript
// app/api/search/route.ts
import { query } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const q = searchParams.get('q');
  const limit = parseInt(searchParams.get('limit') || '10');
  
  if (!q) {
    return NextResponse.json(
      { error: 'Query parameter q is required' },
      { status: 400 }
    );
  }
  
  const queryEmbedding = await generateEmbedding(q);
  
  const result = await query(
    `SELECT 
      id,
      content,
      1 - (embedding <=> $1::vector) as similarity
    FROM documents
    ORDER BY embedding <=> $1::vector
    LIMIT $2`,
    [JSON.stringify(queryEmbedding), limit]
  );
  
  return NextResponse.json({
    query: q,
    results: result.rows
  });
}
```

### Similarity Metrics

- **Cosine similarity** - Measures angle between vectors (0-1, higher is more similar)
- **Cosine distance** - 1 - cosine similarity (0-2, lower is more similar)
- **L2 distance** - Euclidean distance between vectors
- **Inner product** - Dot product of vectors

### Performance Tips

- **Use HNSW indexes** - Fast approximate search for large datasets
- **Set index parameters** - Tune m and ef_construction for your use case
- **Limit results** - Always use LIMIT to avoid large result sets
- **Filter before search** - Apply WHERE clauses before similarity calculation when possible

## Benefits

This approach provides efficient semantic search directly in PostgreSQL. We get fast similarity queries, no separate vector database needed, and standard SQL interface. This pattern works well for:

- **Semantic search** - Find content by meaning, not just keywords
- **Recommendation systems** - Find similar items
- **Content discovery** - Surface related content
- **AI features** - Enable LLM-powered search

The clean separation between vector storage and similarity search means semantic search operations are efficient and maintainable.

This builds on pgvector setup (see [pgvector-setup.md](./pgvector-setup.md)) and API routes (see [nextjs-api-routes.md](./nextjs-api-routes.md)). Next, see how to build a complete RAG API (see [building-rag-api.md](./building-rag-api.md)).
