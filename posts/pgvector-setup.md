<!--
#pgvector #postgresql #vectors #embeddings #database #localdevelopment
-->

# PostgreSQL Vector Search with pgvector

<!-- ![Vector Similarity Search](images/vector-search.png) -->

## Introduction

pgvector extension for PostgreSQL. Store and query vector embeddings. Semantic search, similarity matching, AI features in the database. Enables powerful AI-driven features like semantic search and recommendation systems directly in PostgreSQL. Perfect foundation for building RAG applications and other AI-powered features.

## The Problem

Storing embeddings as JSON arrays doesn't efficiently support similarity search operations. JSON arrays can't use specialized vector indexes, making similarity searches slow. You'd have to fetch all embeddings and calculate similarities in your application code, which doesn't scale. Vector operations need specialized data types and indexes to be efficient.

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding JSONB
);
```

## The Solution

Use pgvector's `vector` type and similarity operators. Store embeddings in pgvector's native `vector` type, which supports efficient similarity calculations. Create HNSW or IVFFlat indexes that make similarity searches fast even with millions of vectors. Use built-in operators like `<=>` for cosine distance calculations.

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
CREATE EXTENSION IF NOT EXISTS vector;
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**Create table with vector:**
```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);
```

**Insert vector data:**
```sql
INSERT INTO documents (content, embedding) VALUES
    ('PostgreSQL is a powerful database', '[0.1, 0.2, 0.3, ...]'::vector);
```

**Similarity search:**
```sql
SELECT id, content, 1 - (embedding <=> '[0.1, 0.2, 0.3, ...]'::vector) as similarity
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
-- HNSW index (recommended)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- IVFFlat index (faster to build)
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

## Benefits

- **Semantic search** - Find similar content by meaning. Vector embeddings capture semantic relationships, enabling search that understands context, not just keywords.
- **AI features** - Store and query embeddings efficiently. Perfect for building recommendation systems, similarity matching, and other AI-powered features directly in PostgreSQL.
- **Simplicity** - No separate vector database needed. Keep everything in PostgreSQL, simplifying your architecture and reducing operational complexity.
- **Performance** - Vector indexes enable fast queries. HNSW indexes make similarity searches fast even with millions of vectors, enabling real-time semantic search.

Next: [vector-similarity-search.md](./vector-similarity-search.md)
