# Project 15: Vector Database & RAG System

## Difficulty: Advanced (2026 Relevant!)
**Estimated Time**: 4-5 weeks

## Problem Statement

Design a Vector Database and Retrieval Augmented Generation (RAG) system like Pinecone, Weaviate, or Chroma that stores embeddings, performs similarity search, and integrates with LLMs for context-aware responses.

**2026 Use Case**: Power AI chatbots, semantic search, recommendation systems with LLM integration

## Functional Requirements

1. **Embedding Storage**: Store vector embeddings (768-1536 dimensions)
2. **Similarity Search**: Find nearest neighbors (ANN - Approximate Nearest Neighbor)
3. **Metadata Filtering**: Filter by tags, categories before search
4. **RAG Pipeline**: Retrieve relevant docs + Generate LLM response
5. **Scalability**: Billion-scale vectors
6. **Low Latency**: < 100ms search

## Architecture

```
┌─────────────────────────────────────────────┐
│           Document Pipeline                 │
│                                             │
│  Documents → Chunking → Embedding → Vector DB│
│               ↓           ↓                  │
│           (Split)    (OpenAI API)           │
└─────────────────────────────────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │   Vector DB    │
              │  ┌──────────┐  │
              │  │  Index   │  │
              │  │  (HNSW)  │  │
              │  └──────────┘  │
              └────────┬───────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
┌───────────────┐           ┌──────────────────┐
│Query Pipeline │           │  RAG Pipeline    │
│               │           │                  │
│Query → Embed  │           │1. Query → Embed  │
│  ↓            │           │2. Search vectors │
│Search vectors │           │3. Get docs       │
│  ↓            │           │4. LLM with context│
│Return docs    │           │5. Generate answer│
└───────────────┘           └──────────────────┘
```

## Vector Embeddings

### What are Embeddings?

```
Text: "The cat sat on the mat"

Embedding: [0.234, -0.567, 0.123, ..., 0.890]
           768 dimensions (for BERT)
           1536 dimensions (for OpenAI text-embedding-ada-002)

Similar texts have similar embeddings:
"The cat sat on the mat"     → [0.234, -0.567, ...]
"A cat is sitting on a rug"  → [0.240, -0.570, ...] (close!)
"I love pizza"               → [-0.890, 0.123, ...] (far!)
```

### Generating Embeddings

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function getEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-ada-002',
    input: text
  });

  return response.data[0].embedding;  // 1536-dimensional vector
}

// Usage:
const embedding = await getEmbedding("How do I reset my password?");
// [0.0234, -0.0567, 0.0123, ..., 0.0890]
```

## Similarity Metrics

### 1. Cosine Similarity

```typescript
function cosineSimilarity(a: number[], b: number[]): number {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }

  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}

// Example:
const v1 = [1, 0, 0];
const v2 = [0.7, 0.7, 0];

cosineSimilarity(v1, v2);  // 0.7 (similar)

const v3 = [0, 0, 1];
cosineSimilarity(v1, v3);  // 0.0 (perpendicular, not similar)
```

### 2. Euclidean Distance

```typescript
function euclideanDistance(a: number[], b: number[]): number {
  let sum = 0;
  for (let i = 0; i < a.length; i++) {
    sum += Math.pow(a[i] - b[i], 2);
  }
  return Math.sqrt(sum);
}

// Smaller distance = more similar
```

### 3. Dot Product

```typescript
function dotProduct(a: number[], b: number[]): number {
  let sum = 0;
  for (let i = 0; i < a.length; i++) {
    sum += a[i] * b[i];
  }
  return sum;
}

// Higher dot product = more similar (for normalized vectors)
```

## Indexing Algorithms

### 1. Brute Force (Exact Search)

```typescript
class BruteForceIndex {
  private vectors: Array<{ id: string; vector: number[] }> = [];

  add(id: string, vector: number[]) {
    this.vectors.push({ id, vector });
  }

  search(query: number[], k: number): Array<{ id: string; score: number }> {
    const results = this.vectors.map(item => ({
      id: item.id,
      score: cosineSimilarity(query, item.vector)
    }));

    // Sort by score descending
    results.sort((a, b) => b.score - a.score);

    return results.slice(0, k);
  }
}

// Time complexity: O(n * d)
// n = number of vectors, d = dimensions
// For 1M vectors × 1536 dims = too slow!
```

### 2. HNSW (Hierarchical Navigable Small World)

**Concept**: Build graph where nodes are vectors, edges connect similar vectors

```
Layer 2:  A ──── B
          │
Layer 1:  A ──── B ──── C ──── D
          │      │      │      │
Layer 0:  A ─ X ─ B ─ Y ─ C ─ Z ─ D ─ ...

Search:
1. Start at top layer (sparse)
2. Greedy traverse to nearest neighbor
3. Drop to next layer
4. Repeat until bottom layer
5. Search locally for top-k

Time: O(log n) instead of O(n)
```

**Implementation** (simplified):

```typescript
class HNSWIndex {
  private layers: Map<number, Map<string, Set<string>>> = new Map();
  private vectors: Map<string, number[]> = new Map();

  add(id: string, vector: number[]) {
    this.vectors.set(id, vector);

    // Insert into multiple layers
    let layer = 0;
    let insertMore = true;

    while (insertMore) {
      if (!this.layers.has(layer)) {
        this.layers.set(layer, new Map());
      }

      const layerGraph = this.layers.get(layer)!;

      // Find M nearest neighbors in this layer
      const neighbors = this.findNearest(vector, layer, 16);

      // Add edges
      layerGraph.set(id, new Set(neighbors));

      // Probabilistically insert into next layer
      insertMore = Math.random() < 0.5;
      layer++;
    }
  }

  search(query: number[], k: number): string[] {
    const visited = new Set<string>();
    let candidates = ['entry_point'];  // Start from top layer

    // Traverse from top to bottom
    for (let layer = this.layers.size - 1; layer >= 0; layer--) {
      candidates = this.searchLayer(query, candidates, layer, visited);
    }

    // Get top-k from final candidates
    const results = candidates
      .map(id => ({
        id,
        score: cosineSimilarity(query, this.vectors.get(id)!)
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, k)
      .map(r => r.id);

    return results;
  }

  private searchLayer(
    query: number[],
    entryPoints: string[],
    layer: number,
    visited: Set<string>
  ): string[] {
    // Greedy search in this layer
    // ... implementation details
  }
}
```

### 3. IVF (Inverted File Index)

**Concept**: Cluster vectors, search only nearest clusters

```
1. Cluster vectors into 1000 groups (k-means)
2. Query arrives
3. Find nearest 10 clusters
4. Search only those clusters (1% of data!)

Trade-off: 99% accuracy, 100x faster
```

## Database Schema

### PostgreSQL with pgvector

```sql
-- Install pgvector extension
CREATE EXTENSION vector;

-- Documents table
CREATE TABLE documents (
  id UUID PRIMARY KEY,
  content TEXT NOT NULL,
  embedding vector(1536),  -- 1536-dimensional vector
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Create index for fast similarity search
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);  -- 100 clusters

-- Search query
SELECT id, content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE metadata @> '{"category": "technical"}'
ORDER BY embedding <=> $1
LIMIT 5;
```

### Pinecone Schema

```typescript
interface PineconeRecord {
  id: string;
  values: number[];  // Embedding vector
  metadata: {
    text: string;
    category: string;
    timestamp: number;
  };
}

// Upsert vectors
await index.upsert({
  vectors: [
    {
      id: 'doc-1',
      values: [0.1, 0.2, ...],
      metadata: { text: 'How to reset password', category: 'support' }
    }
  ]
});

// Query
const results = await index.query({
  vector: queryEmbedding,
  topK: 5,
  filter: { category: 'support' },
  includeMetadata: true
});
```

## RAG Pipeline Implementation

### Basic RAG

```typescript
class RAGSystem {
  constructor(
    private vectorDB: VectorDatabase,
    private llm: OpenAI
  ) {}

  async answer(question: string): Promise<string> {
    // 1. Generate embedding for question
    const questionEmbedding = await getEmbedding(question);

    // 2. Search for relevant documents
    const docs = await this.vectorDB.search(questionEmbedding, 5);

    // 3. Build context from retrieved docs
    const context = docs
      .map((doc, i) => `[${i + 1}] ${doc.content}`)
      .join('\n\n');

    // 4. Generate answer with LLM
    const prompt = `Answer the question based on the context below.

Context:
${context}

Question: ${question}

Answer:`;

    const response = await this.llm.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.7
    });

    return response.choices[0].message.content;
  }
}
```

### Advanced RAG with Re-ranking

```typescript
class AdvancedRAG extends RAGSystem {
  async answer(question: string): Promise<string> {
    // 1. Retrieve more candidates (top 20)
    const candidates = await this.retrieveCandidates(question, 20);

    // 2. Re-rank with cross-encoder (more accurate but slower)
    const reranked = await this.rerank(question, candidates);

    // 3. Use top 5 after re-ranking
    const topDocs = reranked.slice(0, 5);

    // 4. Generate answer
    return await this.generate(question, topDocs);
  }

  private async rerank(
    query: string,
    docs: Document[]
  ): Promise<Document[]> {
    // Use cross-encoder model for precise scoring
    const scores = await Promise.all(
      docs.map(doc =>
        this.crossEncoder.score(query, doc.content)
      )
    );

    return docs
      .map((doc, i) => ({ doc, score: scores[i] }))
      .sort((a, b) => b.score - a.score)
      .map(item => item.doc);
  }
}
```

### Hybrid Search (Vector + Keyword)

```typescript
async function hybridSearch(
  query: string,
  alpha: number = 0.7  // Weight: 0.7 vector, 0.3 keyword
): Promise<Document[]> {
  // 1. Vector search
  const embedding = await getEmbedding(query);
  const vectorResults = await vectorDB.search(embedding, 20);

  // 2. Keyword search (BM25)
  const keywordResults = await elasticsearch.search({
    query: { match: { content: query } }
  });

  // 3. Combine scores
  const combined = new Map<string, number>();

  for (const result of vectorResults) {
    combined.set(result.id, alpha * result.score);
  }

  for (const result of keywordResults) {
    const existing = combined.get(result.id) || 0;
    combined.set(result.id, existing + (1 - alpha) * result.score);
  }

  // 4. Sort by combined score
  return Array.from(combined.entries())
    .sort((a, b) => b[1] - a[1])
    .slice(0, 10)
    .map(([id, score]) => getDocument(id));
}
```

## Document Chunking

**Problem**: LLM context window is limited (4K-32K tokens)

**Solution**: Split documents into chunks

```typescript
function chunkDocument(
  text: string,
  chunkSize: number = 1000,
  overlap: number = 200
): string[] {
  const chunks: string[] = [];
  let start = 0;

  while (start < text.length) {
    const end = start + chunkSize;
    const chunk = text.slice(start, end);
    chunks.push(chunk);

    start += chunkSize - overlap;  // Overlap for context
  }

  return chunks;
}

// Example:
const doc = "Very long document...";
const chunks = chunkDocument(doc, 1000, 200);

// Chunk 1: chars 0-1000
// Chunk 2: chars 800-1800 (200 overlap)
// Chunk 3: chars 1600-2600
```

## API Design

```http
# Upload documents
POST /api/v1/documents
Body: {
  "documents": [
    {
      "id": "doc-1",
      "content": "How to reset your password...",
      "metadata": {"category": "support"}
    }
  ]
}

# Query (RAG)
POST /api/v1/query
Body: {
  "question": "How do I reset my password?",
  "top_k": 5
}
Response: {
  "answer": "To reset your password, go to...",
  "sources": [
    {
      "id": "doc-1",
      "content": "How to reset your password...",
      "score": 0.92
    }
  ]
}

# Similarity search
POST /api/v1/search
Body: {
  "query": "password reset",
  "top_k": 10,
  "filter": {"category": "support"}
}
Response: {
  "results": [
    {
      "id": "doc-1",
      "content": "...",
      "score": 0.95,
      "metadata": {"category": "support"}
    }
  ]
}
```

## Performance Optimization

### Batch Processing

```typescript
// Bad: Process one at a time
for (const doc of docs) {
  const embedding = await getEmbedding(doc.content);
  await vectorDB.insert(doc.id, embedding);
}

// Good: Batch embeddings
const embeddings = await openai.embeddings.create({
  model: 'text-embedding-ada-002',
  input: docs.map(d => d.content)
});

await vectorDB.batchInsert(
  docs.map((doc, i) => ({
    id: doc.id,
    vector: embeddings.data[i].embedding
  }))
);

// 10x faster!
```

### Caching

```typescript
const embeddingCache = new Map<string, number[]>();

async function getEmbeddingCached(text: string): Promise<number[]> {
  const hash = sha256(text);

  if (embeddingCache.has(hash)) {
    return embeddingCache.get(hash)!;
  }

  const embedding = await getEmbedding(text);
  embeddingCache.set(hash, embedding);
  return embedding;
}
```

## Summary

### What We Built:
- Vector database with ANN search
- RAG system for LLM context
- Hybrid search (vector + keyword)
- Document chunking pipeline
- Similarity search with metadata filtering

### Key Concepts:
- ✅ Vector embeddings
- ✅ Similarity metrics (cosine, euclidean)
- ✅ HNSW indexing
- ✅ RAG pipeline
- ✅ Document chunking
- ✅ Hybrid search

### Real-World Examples:
- Pinecone: Managed vector database
- Weaviate: Open-source vector DB
- Chroma: Embedding database for LLMs
- LangChain: RAG framework

### 2026 Relevance:
This is the foundation of modern AI applications:
- ChatGPT plugins use RAG
- GitHub Copilot uses vector search
- Notion AI uses semantic search

---

**Congratulations!** You've completed all 15 system design projects! 🎉

You now have comprehensive knowledge of:
- Distributed systems
- Scalability patterns
- Data storage strategies
- Real-time processing
- Modern AI infrastructure

**Next Steps**:
1. Review projects you found challenging
2. Practice explaining designs out loud
3. Do mock interviews
4. Build simplified versions
5. Share your learnings with others

**Good luck with your interviews!** 💪
