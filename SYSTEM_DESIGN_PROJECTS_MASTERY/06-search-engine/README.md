# Project 06: Distributed Search Engine

## Difficulty: Advanced
**Estimated Time**: 4-5 weeks

## Problem Statement

Design a distributed search engine like Elasticsearch, Solr, or Algolia that indexes documents, provides full-text search with ranking, handles typos, and scales to billions of documents.

**Real-World Example**: Google searches 30+ trillion web pages in <0.5 seconds. We'll build a simpler version that searches millions of documents in <100ms.

## Functional Requirements

### Core Features:
1. **Indexing**: Index documents with multiple fields
2. **Full-Text Search**: Search across all fields with relevance ranking
3. **Filters**: Filter by category, price range, date, etc.
4. **Facets**: Aggregate results (e.g., "10 in Electronics, 5 in Books")
5. **Fuzzy Search**: Handle typos (e.g., "iphne" → "iphone")
6. **Autocomplete**: Suggest completions as user types
7. **Highlighting**: Show matching text snippets

### Out of Scope:
- Web crawling
- Image search
- Semantic search (vector-based)

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Query Latency** | < 100ms | P95 for search queries |
| **Indexing Throughput** | 10K docs/sec | Per node |
| **Index Size** | 1TB | Per shard |
| **Availability** | 99.9% | ~8 hours downtime/year |
| **Freshness** | < 1 second | Near real-time indexing |
| **Relevance** | > 90% | First page contains relevant results |

## Capacity Estimation

### Assumptions:
- **100 million documents** to index
- **100K queries per second** (search-heavy)
- **Average document size**: 10 KB
- **Query:Index ratio = 1000:1** (read-heavy)

### Storage:
```
Raw documents: 100M × 10 KB = 1 TB

Inverted index (3x raw size): 3 TB

With replication (3x): 9 TB

With compression (40%): ~4 TB total
```

### Sharding:
```
Index size per shard: 50 GB (Elasticsearch recommendation)

Number of shards: 1 TB / 50 GB = 20 shards

Queries per shard: 100K QPS / 20 = 5K QPS/shard

Nodes needed (with 3x replication): 20 × 3 = 60 nodes
```

## Core Concept: Inverted Index

### What is it?

```
Documents:
Doc 1: "The quick brown fox"
Doc 2: "The lazy dog"
Doc 3: "Quick brown dogs"

Inverted Index:
Word      → Document IDs
───────────────────────
"the"     → [1, 2]
"quick"   → [1, 3]
"brown"   → [1, 3]
"fox"     → [1]
"lazy"    → [2]
"dog"     → [2]
"dogs"    → [3]

Query "quick brown" → Intersect [1,3] ∩ [1,3] = [1, 3]
```

### Data Structure:

```typescript
interface InvertedIndex {
  term: string;
  postingsList: Posting[];
}

interface Posting {
  docId: number;
  frequency: number;     // How many times term appears
  positions: number[];   // Positions in document (for phrase queries)
  fieldNorm: number;     // Length normalization
}

Example:
{
  term: "elasticsearch",
  postingsList: [
    {
      docId: 123,
      frequency: 3,
      positions: [5, 42, 89],
      fieldNorm: 0.5
    },
    {
      docId: 456,
      frequency: 1,
      positions: [12],
      fieldNorm: 0.3
    }
  ]
}
```

## Indexing Pipeline

```
Document → Analyzer → Inverted Index → Segment → Shard

Step-by-step:
1. Document arrives
2. Tokenize: "Quick Brown Fox!" → ["Quick", "Brown", "Fox"]
3. Lowercase: ["quick", "brown", "fox"]
4. Remove stopwords: ["quick", "brown", "fox"] (keep all)
5. Stem: ["quick", "brown", "fox"] → same
6. Build inverted index
7. Write to segment (immutable file)
8. Merge segments periodically
```

### Analyzer Pipeline:

```typescript
class Analyzer {
  async analyze(text: string): Promise<string[]> {
    // 1. Character filters (remove HTML, etc.)
    text = this.removeHTML(text);

    // 2. Tokenizer (split into words)
    let tokens = this.tokenize(text);

    // 3. Token filters
    tokens = this.lowercase(tokens);
    tokens = this.removeStopwords(tokens);
    tokens = this.stem(tokens);  // "running" → "run"
    tokens = this.dedup(tokens);

    return tokens;
  }

  private tokenize(text: string): string[] {
    // Split on whitespace and punctuation
    return text.split(/[\s\p{P}]+/u).filter(t => t.length > 0);
  }

  private stem(tokens: string[]): string[] {
    // Porter stemmer
    return tokens.map(token => this.porterStem(token));
  }
}
```

## Scoring and Ranking

### TF-IDF (Term Frequency-Inverse Document Frequency)

```
TF-IDF measures how important a word is to a document

TF (Term Frequency):
  How often does term appear in document?
  TF = (count of term in doc) / (total terms in doc)

IDF (Inverse Document Frequency):
  How rare is the term across all documents?
  IDF = log(total documents / documents containing term)

TF-IDF = TF × IDF

Example:
  Query: "elasticsearch tutorial"
  Doc 1: "elasticsearch tutorial for beginners" (10 words)

  Term "elasticsearch":
    TF = 1/10 = 0.1
    IDF = log(100M docs / 1M docs with "elasticsearch") = 2.0
    TF-IDF = 0.1 × 2.0 = 0.2

  Term "tutorial":
    TF = 1/10 = 0.1
    IDF = log(100M / 10M) = 1.0
    TF-IDF = 0.1 × 1.0 = 0.1

  Score = 0.2 + 0.1 = 0.3
```

### BM25 (Best Matching 25) - Modern Standard

```
Improved TF-IDF that handles:
- Diminishing returns (multiple occurrences have less impact)
- Document length normalization

Score = IDF × (TF × (k1 + 1)) / (TF + k1 × (1 - b + b × docLength/avgDocLength))

Where:
- k1 = 1.2 (term frequency saturation)
- b = 0.75 (length normalization)
```

```typescript
class BM25Scorer {
  private k1 = 1.2;
  private b = 0.75;

  score(query: string[], doc: Document): number {
    let score = 0;

    for (const term of query) {
      const tf = doc.termFrequency(term);
      const idf = this.calculateIDF(term);
      const docLength = doc.length;
      const avgDocLength = this.avgDocLength;

      const numerator = tf * (this.k1 + 1);
      const denominator = tf + this.k1 * (1 - this.b + this.b * docLength / avgDocLength);

      score += idf * (numerator / denominator);
    }

    return score;
  }

  private calculateIDF(term: string): number {
    const N = this.totalDocs;
    const df = this.docFrequency(term);  // Docs containing term

    return Math.log((N - df + 0.5) / (df + 0.5) + 1);
  }
}
```

## Fuzzy Search (Typo Tolerance)

### Levenshtein Distance

```
Distance = minimum edits to transform one word into another

"kitten" → "sitting"
1. k→s  (substitute)
2. e→i  (substitute)
3. +t   (insert)
4. +g   (insert)
Distance = 3

For search:
Query: "iphne"
Candidates: ["iphone" (dist=1), "ipad" (dist=4)]
Select: "iphone" (closest match)
```

### Fuzzy Query Implementation:

```typescript
class FuzzyMatcher {
  async search(query: string, maxDistance: number = 2): Promise<string[]> {
    const matches: Array<{ term: string; distance: number }> = [];

    // Get all terms from dictionary
    for (const term of this.dictionary) {
      const distance = this.levenshtein(query, term);

      if (distance <= maxDistance) {
        matches.push({ term, distance });
      }
    }

    // Sort by distance (closest first)
    matches.sort((a, b) => a.distance - b.distance);

    return matches.map(m => m.term);
  }

  private levenshtein(a: string, b: string): number {
    const matrix: number[][] = [];

    // Initialize matrix
    for (let i = 0; i <= a.length; i++) {
      matrix[i] = [i];
    }

    for (let j = 0; j <= b.length; j++) {
      matrix[0][j] = j;
    }

    // Fill matrix
    for (let i = 1; i <= a.length; i++) {
      for (let j = 1; j <= b.length; j++) {
        if (a[i - 1] === b[j - 1]) {
          matrix[i][j] = matrix[i - 1][j - 1];
        } else {
          matrix[i][j] = Math.min(
            matrix[i - 1][j] + 1,     // deletion
            matrix[i][j - 1] + 1,     // insertion
            matrix[i - 1][j - 1] + 1  // substitution
          );
        }
      }
    }

    return matrix[a.length][b.length];
  }
}
```

### N-gram Indexing (Faster Fuzzy Search)

```
Instead of calculating distance for every term,
index substrings (n-grams) for fast lookup:

Word: "iphone"
Trigrams (n=3): ["iph", "pho", "hon", "one"]

Query: "iphne" (typo)
Trigrams: ["iph", "phn", "hne"]

Match: "iphone" has "iph" in common → candidate
Calculate exact distance only for candidates
```

## Autocomplete (Type-ahead)

### Prefix Trie (Tree)

```
Dictionary: ["cat", "car", "card", "care", "careful"]

Trie:
    c
    └─ a
       ├─ t (cat)
       └─ r (car)
          ├─ d (card)
          └─ e (care)
             └─ f
                └─ u
                   └─ l (careful)

Query "car" → Traverse to "r" → Return ["car", "card", "care", "careful"]
```

```typescript
class TrieNode {
  children = new Map<string, TrieNode>();
  isEndOfWord = false;
  frequency = 0;  // For popularity ranking
}

class Autocomplete {
  private root = new TrieNode();

  insert(word: string, frequency: number = 1) {
    let node = this.root;

    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char)!;
    }

    node.isEndOfWord = true;
    node.frequency = frequency;
  }

  suggest(prefix: string, limit: number = 10): string[] {
    let node = this.root;

    // Traverse to prefix
    for (const char of prefix) {
      if (!node.children.has(char)) {
        return [];  // Prefix not found
      }
      node = node.children.get(char)!;
    }

    // Collect all words from this node
    const results: Array<{ word: string; frequency: number }> = [];
    this.dfs(node, prefix, results);

    // Sort by frequency and return top N
    results.sort((a, b) => b.frequency - a.frequency);
    return results.slice(0, limit).map(r => r.word);
  }

  private dfs(
    node: TrieNode,
    prefix: string,
    results: Array<{ word: string; frequency: number }>
  ) {
    if (node.isEndOfWord) {
      results.push({ word: prefix, frequency: node.frequency });
    }

    for (const [char, childNode] of node.children) {
      this.dfs(childNode, prefix + char, results);
    }
  }
}
```

## Sharding Strategy

### Hash-based Sharding

```
Shard = hash(documentId) % numShards

Example:
  3 shards: 0, 1, 2
  Doc "abc" → hash("abc") % 3 = 1 → Shard 1
  Doc "xyz" → hash("xyz") % 3 = 2 → Shard 2

Query process:
  1. Send query to all shards (scatter)
  2. Each shard searches locally
  3. Merge and rank results (gather)
```

### Routing (Custom Sharding)

```
Group related documents together:

Route by tenant: shard = hash(tenantId) % numShards
  All docs for tenant "acme" go to same shard
  Benefit: Tenant-specific queries hit only 1 shard

Route by date: shard = yearMonth
  Shard 0: 2024-01
  Shard 1: 2024-02
  Benefit: Recent searches (common) hit fewer shards
```

## Query Execution

### Single-shard Query

```typescript
class SearchEngine {
  async search(query: string): Promise<SearchResult[]> {
    // 1. Analyze query
    const terms = await this.analyzer.analyze(query);

    // 2. Lookup each term in inverted index
    const postingsLists = await Promise.all(
      terms.map(term => this.index.getPostingsList(term))
    );

    // 3. Intersect posting lists (docs containing ALL terms)
    const candidateDocs = this.intersect(postingsLists);

    // 4. Score and rank
    const scoredDocs = candidateDocs.map(doc => ({
      doc,
      score: this.scorer.score(terms, doc)
    }));

    scoredDocs.sort((a, b) => b.score - a.score);

    // 5. Fetch top N documents
    const topDocs = scoredDocs.slice(0, 10);
    const results = await this.fetchDocuments(topDocs.map(d => d.doc.id));

    return results;
  }

  private intersect(postingsLists: Posting[][]): Posting[] {
    if (postingsLists.length === 0) return [];
    if (postingsLists.length === 1) return postingsLists[0];

    // Sort by length (smallest first for efficiency)
    postingsLists.sort((a, b) => a.length - b.length);

    let result = postingsLists[0];

    for (let i = 1; i < postingsLists.length; i++) {
      result = this.intersectTwo(result, postingsLists[i]);
    }

    return result;
  }

  private intersectTwo(list1: Posting[], list2: Posting[]): Posting[] {
    const result: Posting[] = [];
    let i = 0, j = 0;

    // Two-pointer intersection (both lists sorted by docId)
    while (i < list1.length && j < list2.length) {
      if (list1[i].docId === list2[j].docId) {
        result.push(list1[i]);
        i++;
        j++;
      } else if (list1[i].docId < list2[j].docId) {
        i++;
      } else {
        j++;
      }
    }

    return result;
  }
}
```

### Multi-shard Query (Scatter-Gather)

```typescript
class DistributedSearch {
  private shards: SearchShard[];

  async search(query: string, size: number = 10): Promise<SearchResult[]> {
    // 1. Scatter: Send query to all shards in parallel
    const shardResults = await Promise.all(
      this.shards.map(shard =>
        shard.search(query, size * 2)  // Request more than needed
      )
    );

    // 2. Gather: Merge results from all shards
    const allResults: Array<{ doc: Document; score: number }> = [];

    for (const results of shardResults) {
      allResults.push(...results);
    }

    // 3. Global ranking
    allResults.sort((a, b) => b.score - a.score);

    // 4. Return top N
    return allResults.slice(0, size);
  }
}
```

## Faceted Search

```typescript
interface SearchResponse {
  results: Document[];
  facets: {
    [field: string]: { [value: string]: number };
  };
}

async function searchWithFacets(query: string): Promise<SearchResponse> {
  // 1. Execute search
  const results = await search(query);

  // 2. Aggregate facets
  const facets = {
    category: {},
    brand: {},
    price_range: {}
  };

  for (const doc of results) {
    // Count category
    facets.category[doc.category] = (facets.category[doc.category] || 0) + 1;

    // Count brand
    facets.brand[doc.brand] = (facets.brand[doc.brand] || 0) + 1;

    // Count price range
    const priceRange = this.getPriceRange(doc.price);
    facets.price_range[priceRange] = (facets.price_range[priceRange] || 0) + 1;
  }

  return { results, facets };
}

// Example response:
{
  "results": [...],
  "facets": {
    "category": {
      "Electronics": 45,
      "Books": 23,
      "Clothing": 12
    },
    "brand": {
      "Apple": 20,
      "Samsung": 15,
      "Sony": 10
    },
    "price_range": {
      "$0-$50": 30,
      "$50-$100": 25,
      "$100+": 25
    }
  }
}
```

## Highlighting

```typescript
function highlight(text: string, query: string[]): string {
  let highlighted = text;

  for (const term of query) {
    const regex = new RegExp(`\\b${term}\\b`, 'gi');
    highlighted = highlighted.replace(
      regex,
      match => `<em>${match}</em>`
    );
  }

  return highlighted;
}

// Example:
// Text: "Elasticsearch is a distributed search engine"
// Query: ["elasticsearch", "search"]
// Result: "<em>Elasticsearch</em> is a distributed <em>search</em> engine"
```

## API Design

```http
# Search API
GET /api/v1/search?q=laptop&size=10&from=0
Response: {
  "took": 5,  # milliseconds
  "hits": {
    "total": 1523,
    "max_score": 4.567,
    "hits": [
      {
        "_id": "123",
        "_score": 4.567,
        "_source": {
          "title": "MacBook Pro",
          "price": 1299,
          "category": "Electronics"
        },
        "highlight": {
          "title": ["<em>MacBook</em> Pro"]
        }
      }
    ]
  },
  "aggregations": {
    "categories": {
      "buckets": [
        {"key": "Electronics", "doc_count": 1200},
        {"key": "Computers", "doc_count": 323}
      ]
    }
  }
}

# Index document
POST /api/v1/index/products/_doc
Body: {
  "title": "iPhone 15 Pro",
  "description": "Latest smartphone",
  "price": 999,
  "category": "Electronics"
}

# Bulk indexing
POST /api/v1/_bulk
Body:
{ "index": { "_index": "products" } }
{ "title": "Product 1", "price": 10 }
{ "index": { "_index": "products" } }
{ "title": "Product 2", "price": 20 }

# Autocomplete
GET /api/v1/suggest?prefix=ipho
Response: {
  "suggestions": [
    "iphone",
    "iphone 15",
    "iphone 15 pro"
  ]
}
```

## Performance Optimization

### Caching

```typescript
class SearchCache {
  private cache = new Map<string, CacheEntry>();
  private ttl = 300000;  // 5 minutes

  async get(query: string): Promise<SearchResult[] | null> {
    const cached = this.cache.get(query);

    if (!cached) return null;

    if (Date.now() - cached.timestamp > this.ttl) {
      this.cache.delete(query);
      return null;
    }

    return cached.results;
  }

  set(query: string, results: SearchResult[]) {
    this.cache.set(query, {
      results,
      timestamp: Date.now()
    });
  }
}
```

### Filters Before Scoring

```
Optimization: Filter first, score later

Bad:
1. Score all 10M docs
2. Filter to matching docs
3. Return top 10

Good:
1. Filter to 10K matching docs
2. Score only those 10K
3. Return top 10

10x-1000x faster!
```

## Summary

### What We Built:
- Distributed search engine
- Inverted index for fast lookups
- BM25 scoring for relevance
- Fuzzy search for typos
- Autocomplete with trie
- Sharding for horizontal scaling

### Key Concepts:
- ✅ Inverted index
- ✅ TF-IDF and BM25 scoring
- ✅ Analyzer pipeline
- ✅ Levenshtein distance (fuzzy)
- ✅ Prefix trie (autocomplete)
- ✅ Scatter-gather query execution

### Real-World Examples:
- Elasticsearch: 1 PB+ indexes, <100ms queries
- Algolia: 20K+ queries/sec per cluster
- Solr: Billions of documents indexed

---

**Next Project**: [07. GraphQL Federation Gateway](../07-graphql-gateway/README.md)
