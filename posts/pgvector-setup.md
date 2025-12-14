<!--
#pgvector #postgresql #vectors #embeddings #database #localdevelopment
-->

# PostgreSQL Vector Search with pgvector

## Introduction

Set up pgvector extension in PostgreSQL for storing and querying vector embeddings. This approach enables semantic search, similarity matching, and AI-powered features directly in the database.

## The Problem

When building applications with AI features, you need to store and search vector embeddings. The typical approaches involve using separate vector databases or storing embeddings as JSON arrays, which adds complexity and doesn't leverage PostgreSQL's capabilities.

```sql
-- Basic approach - no vector capabilities
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding JSONB  -- Stored as array, can't query efficiently
);
```

This works, but doesn't support efficient similarity search or leverage database indexes for vector operations.

## The Solution

Instead of storing embeddings as JSON, we use pgvector's vector type and similarity search functions. The architecture flows from pgvector extension through vector columns to efficient similarity queries.

### Architecture Overview

pgvector Extension → Vector Columns → Vector Indexes → Similarity Queries

- **pgvector extension**: Enables vector capabilities
- **Vector columns**: Store embeddings as vector type
- **Vector indexes**: HNSW or IVFFlat indexes for fast search
- **Similarity queries**: Cosine, L2, or inner product similarity

### Implementation

**Docker setup with pgvector:**
```yaml
# docker-compose.yml
services:
  db:
    image: ankane/pgvector:latest  # pgvector-enabled image
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

**Alternative**: Install pgvector on standard PostgreSQL image using a custom Dockerfile or use `pgvector/pgvector:pg15` if available.

**Enable pgvector extension:**
```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify installation
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**Create table with vector column:**
```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- OpenAI ada-002 dimension
);

-- Create HNSW index for fast similarity search
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

**Basic similarity search:**
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

### pgvector Operators

- **<=>** - Cosine distance (1 - cosine similarity)
- **<->** - L2 distance (Euclidean)
- **<#)** - Negative inner product
- **Similarity ranges**: 0 (identical) to 2 (opposite) for cosine distance

## Benefits

This approach provides efficient vector storage and similarity search directly in PostgreSQL. We get fast similarity queries, standard SQL interface, and no need for separate vector databases. This pattern works well for:

- **Semantic search** - Find similar content by meaning
- **AI features** - Store and query embeddings efficiently
- **Simplicity** - No separate vector database needed
- **Performance** - Vector indexes enable fast queries

The clean separation between vector storage and queries means semantic search operations are efficient and maintainable.

This builds on Docker setup (see [docker-postgresql-setup.md](./docker-postgresql-setup.md)). Next, see how to implement vector similarity search patterns (see [vector-similarity-search.md](./vector-similarity-search.md)).
