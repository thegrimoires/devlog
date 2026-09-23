# Length is where invention grows

*Les Grimoires devlog · September 2026*

**TL;DR**

- In one test game, our GM stayed **9 turns** on a paragraph with two exits, inventing a bedroom, a staircase, a whole night.
- The reasoning trace showed three distinct holes in the prompt, not a dumb model.
- Three levers: a **length budget**, one **invariant** instead of a list of bans, and an **honest answer** to "what can I do?".

## What happened

The paragraph: two branches, both in the same room. The GM invented a room upstairs, a stairway, a night's sleep. Nine turns, none of them in the book.

Turns per paragraph in that game: `2 1 6 1 1 6 4 9`. The 9 is the drift.

Our decision model classified every player message correctly. The problem was the GM.

## Three holes, read from the reasoning

**Turn 24, the slip.**
> "This is dialogue with [NPC]. Stay in character [...] offering a room is fine."

The dialogue instructions pointed to the NPC roleplay rules, not to the "invention limits" section. And the rule "an NPC never grants an exit" only covered **improvised** NPCs. This one was from the book, so not covered.

**Turn 27, contradictory rules.**
> "But consigne says don't list branches. However softlock rule says surface branches as suggestions."

It couldn't name the real exits, so it invented one ("go to sleep"). **The rule banning the list forced the lie.**

**Turn 28, no way out.**
> "I cannot guess sequence numbers."

The story was in a bedroom, the exits in the living room. The only move left was "answer in the scene and stay".

## The number that pointed at length

Average GM message length:

| Turn type | Characters |
|---|---|
| The GM moves the player (the book narrates) | **319** |
| The GM stays | **736** (up to 1,137) |

The four longest turns of the game were the four turns of the drift. Dialogue was budgeted in "lines", a unit with no size.

## The three levers

**1. Budget.** A `How much you write` section: two sentences max, always, with a table per situation.
> Length is where invention grows.

**2. Invariant.** Replace the list of bans with one rule that holds for everyone (player, book NPC, improvised NPC, GM):
> The sequence places the player. Only a branch moves them, in space and in time.

**3. Honest answer.** "What can I do?" is answered **from the actual branches**, told in the fiction, never as a list. The golden rule went from "never show the options" to "never show the options **as a list**".

Plus a counter: `turns_on_sequence`, incremented by the server each turn, reset on entering a new paragraph, visible to the GM in the state block. The GM knew the anti-softlock rule (it cited it 4 times) but had **no way to count**.

## Takeaways

- Cap length explicitly. Long answers are where made-up content lives.
- One invariant beats a list of bans. Lists always miss a case.
- A ban that leaves no honest answer produces a dishonest one.
- If a rule depends on a count, give the model the count.
- Read the reasoning trace. It tells you which rule failed and why.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
