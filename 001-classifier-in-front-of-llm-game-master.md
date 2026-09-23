# A calibrated classifier in front of an LLM game master

*Les Grimoires devlog · September 2026*

**TL;DR**

- We put a fast decision model (Jev) in front of our LLM game master.
- 13% of turns now skip the LLM: **~420 ms instead of ~9 s**.
- **0 wrong moves** over 139 played turns and 159 replayed turns.
- On the bill it's slightly negative. We did it for latency and reliability, not cost.

## Context

Les Grimoires runs gamebooks with an LLM game master (GM).

- The book is compiled into sequences (paragraphs) with typed branches.
- The GM is an agent with ~20 tools: dice, combat, stats, inventory, NPC memory, and `initSequence` to move to the next paragraph.
- The player types freely: "I take the left door", "I ask the old man about the key".

The most frequent GM failure: **the player picks a branch and the GM doesn't move.** It narrates the move, sometimes even writes `[SEQUENCE]`, but never calls `initSequence`.

## The idea

Most turns on a choice paragraph are a classification problem:

> Did the player pick one of these branches, or are they doing something else?

That doesn't need a 12k-token prompt and 9 seconds.

## Jev in one paragraph

[Jev](https://docs.typesafe.ai/primitives) (`typesafe/jev-1.13`, by TypeSafe, served through OpenRouter's alpha `decisions` endpoint) is a decision model:

- no text generation;
- input: a JSON `state` + typed questions;
- output: a typed answer + **calibrated probabilities**.

Measured on our traffic:

| | |
|---|---|
| Latency | 250 to 600 ms, median 424 ms |
| Cost | $0.000054 per call |
| Context | 32k |

## What we ask

One call, one `choice` question named `route`:

```js
questions: {
  route: {
    type: "choice",
    instructions: "What does the player do with this message?",
    criteria: {
      branch_0: "Takes the left door. Leads to: <300-char preview of the target>",
      branch_1: "Goes back to the crossroads. Leads to: <preview>",
      dialogue: "Talks to a character present in the scene",
      exploration: "Looks around or acts without choosing any branch",
      dice: "Answers a dice roll",
      meta_question: "Asks about the game itself",
      ambiguous: "Cannot be decided from the message"
    }
  }
}
```

If two or more NPCs are present, a second `choice` question (`interlocutor`) rides in the **same call**. Extra questions cost almost nothing: price follows the size of the `state`, not the number of questions.

## The routing rule

```
branch with p >= 0.8  AND  simple target  ->  server calls initSequence, no LLM
anything else                             ->  normal LLM turn (+ Jev's guess as a hint)
Jev down, slow (> 1.5 s) or no key        ->  normal LLM turn
```

"Simple target" = no combat, no dice roll, no conditional effect, no ending. That covers **78%** of branches in our compiled books.

Two guards:

- **NPC present**: never route. An "ok" said while arriving would skip the conversation.
- **Branch gated by a hidden variable**: never route. Jev doesn't see the game state.

## How we tuned it: offline replay

Before shipping, we replayed **159 real turns** and compared Jev's decision with what the LLM had done.

| Version | Change | Decided / agree / "moves when the LLM stayed" (threshold 0.9) |
|---|---|---|
| v1 | branch labels + `none` / `ambiguous` | 73 / 63 / 7 |
| v2 | show **where each branch leads** (300-char preview) | 81 / 72 / 6 |
| v3 | define what "continue" is **not** | 82 / 73 / 6 |
| v4 | split `none` into dialogue / exploration / mechanic / meta | 86 / 79 / 4 |
| **v5** | exploration only if no branch describes the action | **85 / 79 / 3** |
| v6 | two passes (intent type, then branch) | 81 / 76 / 2, too cautious, dropped |

The 3 remaining disagreements in v5? **All LLM mistakes.** "Yes" to "Do you continue on your way?" is a move; the LLM had stayed.

## Results in play

91 GM turns (staging + local test games), medians:

| | LLM turn | Routed turn |
|---|---|---|
| Input tokens | 11,839 | **0** |
| Output tokens | 275 | **0** |
| Latency | 9,137 ms | **420 ms** |

- **13%** of turns routed, **19x** faster.
- **0** wrong routing decisions. Every defect we tracked came from the GM.

## The threshold is doing the real work

Of the branches Jev saw but didn't route, **11 of 15 were blocked by confidence**, not by target complexity. And none of them should have been routed:

```
0.50  "I run to the nearest village"        (there is no village)
0.54  "I down both glasses"                 (branch 0.42 / dialogue 0.32)
0.75  "I walk back out"                     (an NPC blocks the door)
0.78  "I'll follow you wherever you want"   (the NPC decides, not the player)
```

Reading `choice` and ignoring `confidence` would have moved the player to a village the book doesn't have.

**The calibration is the product. The label alone is dangerous.**

## The honest part: cost

Jev is called on **every** eligible turn and saves only 13% of LLM calls:

- saved: $0.000032 per turn;
- spent: $0.000054 per turn.

**Slightly negative.** What we bought:

- latency on the easy turns;
- a GM that no longer "forgets" to move.

The real token sink is elsewhere: 30% cache hit, 230 reasoning tokens for 45 written. Jev doesn't touch that.

## Lessons

- **Describe the criteria, don't just name them.** Showing where each branch leads gave the biggest jump (73 to 81). Jev writes nothing, so there's no spoiler risk.
- **Define the negative.** "Continue" means nothing. "Agreeing to keep talking is NOT moving on" fixed it.
- **One pass beats two.** Competing categories add up; the detail is just information.
- **Mono-label per question.** "I drink the potion and draw my sword" can't be represented. It shows up as low confidence (0.60), not as an error. The threshold handles it.
- **Your LLM is not the ground truth.** Every "disagreement" needs a human look. Here, they were all LLM bugs.
- **Always have a fallback.** Alpha API, single provider: if Jev fails, the turn goes to the LLM as before.

## What's next

- Shadow mode: 10% of turns Jev would route still go to the LLM, to keep comparing where Jev is confident.
- Tool picking: first tests on inventory ("I heal myself" → `potion-of-healing` without the word "potion"): 11/11 on clear cases.
- Next post: why "which one?" questions work and "should I?" questions don't.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
