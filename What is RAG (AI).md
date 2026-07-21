# What is RAG (AI)?

## TLDR

Retrieval-Augmented Generation (RAG) is a technique for grounding a language model's answers in your own data. Instead of relying only on what the model memorized during training, RAG looks up relevant documents at query time and feeds them into the prompt, so the model answers from real, up-to-date sources instead of guessing.

## Background story

A friend asked me why every AI product these days seems to mention "RAG" in its pitch deck. It's not a new model, not a new company, not even a new algorithm in the deep-learning sense, it's a pattern for wiring a language model up to a knowledge base. Once you see the pattern, you start noticing it everywhere: chatbots that answer questions about a company's internal docs, coding assistants that reference your actual codebase, support bots that quote your changelog instead of hallucinating a feature that doesn't exist.

## The problem RAG solves

A language model's knowledge is frozen at training time and limited to whatever text it was trained on. Ask it about your company's internal API, a document you wrote yesterday, or anything published after its training cutoff, and it will either say "I don't know" or, worse, confidently make something up. Fine-tuning a model on your data is one fix, but it's expensive, slow to update, and still doesn't guarantee the model will recall a specific fact correctly.

RAG sidesteps this by not asking the model to *remember* your data at all. It asks the model to *read* it, on demand, as part of the prompt.

## How RAG differs from a plain LLM call

A plain call to a model looks like this:

```typescript
const answer = await llm.complete({
  prompt: `Answer this question: ${question}`,
});
```

The model only has its training data and the question to work with. A RAG pipeline adds a retrieval step before the model ever sees the question:

```typescript
async function ragAnswer(question: string) {
  // 1. Turn the question into a vector and search a knowledge base
  const relevantChunks = await vectorStore.search(question, { topK: 5 });

  // 2. Stuff the retrieved text into the prompt as context
  const context = relevantChunks.map((c) => c.text).join("\n\n");

  // 3. Ask the model to answer using only that context
  return llm.complete({
    prompt: `Using only the context below, answer the question.

Context:
${context}

Question: ${question}`,
  });
}
```

That's the whole idea: retrieve, then generate. The "R" and the "G" in RAG.

## The core components

- **A knowledge base**: your documents, split into chunks (paragraphs, sections, or pages).
- **An embedding model**: converts each chunk, and the incoming question, into a vector that captures its meaning.
- **A vector store**: an index that can quickly find the chunks whose vectors are closest to the question's vector (i.e. the most semantically relevant ones).
- **A generator**: the language model that reads the retrieved chunks alongside the question and writes the final answer.

## Why it matters

- **Fewer hallucinations**: the model is answering from text it was just handed, not from memory, so it's far less likely to invent facts.
- **Up-to-date answers**: update the knowledge base and the next query immediately reflects the change, no retraining required.
- **Traceability**: because you know which chunks were retrieved, you can show the user exactly which source an answer came from.
- **Cheaper than fine-tuning**: indexing documents into a vector store is far cheaper and faster than retraining or fine-tuning a model.

## Limitations

RAG isn't magic. If retrieval pulls back the wrong chunks, the model will confidently answer from the wrong context. Chunking strategy, embedding quality, and how many chunks you retrieve all meaningfully affect answer quality, and tuning them is its own skill. RAG also doesn't help with tasks that require reasoning across information that was never retrieved in the first place.

## Feel free to contact me

I hope this explanation of RAG has been helpful! If you have any questions or would like further information, please don't hesitate to ask.

## Full Disclosure

I used AI tools to help me write this article. These tools assisted in the writing process by suggesting alternative phrasing, but the content of the article is entirely my own work.
