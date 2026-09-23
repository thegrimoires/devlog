# Les Grimoires · devlog

Engineering notes from building an AI game master for gamebooks.

Short posts, real numbers, including what didn't work.

## Posts

| # | Title | Topic |
|---|---|---|
| 001 | [A calibrated classifier in front of an LLM game master](001-classifier-in-front-of-llm-game-master.md) | Routing easy turns to a decision model: 420 ms instead of 9 s, 0 wrong moves |

## What we're building

Les Grimoires plays gamebooks with an LLM game master:

- the book is compiled into paragraphs with typed branches;
- the GM is an agent with ~20 tools (dice, combat, stats, inventory, NPC memory);
- the player types freely, the GM narrates and applies the rules;
- the AI follows the authored story, it doesn't invent a new one.

## Stack

- **Client**: Vue 3
- **Server**: Node.js, Fastify, SSE streaming
- **LLM layer**: Vercel AI SDK, several providers
- **Decision model**: Jev (TypeSafe) for fast, calibrated classification
- **Campaign pipeline**: Python, book to structured campaign
- **Evals**: in-house harness for tool-calling reliability

## Topics we write about

- Making an LLM agent actually call its tools
- Calibrated classifiers vs. LLMs for routing
- Evaluating non-deterministic systems
- Cost and latency of an agent in production

## Links

- 🎲 Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)
- ✉️ contact@lesgrimoires.fr
