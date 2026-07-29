# A brief overview about Agents

## TLDR

An AI agent is a language model wrapped in a loop: give it a goal and a set of tools, and instead of returning one answer it decides what to do next, calls a tool, looks at the result, and repeats until the goal is done. The shift from a regular model is the shift from "answer a question" to "take actions toward an outcome."

## Background story

I kept hearing "agent" used to describe wildly different things, a chatbot, a background script, a whole product, and it took me a while to realize they're often the same underlying pattern wearing different clothes. Once I built one myself, the term stopped being fuzzy: an agent is just a model that gets to act, observe the result of its action, and decide again.

## What makes something an "agent"

A regular model call is a single request/response: you send a prompt, you get text back, done. An agent adds a loop around that call, and gives the model tools it can invoke:

```typescript
async function runAgent(goal: string, tools: Tool[]) {
  const messages = [{ role: "user", content: goal }];

  while (true) {
    const response = await llm.complete({ messages, tools });

    if (response.type === "final_answer") {
      return response.text;
    }

    // The model chose a tool call, run it and feed the result back
    const tool = tools.find((t) => t.name === response.toolName);
    const result = await tool.run(response.args);

    messages.push({ role: "assistant", content: response.raw });
    messages.push({ role: "tool", content: JSON.stringify(result) });
  }
}
```

Each iteration, the model sees everything that happened so far, including the outcome of its own previous actions, and decides whether it's done or needs to act again. That loop, plan, act, observe, repeat, is the whole idea behind "agentic" AI.

## Types of agents

- **Single-tool agents**: a narrow loop around one capability, e.g. a research assistant that can only search the web.
- **Multi-tool agents**: a model with a toolbox (search, code execution, file access, APIs) that picks whichever tool fits the current step.
- **Multi-agent workflows**: several specialized agents (a planner, a coder, a reviewer) that hand work off to each other rather than one model doing everything.

## Key components of agent architecture

- **A goal or task**: what the agent is trying to accomplish.
- **Tools**: functions the model can call, each with a clear name, description, and input schema so the model knows when and how to use it.
- **Memory / context**: the running transcript of what's been tried and what came back, so the agent doesn't repeat itself or lose track of progress.
- **A stopping condition**: something that tells the loop when the goal is met (or when to give up), otherwise the agent can loop forever.

## Real-world applications

- Coding assistants that read a codebase, run tests, and fix failures in a loop instead of writing code blind.
- Research agents that search, read, and synthesize sources over several steps instead of one search query.
- Support and ops agents that look up records, take actions in internal systems, and confirm the outcome before responding.

## Limitations

Agents compound errors: a wrong decision at step two can send the whole loop down the wrong path, and each extra step is another chance to go off track or burn time and tokens. They also need real guardrails, since giving a model the ability to act (send an email, delete a file, spend money) means a bad decision has real consequences, not just a wrong sentence. Good stopping conditions and human checkpoints matter as much as the model itself.

## Feel free to contact me

I hope this overview of AI agents has been helpful! If you have any questions or would like further information, please don't hesitate to ask.

## Full Disclosure

I used AI tools to help me write this article. These tools assisted in the writing process by suggesting alternative phrasing, but the content of the article is entirely my own work.
