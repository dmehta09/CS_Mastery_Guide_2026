# Project 10: AI-Powered Search & RAG System

## Difficulty Level: Advanced
**Estimated Time**: 4-5 weeks per language

## What You're Building

An intelligent search and question-answering system using vector databases and RAG (Retrieval Augmented Generation) - like Perplexity AI or ChatGPT with custom knowledge.

## Key Concepts (2026 Relevant!)

- Vector embeddings
- Semantic search
- RAG (Retrieval Augmented Generation)
- LLM integration (OpenAI, Anthropic)
- Vector databases (Pinecone, Weaviate)
- Chunking strategies
- Prompt engineering

## Architecture

```
User Query → Embedding → Vector Search → Retrieved Docs → LLM → Response
                ↓                                ↓
          Vector DB                       Context Injection
```

## How RAG Works

### Traditional Search

```
User: "What is photosynthesis?"
System: Returns documents with "photosynthesis"
User: Must read and understand documents
```

### RAG Search

```
User: "What is photosynthesis?"
System:
1. Convert query to vector embedding
2. Find semantically similar documents in vector DB
3. Retrieve top 5 relevant documents
4. Feed documents as context to LLM
5. LLM generates natural answer using context

Response: "Photosynthesis is the process by which..."
[Generated answer with citations]
```

## Implementation Flow

### 1. Document Ingestion

```typescript
// Process and store documents
async function ingestDocument(doc: Document) {
  // 1. Chunk document
  const chunks = chunkDocument(doc.content, {
    size: 1000, // characters
    overlap: 200 // characters
  });

  // 2. Generate embeddings
  for (const chunk of chunks) {
    const embedding = await openai.embeddings.create({
      model: "text-embedding-3-large",
      input: chunk.text
    });

    // 3. Store in vector DB
    await vectorDB.upsert({
      id: chunk.id,
      values: embedding.data[0].embedding,
      metadata: {
        document_id: doc.id,
        text: chunk.text,
        page: chunk.page
      }
    });
  }
}
```

### 2. Query Processing

```typescript
async function search(query: string, k: number = 5) {
  // 1. Convert query to embedding
  const queryEmbedding = await openai.embeddings.create({
    model: "text-embedding-3-large",
    input: query
  });

  // 2. Vector similarity search
  const results = await vectorDB.query({
    vector: queryEmbedding.data[0].embedding,
    topK: k,
    includeMetadata: true
  });

  // 3. Return relevant chunks
  return results.matches.map((match) => ({
    text: match.metadata.text,
    score: match.score,
    document_id: match.metadata.document_id
  }));
}
```

### 3. RAG Generation

```typescript
async function answerQuestion(question: string) {
  // 1. Retrieve relevant context
  const context = await search(question, 5);

  // 2. Build prompt
  const prompt = `
Context information:
${context.map((c) => c.text).join("\n\n")}

Question: ${question}

Answer the question based on the context above.
Include citations using [1], [2], etc.
`;

  // 3. Generate answer
  const response = await openai.chat.completions.create({
    model: "gpt-4-turbo",
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: prompt }
    ],
    temperature: 0.3
  });

  return {
    answer: response.choices[0].message.content,
    sources: context
  };
}
```

## Vector Database Options

### Pinecone

```typescript
import { Pinecone } from "@pinecone-database/pinecone";

const pinecone = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY
});

const index = pinecone.index("knowledge-base");

// Upsert vectors
await index.upsert([
  {
    id: "vec1",
    values: [0.1, 0.2, 0.3, ...], // 1536 dimensions
    metadata: { text: "..." }
  }
]);

// Query
const results = await index.query({
  vector: [0.1, 0.2, ...],
  topK: 5
});
```

### Weaviate

```typescript
import weaviate from "weaviate-ts-client";

const client = weaviate.client({
  scheme: "http",
  host: "localhost:8080"
});

// Create schema
await client.schema
  .classCreator()
  .withClass({
    class: "Document",
    vectorizer: "text2vec-openai",
    properties: [
      { name: "content", dataType: ["text"] },
      { name: "title", dataType: ["string"] }
    ]
  })
  .do();

// Search
const result = await client.graphql
  .get()
  .withClassName("Document")
  .withNearText({ concepts: ["photosynthesis"] })
  .withLimit(5)
  .withFields("content title _additional { distance }")
  .do();
```

## Advanced Features

### Hybrid Search (Vector + Keywords)

```typescript
async function hybridSearch(query: string) {
  // 1. Vector search
  const vectorResults = await vectorDB.query({
    vector: await getEmbedding(query),
    topK: 10
  });

  // 2. Keyword search (Elasticsearch)
  const keywordResults = await elasticsearch.search({
    index: "documents",
    body: {
      query: {
        match: { content: query }
      },
      size: 10
    }
  });

  // 3. Combine with RRF (Reciprocal Rank Fusion)
  return combineResults(vectorResults, keywordResults);
}
```

### Chunking Strategies

```typescript
// Fixed-size chunks with overlap
function chunkFixedSize(text: string, size = 1000, overlap = 200) {
  const chunks = [];
  for (let i = 0; i < text.length; i += size - overlap) {
    chunks.push(text.slice(i, i + size));
  }
  return chunks;
}

// Semantic chunking (by paragraphs/sections)
function chunkSemantic(text: string) {
  return text.split("\n\n").filter((p) => p.length > 100);
}

// Sentence-based chunking
function chunkSentences(text: string, sentencesPerChunk = 5) {
  const sentences = text.match(/[^.!?]+[.!?]+/g) || [];
  const chunks = [];
  for (let i = 0; i < sentences.length; i += sentencesPerChunk) {
    chunks.push(
      sentences.slice(i, i + sentencesPerChunk).join(" ")
    );
  }
  return chunks;
}
```

## API Design

```http
# Ingest Documents
POST /api/v1/documents
{
  "title": "Biology Textbook Chapter 1",
  "content": "...",
  "metadata": { "author": "...", "date": "..." }
}

# Search
POST /api/v1/search
{
  "query": "How does photosynthesis work?",
  "top_k": 5
}

Response:
{
  "results": [
    {
      "text": "Photosynthesis is...",
      "score": 0.92,
      "document_id": "doc_123",
      "metadata": { ... }
    }
  ]
}

# Ask Question (RAG)
POST /api/v1/ask
{
  "question": "What are the steps of photosynthesis?"
}

Response:
{
  "answer": "Photosynthesis occurs in two main stages...",
  "sources": [
    { "document_id": "doc_123", "excerpt": "..." },
    { "document_id": "doc_456", "excerpt": "..." }
  ],
  "confidence": 0.89
}
```

## Technology Stack

### NestJS
- @pinecone-database/pinecone
- openai
- langchain

### Golang
- pinecone-go-client
- go-openai
- vector libraries

### Python
- pinecone-client
- openai
- langchain
- sentence-transformers

## Success Criteria

✅ Sub-second search response
✅ Accurate semantic matching
✅ Contextual answers from LLM
✅ Source citations
✅ Handle 10K+ documents
✅ Embedding generation pipeline

---

**Next**: Project 11 (Distributed Caching)
