<!--
#rag #pgvector #nextjs #api #llm #embeddings #semanticsearch #postgresql
-->

# Building a RAG API with PostgreSQL and Next.js

<!-- ![RAG Architecture](images/rag-architecture.png) -->

## Introduction

RAG (Retrieval Augmented Generation) API using pgvector and Next.js. Semantic search + LLM generation. Store embeddings, retrieve context, generate answers. Combines the power of semantic search with large language models to create intelligent applications that can answer questions based on your own data. Perfect for building AI assistants, documentation systems, and knowledge bases.

## The Problem

Sending all documents to LLMs can hit token limits and doesn't scale well for large document collections. Large language models have token limits, and sending entire document collections quickly exhausts those limits. Even if you could send everything, you're paying for processing irrelevant context. As your document collection grows, this approach becomes completely impractical.

```typescript
const allDocs = await query('SELECT content FROM documents');
const prompt = `Context: ${allDocs.map(d => d.content).join('\n')}\n\nQuestion: ${question}`;
const answer = await callLLM(prompt);
```

## The Solution

Use vector similarity search to retrieve only relevant context, then combine with user question. Generate an embedding for the user's question, then use vector similarity search to find the most relevant documents. Retrieve only the top-k most similar documents as context, keeping your prompt within token limits while ensuring the LLM has relevant information to answer the question.

**Document ingestion:**
```typescript
// pages/api/documents/ingest.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';

export default async function handler(req, res) {
  const { content, metadata } = req.body;
  const embedding = await generateEmbedding(content);
  
  const result = await query(
    `INSERT INTO documents (content, embedding, metadata)
     VALUES ($1, $2::vector, $3::jsonb) RETURNING id`,
    [content, JSON.stringify(embedding), JSON.stringify(metadata || {})]
  );
  
  res.json({ id: result.rows[0].id, status: 'ingested' });
}
```

**RAG search endpoint:**
```typescript
// pages/api/rag/search.ts
import { query } from '@/lib/db';
import { generateEmbedding } from '@/lib/embeddings';
import { callLLM } from '@/lib/llm';

export default async function handler(req, res) {
  const { question, topK = 5 } = req.body;
  const queryEmbedding = await generateEmbedding(question);
  
  const searchResult = await query(
    `SELECT id, content, metadata, 1 - (embedding <=> $1::vector) as similarity
     FROM documents
     WHERE embedding IS NOT NULL AND (embedding <=> $1::vector) < 0.5
     ORDER BY embedding <=> $1::vector LIMIT $2`,
    [JSON.stringify(queryEmbedding), topK]
  );
  
  const context = searchResult.rows
    .map(row => `[${row.metadata?.source || 'Document'}]: ${row.content}`)
    .join('\n\n')
    .substring(0, 3000);
  
  const prompt = `Use the following context to answer the question. If the context doesn't contain enough information, say so.

Context:
${context}

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
  const { question, topK = 5 } = await request.json();
  const queryEmbedding = await generateEmbedding(question);
  
  const searchResult = await query(
    `SELECT id, content, metadata, 1 - (embedding <=> $1::vector) as similarity
     FROM documents ORDER BY embedding <=> $1::vector LIMIT $2`,
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

- **Question answering** - Answer using document context. LLMs can provide accurate, contextual answers based on your own data instead of generic responses.
- **Document Q&A** - Query large collections. Users can ask questions about thousands or millions of documents, and the system finds relevant context automatically.
- **Knowledge bases** - Searchable systems. Build intelligent knowledge bases that understand questions and retrieve relevant information automatically.
- **AI assistants** - Context-aware responses. Create AI assistants that answer questions based on your documentation, codebase, or any text corpus you provide.

Next: Complete RAG implementation ready to use.
