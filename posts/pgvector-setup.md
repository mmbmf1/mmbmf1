<!--
#pgvector #postgresql #vectors #embeddings #database #localdevelopment
-->

# PostgreSQL Vector Search with pgvector

![Vector Similarity Search](./images/vector-search.png)

## Introduction

pgvector extension for PostgreSQL. Store and query vector embeddings. Semantic search, similarity matching, AI features in the database.

## The Problem

Storing embeddings as JSON arrays doesn't support efficient similarity search.

```sql
-- Basic approach - no vector capabilities
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding JSONB  -- Can't query efficiently
);
```

## The Solution

Use pgvector's `vector` type and similarity operators.

**Docker setup:**
```yaml
# docker-compose.yml
services:
  db:
    image: ankane/pgvector:latest
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

**Enable extension:**
```sql
-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**Create table with vector:**
```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- OpenAI ada-002 dimension
);

-- HNSW index for fast similarity search
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops);
```

**Insert vector data:**
```sql
-- Insert document with embedding
INSERT INTO documents (content, embedding) VALUES
    (
        'PostgreSQL is a powerful database',
        '[0.1, 0.2, 0.3, ...]'::vector
    );
```

**Similarity search:**
```sql
-- Find similar documents using cosine similarity
SELECT 
    id,
    content,
    1 - (embedding <=> '[0.1, 0.2, 0.3, ...]'::vector) as similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, 0.3, ...]'::vector
LIMIT 5;
```

**pgvector operators:**
- `<=>` - Cosine distance (1 - cosine similarity)
- `<->` - L2 distance (Euclidean)
- `<#)` - Negative inner product

**Index types:**
```sql
-- HNSW index (recommended for large datasets)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- IVFFlat index (faster to build, slower queries)
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops) 
WITH (lists = 100);
```

## Benefits

- Semantic search - Find similar content by meaning
- AI features - Store and query embeddings efficiently
- Simplicity - No separate vector database
- Performance - Vector indexes enable fast queries

Next: [vector-similarity-search.md](./vector-similarity-search.md)
