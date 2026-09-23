# Why our prompt cache never passed 50%

*Les Grimoires devlog · September 2026*

**TL;DR**

- Our LLM game master resends a growing prompt every turn: system prompt, tools, full history.
- On paper, most of it should hit the provider's prompt cache. In practice: **43 to 48%** with DeepSeek, **90%** with MiniMax, **well under 10%** with Gemini. Same engine.
- On DeepSeek the cache **plateaus around 8k tokens**: the fixed prefix is reused, the history almost never.
- No conclusion yet. This is a status report, with numbers and three hypotheses we'll test next.

## The setup

Each GM turn sends:

```
[system prompt ~6k tokens][tools][history .............][state block]
 └──────────── should be cached ─────────────────────┘   └ changes every turn
```

- **Implicit, prefix-based caching** on all providers. No `cache_control` anywhere.
- Everything that changes per turn (game state, detected player intent) goes in a **state block, last message**, outside the prefix.
- A cached token costs **~10%** of a fresh one.

## The data

362 GM turns from staging and prod, read from the `usage` metadata of each message.

| Model (via) | Turns | Cache hit (weighted) | Turns at 0% | Turns > 90% |
|---|---|---|---|---|
| MiniMax M3 (free pool) | 55 | **90%** | 0% | 78% |
| DeepSeek V4 Flash (OpenRouter), prod | 70 | **48%** | 36% | 10% |
| DeepSeek V4 Flash (OpenRouter), staging | 100 | **43%** | 41% | 9% |

Caveat: the MiniMax turns are from early September, the DeepSeek ones from the following two weeks. The prompt changed in between. Not a clean A/B.

**Gemini** isn't in this dataset: we tested it earlier, outside these logs. What we saw:

- cache hit **well under 10%**;
- when it did show up, it often took **2 to 3 turns** to kick in.

Worst of the three, on the same kind of prompt.

## What we see

**1. It's not cache expiry.**
Median gap before a 0% turn: 1.5 min. Before a hit: 1.1 min. Misses don't follow long pauses.

**2. The cache plateaus at ~8k tokens.**
The most frequent cached amount on DeepSeek sits around 8,192 tokens. One game, consecutive turns:

```
input   cached
10,672  8,192
10,891  8,192
11,138  8,192
11,301  8,192
11,554  8,192
```

The history grows, the cached part doesn't move. The fixed prefix (system + tools) is reused; the history behind it isn't. Median: a turn reuses **42%** of what the previous turn sent.

**3. Cross-session reuse works.**
6 of 9 first turns already have cache hits. The shared system prompt is cached across players.

**4. Shrinking history hurts.**
When the prompt is shorter than the previous turn (history trimmed), 59% of turns are at 0%, against 40% overall.

## The math that changed our design

Should we inject the combat rules only during fights? Let X be their size:

| Strategy | Cost |
|---|---|
| Always in the system prompt, cached | `0.1 × X × every turn` |
| Only during fights, in the state block | `1.0 × X × fight turns` |
| Swap the system prompt in fights | above **+ a full prefix break** on entry and exit |

On-demand only wins if fights are **under ~10%** of turns. They aren't.

**A token sleeping in a cached prefix is 10x cheaper than one woken up on demand.** And never change the system prompt mid-game: it invalidates the whole history behind it.

## Side note: caching can cost more than it saves

On Gemini 3.5 Flash-Lite, cache **storage** is billed $1.00 per million tokens per hour, twice the Flash rate. On short runs, explicit caching costs more than it saves. Check storage pricing, not just read pricing.

## Hypotheses

**A. OpenRouter spreads DeepSeek across several hosts, each with its own cache.**
Switching host between turns means 0%, regardless of timing. That would explain ~40% of misses.
*Test:* pin the host (`provider.order`, `allow_fallbacks: false`) and measure again.

**B. Something rewrites the start of the history each turn.**
History trimming, narration removed from past tool results, a state block leaking into history. Any byte change early in the history breaks everything after it. That would explain the 8k plateau.
*Test:* log a hash of each message sent, per turn, and find the first one that changes.

**C. Gemini's implicit cache needs a warm-up.**
The 2 to 3 turns before the first hit suggest the cache isn't written on first sight of a prefix. With a prefix that keeps shifting (hypothesis B), it may never get the chance.
*Test:* same game replayed on Gemini with a frozen history, then with explicit caching (`cachedContent`), watching storage cost.

## What we're missing in our logs

- **The actual upstream host** served by OpenRouter.
- **Per-step usage.** A turn can have several steps (tool calls), and we only store the sum.

Both are cheap to add. Without them, we're guessing.

## Next

1. Log host + per-step usage.
2. Pin the DeepSeek host for a week.
3. Hash the prefix per turn.
4. Follow-up post with the answer, whatever it is.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
