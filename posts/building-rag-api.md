<!--
#rag #pgvector #nextjs #api #llm #embeddings #semanticsearch #postgresql
-->

# Building a RAG API with PostgreSQL and Next.js

![RAG Architecture](./images/rag-architecture.png)

## Introduction

RAG (Retrieval Augmented Generation) API using pgvector and Next.js. Semantic search + LLM generation. Store embeddings, retrieve context, generate answers.

## The Problem

Sending all documents to LLMs hits token limits and doesn't scale.

```typescript
// Naive approach - sends all documents
const allDocs = await query('SELECT content FROM documents');
const prompt = `Context: ${allDocs.map(d => d.content).join('\n')}\n\nQuestion: ${question}`;
const answer = await callLLM(prompt);
```

## The Solution

Use vector similarity search to retrieve only relevant context, then combine with user question.

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
    const embedding = await generateEmbedding(content);
    
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
    const queryEmbedding = await generateEmbedding(question);
    
    const searchResult = await query(
      `SELECT 
        id,
        content,
        metadata,
        1 - (embedding <=> $1::vector) as similarity
      FROM documents
      WHERE embedding IS NOT NULL
        AND (embedding <=> $1::vector) < 0.5
      ORDER BY embedding <=> $1::vector
      LIMIT $2`,
      [JSON.stringify(queryEmbedding), topK]
    );
    
    const context = searchResult.rows
      .map(row => `[${row.metadata?.source || 'Document'}]: ${row.content}`)
      .join('\n\n');
    
    const maxContextLength = 3000;
    const truncatedContext = context.length > maxContextLength 
      ? context.substring(0, maxContextLength) + '...'
      : context;
    
    const prompt = `You are a helpful assistant. Use the following context to answer the question. If the context doesn't contain enough information, say so.

Context:
${truncatedContext}

Question: ${question}

Answer:`;
    
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

**App Router:**
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
  
  const context = searchResult.rows.map(row => row.content).join('\n\n');
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

**RAG workflow:**
1. Ingest documents with embeddings
2. User query → Generate embedding
3. Vector search → Find similar documents
4. Retrieve top-k context
5. Build prompt with context + question
6. Generate answer with LLM
7. Return answer + sources

## Benefits

- Question answering - Answer using document context
- Document Q&A - Query large collections
- Knowledge bases - Searchable systems
- AI assistants - Context-aware responses

Next: Complete RAG implementation ready to use.
