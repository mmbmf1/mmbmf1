<!--
#rag #pgvector #nextjs #api #llm #embeddings #semanticsearch #postgresql
-->

# Building a RAG API with PostgreSQL and Next.js

## Introduction

Built a Retrieval Augmented Generation (RAG) API using pgvector, PostgreSQL, and Next.js that combines semantic search with LLM generation. This approach stores document embeddings in PostgreSQL and retrieves relevant context for LLM prompts.

## The Problem

When building AI applications, you need to provide relevant context to LLMs without exceeding token limits. The typical approaches involve sending entire documents or using separate vector databases, which is inefficient and adds complexity.

```typescript
// Naive approach - sends all documents
const allDocs = await query('SELECT content FROM documents');
const prompt = `Context: ${allDocs.map(d => d.content).join('\n')}\n\nQuestion: ${question}`;
const answer = await callLLM(prompt);
```

This works for small datasets, but quickly hits token limits and doesn't scale to large document collections.

## The Solution

Instead of sending all documents, we use vector similarity search to retrieve only relevant context, then combine it with the user's question for the LLM. The architecture flows from user queries through semantic search to context retrieval and LLM generation.

### Architecture Overview

User Query → Generate Embedding → Vector Search → Retrieve Context → LLM Prompt → Response

- **User query**: Question or prompt from user
- **Generate embedding**: Convert query to vector
- **Vector search**: Find similar documents using pgvector
- **Retrieve context**: Get top-k relevant documents
- **LLM prompt**: Combine context with user question
- **Response**: Generated answer with sources

### Implementation

**Document ingestion:**
```typescript
// pages/api/documents/ingest.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { content, metadata } = req.body;
  
  if (!content) {
    return res.status(400).json({ error: 'Content is required' });
  }
  
  try {
    // Generate embedding for document
    const embedding = await generateEmbedding(content);
    
    // Store document with embedding
    const result = await query(
      `INSERT INTO documents (content, embedding, metadata)
       VALUES ($1, $2::vector, $3::jsonb)
       RETURNING id`,
      [content, JSON.stringify(embedding), JSON.stringify(metadata || {})]
    );
    
    res.json({ id: result.rows[0].id, status: 'ingested' });
  } catch (error) {
    console.error('Ingestion error:', error);
    res.status(500).json({ error: 'Failed to ingest document' });
  }
}
```

**RAG search endpoint:**
```typescript
// pages/api/rag/search.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';
import { callLLM } from '@/lib/llm';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { question, topK = 5 } = req.body;
  
  if (!question) {
    return res.status(400).json({ error: 'Question is required' });
  }
  
  try {
  
  // Generate embedding for question
  const queryEmbedding = await generateEmbedding(question);
  
  // Find similar documents
  const searchResult = await query(
    `SELECT 
      id,
      content,
      metadata,
      1 - (embedding <=> $1::vector) as similarity
    FROM documents
    WHERE embedding IS NOT NULL
      AND (embedding <=> $1::vector) < 0.5  -- Similarity threshold
    ORDER BY embedding <=> $1::vector
    LIMIT $2`,
    [JSON.stringify(queryEmbedding), topK]
  );
  
  // Build context from retrieved documents
  const context = searchResult.rows
    .map(row => `[${row.metadata?.source || 'Document'}]: ${row.content}`)
    .join('\n\n');
  
  // Truncate context if too long (LLM token limits)
  const maxContextLength = 3000; // Adjust based on your model's context window
  const truncatedContext = context.length > maxContextLength 
    ? context.substring(0, maxContextLength) + '...'
    : context;
  
  // Create RAG prompt
  const prompt = `You are a helpful assistant. Use the following context to answer the question. If the context doesn't contain enough information, say so.

Context:
${truncatedContext}

Question: ${question}

Answer:`;
  
  // Generate answer using LLM
  const answer = await callLLM(prompt);
  
    res.json({
      question,
      answer,
      sources: searchResult.rows.map(row => ({
        id: row.id,
        similarity: parseFloat(row.similarity),
        metadata: row.metadata
      }))
    });
  } catch (error) {
    console.error('RAG search error:', error);
    res.status(500).json({ error: 'Failed to process question' });
  }
}
```

**Embedding utility:**
```typescript
// lib/embeddings.ts
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function generateEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-ada-002',
    input: text,
  });
  
  return response.data[0].embedding;
}
```

**LLM utility:**
```typescript
// lib/llm.ts
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function callLLM(prompt: string): Promise<string> {
  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.7,
  });
  
  return response.choices[0].message.content || '';
}
```

**App Router example:**
```typescript
// app/api/rag/search/route.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';
import { callLLM } from '@/lib/llm';
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const body = await request.json();
  const { question, topK = 5 } = body;
  
  const queryEmbedding = await generateEmbedding(question);
  
  const searchResult = await query(
    `SELECT 
      id,
      content,
      metadata,
      1 - (embedding <=> $1::vector) as similarity
    FROM documents
    ORDER BY embedding <=> $1::vector
    LIMIT $2`,
    [JSON.stringify(queryEmbedding), topK]
  );
  
  const context = searchResult.rows
    .map(row => row.content)
    .join('\n\n');
  
  const prompt = `Context:\n${context}\n\nQuestion: ${question}\n\nAnswer:`;
  const answer = await callLLM(prompt);
  
  return NextResponse.json({
    question,
    answer,
    sources: searchResult.rows.map(row => ({
      id: row.id,
      similarity: parseFloat(row.similarity)
    }))
  });
}
```

### RAG Workflow

1. **Ingest documents** - Store documents with embeddings
2. **User query** - Receive question from user
3. **Semantic search** - Find relevant documents using vector similarity
4. **Context retrieval** - Extract top-k most relevant documents
5. **Prompt construction** - Combine context with user question
6. **LLM generation** - Generate answer using LLM
7. **Response** - Return answer with source citations

## Benefits

This approach provides efficient RAG capabilities using PostgreSQL for storage and retrieval. We get semantic search, context-aware responses, and source citations without separate vector databases. This pattern works well for:

- **Question answering** - Answer questions using document context
- **Document Q&A** - Query large document collections
- **Knowledge bases** - Build searchable knowledge systems
- **AI assistants** - Provide context-aware responses

The clean separation between document storage, semantic search, and LLM generation means RAG systems are efficient and maintainable.

This builds on vector similarity search (see [vector-similarity-search.md](./vector-similarity-search.md)) and API routes (see [nextjs-api-routes.md](./nextjs-api-routes.md)). Together with pgvector setup (see [pgvector-setup.md](./pgvector-setup.md)), this provides a complete RAG implementation.
