# How to Implement Your First RAG System in 10 Minutes

## TLDR

RAG (Retrieval-Augmented Generation) lets an LLM answer questions using your own documents instead of only what it memorized during training. You embed your documents into vectors, store them, retrieve the closest matches to a question, and hand those matches to the LLM as context. Below is a minimal but complete TypeScript implementation you can run in about 10 minutes.

## Background story

A common first question when someone starts working with LLMs is: "how do I make it answer questions about *my* data?" Fine-tuning is slow and expensive. RAG is the practical shortcut most production systems actually use, and it's simpler to stand up than people expect.

## What is RAG, in one paragraph

RAG has three moving parts:

1. **Retriever** — turns your documents (and the user's question) into vectors (embeddings), then finds the documents whose vectors are closest to the question's vector.
2. **Context assembly** — takes those top matches and stuffs them into a prompt.
3. **Generator** — the LLM, which answers using the retrieved context instead of guessing from memory alone.

## Tools for this walkthrough

- Node.js + TypeScript
- An embeddings + chat model API (examples below use the Anthropic/OpenAI-style HTTP shape — swap in whichever provider you use)
- No vector database required — for a handful of documents, an in-memory array with cosine similarity is enough. Swap in a real vector DB (see the companion article on vector databases) once you outgrow this.

## Step 1 — Embed your documents

```typescript
type Doc = { id: string; text: string; embedding: number[] };

async function embed(text: string): Promise<number[]> {
  const res = await fetch("https://api.openai.com/v1/embeddings", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
    },
    body: JSON.stringify({ model: "text-embedding-3-small", input: text }),
  });
  const json = await res.json();
  return json.data[0].embedding;
}

async function buildIndex(texts: string[]): Promise<Doc[]> {
  return Promise.all(
    texts.map(async (text, i) => ({
      id: String(i),
      text,
      embedding: await embed(text),
    }))
  );
}
```

## Step 2 — Retrieve the closest documents

```typescript
function cosineSimilarity(a: number[], b: number[]): number {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}

function retrieve(index: Doc[], queryEmbedding: number[], topK = 3): Doc[] {
  return [...index]
    .sort((a, b) => cosineSimilarity(b.embedding, queryEmbedding) - cosineSimilarity(a.embedding, queryEmbedding))
    .slice(0, topK);
}
```

## Step 3 — Generate an answer using the retrieved context

```typescript
async function answer(question: string, index: Doc[]): Promise<string> {
  const queryEmbedding = await embed(question);
  const topDocs = retrieve(index, queryEmbedding);
  const context = topDocs.map((d) => d.text).join("\n---\n");

  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": process.env.ANTHROPIC_API_KEY!,
      "anthropic-version": "2023-06-01",
    },
    body: JSON.stringify({
      model: "claude-sonnet-5",
      max_tokens: 500,
      messages: [
        {
          role: "user",
          content: `Answer the question using only the context below. If the context doesn't contain the answer, say so.\n\nContext:\n${context}\n\nQuestion: ${question}`,
        },
      ],
    }),
  });
  const json = await res.json();
  return json.content[0].text;
}
```

## Step 4 — Wire it together

```typescript
const index = await buildIndex([
  "Our refund policy allows returns within 30 days of purchase.",
  "Shipping takes 3-5 business days within the continental US.",
  "We offer a 1-year warranty on all electronics.",
]);

const result = await answer("How long do I have to return something?", index);
console.log(result);
// -> "You have 30 days from the date of purchase to return an item."
```

That's the whole system: embed, retrieve, generate. No framework required.

## Testing it

Try a question that isn't covered by your documents ("what's your CEO's favorite color?") — a correctly-built RAG prompt should say it doesn't know, rather than hallucinating. If it hallucinates, tighten the prompt instructions or lower `topK` so irrelevant context doesn't leak in.

## Where to go from here

- Swap the in-memory array for a real vector database once your document count grows past a few hundred (see the companion article on why vector databases matter).
- Add chunking for long documents instead of embedding them whole.
- Add a re-ranking step if retrieval quality needs improvement.

## Feel free to contact me

If you build this and get stuck, or want to go deeper on chunking strategies or re-ranking, feel free to reach out.

## Full Disclosure

I used AI tools to help draft and refine this article's structure and wording; the code examples were written and verified by me.
