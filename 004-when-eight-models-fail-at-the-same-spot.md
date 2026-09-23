# When eight models fail at the same spot, fix the prompt

*Les Grimoires devlog · September 2026*

**TL;DR**

- We compile gamebooks into structured JSON with an LLM.
- 8 models, same 19 paragraphs: they all diverged from our reference **at the same places**, Gemini included.
- The cause was the prompt: two missing rules and one criterion that was simply **wrong**.

## Context

A structuring pass turns each paragraph into a `sequence_json`: type, branches, opponents, combat flags. We keep a human-reviewed reference and compare models against it.

We ran 8 models on the same 19 paragraphs.

## The signal

Errors weren't random. Independent models made **the same mistakes on the same fields**.

> When independent models all get it wrong at the same point, the cause is in the prompt, not in the models.

Three fields.

## 1. `type`: mechanism beats appearance

Every model classified this as `choice`:

> "If you have a rope, go to X, otherwise go to Y."

Fair reading. But it's an inventory check, so the type is `resolution`. The prompt listed what `resolution` covers, without saying **which wins** when both readings fit.

Fix: an explicit priority rule. As soon as a branch carries a mechanism, the type is `resolution`.

## 2. `opponent`: an unwritten convention

Two identical enemies. Gemini spontaneously wrote "First GOBLIN", "Second GOBLIN". Other models didn't.

The convention came from one model's habit and was **written nowhere**, so nobody else could follow it.

Why it matters: the engine identifies creatures by position; the ordinal is only for narration. Without it, the player reads "GOBLIN hits you" twice and doesn't know which one dies.

Fix: the convention is now in the prompt, **with its reason**.

## 3. `immediate_combat`: the criterion was false

The prompt said:

> `true` if combat starts without prior narration.

We checked our reference: **124 of 144** combat paragraphs are `true`, and the length of the narration before the fight separates nothing (medians 238 vs 474 characters, overlapping ranges).

**The models were correctly applying a wrong rule.**

Fix: replace the criterion with the measured default ("usually true; when in doubt, true").

Still open: the 20 `false` cases share no structural trait. The field is noisy in the reference itself and needs its own review.

## Takeaways

- Run several models on the same inputs. **Where they agree against you is where to look.**
- Write down conventions you rely on, with the why. One model's habit is not a spec.
- Check your criteria against your own reference data. A rule can be precise and false.
- A model disagreeing with the reference is not always the model's fault. Sometimes the reference is noisy.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
