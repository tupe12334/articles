# Why Vector Databases Are Becoming Essential for Modern AI Applications

## TLDR

Traditional databases find rows that match exactly (`WHERE email = 'x'`). AI applications need to find things that mean the same thing, even when the words differ. Vector databases store embeddings — numeric representations of meaning — and let you search by similarity instead of equality. That's the missing piece behind semantic search, RAG, recommendations, and most "AI-powered" features shipping today.

## What's actually different about a vector database

A traditional database indexes exact or range-based values: an ID, a timestamp, a string. A vector database indexes **embeddings** — arrays of floating-point numbers (typically 256–3072 dimensions) produced by an embedding model, where geometric closeness between two vectors corresponds to semantic closeness between the things they represent.

Concretely: the sentences "the cat sat on the mat" and "a feline rested on the rug" will produce embeddings that are close together in vector space, even though they share almost no words. A `LIKE` query would never find that match. A vector database will.

## The connection between embeddings and vector databases

An embedding model (e.g. `text-embedding-3-small`, Cohere's embed models, or open-source options like `bge`) converts text, images, or audio into a fixed-length vector. The vector database's job is purely the search problem that follows: given a query vector, find the `k` nearest stored vectors, fast, at scale — typically using approximate nearest neighbor (ANN) algorithms like HNSW, since exact nearest-neighbor search doesn't scale past a few thousand vectors.

## Popular vector database options

- **pgvector** — a Postgres extension. Best choice if you already run Postgres and don't need to scale past a few million vectors; zero new infrastructure.
- **Pinecone** — fully managed, purpose-built, good default for teams that don't want to operate infrastructure.
- **Qdrant / Weaviate / Milvus** — self-hostable, open-source, better for larger scale or when you need more control.
- **In-memory array + cosine similarity** — genuinely fine for prototypes with a few hundred documents (see the companion RAG tutorial article for exactly this).

## A minimal TypeScript example with pgvector

```typescript
import { Client } from "pg";

const client = new Client({ connectionString: process.env.DATABASE_URL });
await client.connect();

// One-time setup
await client.query(`CREATE EXTENSION IF NOT EXISTS vector;`);
await client.query(`
  CREATE TABLE IF NOT EXISTS documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536)
  );
`);

// Insert
async function insertDoc(content: string, embedding: number[]) {
  await client.query(
    `INSERT INTO documents (content, embedding) VALUES ($1, $2)`,
    [content, JSON.stringify(embedding)]
  );
}

// Search: find the 5 most similar documents to a query embedding
async function search(queryEmbedding: number[], topK = 5) {
  const res = await client.query(
    `SELECT content, embedding <-> $1 AS distance
     FROM documents
     ORDER BY embedding <-> $1
     LIMIT $2`,
    [JSON.stringify(queryEmbedding), topK]
  );
  return res.rows;
}
```

The `<->` operator is pgvector's distance operator (L2 by default; `<=>` for cosine distance). Everything else is ordinary SQL — that's the appeal of pgvector over a standalone vector DB.

## Use cases where this actually matters

- **Semantic search** — "find docs about X" instead of "find docs containing the exact word X."
- **RAG** — giving an LLM relevant context instead of its whole memorized training set.
- **Recommendations** — "users who engaged with this also engaged with..." based on content similarity, not just collaborative filtering.
- **Deduplication** — finding near-duplicate records that don't match on any single field.

## Performance notes

For under ~100k vectors, almost any option (including pgvector with a basic index) performs well enough that the choice barely matters — optimize for operational simplicity instead. Past a few million vectors, index choice (HNSW vs IVFFlat) and a purpose-built vector DB start to matter for both latency and recall.

## Future trends

Expect embedding models to keep shrinking (smaller, cheaper, nearly-as-accurate models), and vector search to keep merging into general-purpose databases rather than staying a separate category — pgvector's rise is the clearest signal of that.

## Feel free to contact me

Happy to go deeper on index tuning or hybrid (keyword + vector) search if there's interest.

## Full Disclosure

I used AI tools to help draft and refine this article's structure and wording; the technical claims and code examples were written and verified by me.
