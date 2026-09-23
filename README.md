# Les Grimoires · devlog

Engineering notes from building an AI game master for gamebooks.

Short posts, real numbers, including what didn't work.

## Posts

| # | Title | Topic |
|---|---|---|
| 001 | [A calibrated classifier in front of an LLM game master](001-classifier-in-front-of-llm-game-master.md) | Routing easy turns to a decision model: 420 ms instead of 9 s, 0 wrong moves |
| 002 | [Why our prompt cache never passed 50%](002-prompt-cache-never-passed-50-percent.md) | Implicit prefix caching in production: DeepSeek, MiniMax, Gemini, an 8k plateau, three hypotheses |
| 003 | [Put the rules last](003-put-the-rules-last.md) | Tool result design: data first, rules last, paraphrase from 1,000+ to ~150 characters |
| 004 | [When eight models fail at the same spot, fix the prompt](004-when-eight-models-fail-at-the-same-spot.md) | Cross-model divergence as a prompt bug detector |
| 005 | ["Which" beats "whether"](005-which-beats-whether.md) | What a decision model can and can't decide: 44/44 vs 0/13 |
| 006 | [Length is where invention grows](006-length-is-where-invention-grows.md) | Why the GM made up a whole night, and three levers against it |
| 007 | [A second opinion at build time](007-a-second-opinion-at-build-time.md) | Finding executable bugs in LLM-generated data for ~$0.15 |
| 008 | [LLM fallback before the first token](008-llm-fallback-before-the-first-token.md) | Switching provider on failure without the player noticing |

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
