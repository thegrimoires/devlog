# A second opinion at build time: finding executable bugs in LLM-generated data

*Les Grimoires devlog · September 2026*

**TL;DR**

- An LLM turns each gamebook paragraph into structured JSON. Some fields are wrong, and some wrong fields become **bugs the server executes**.
- A decision model (Jev) re-reads the closed fields as a second pass.
- On a book it had never seen: **96.1% agreement**, 13 flags on 613 branches, **3 real bugs** our deterministic checks missed.
- Full corpus re-check: **~$0.15**.

## Why it matters now

Our server routes some player moves **without the LLM** (see post 001). It only does so on branches typed `choice` or `continuation`.

So a branch with a dice roll mislabeled `choice` becomes a server-executed bug. Real example: a paragraph said "roll a die: 1-2 → A, 3-6 → B". The data had two `choice` branches. A player typing "I pick B" was routed there **without rolling**, choosing their own outcome.

Before routing, the LLM might have noticed. Now nothing did.

## The split

| Jev **writes** | Jev **flags** | Left to the LLM |
|---|---|---|
| closed fields: `lucky`, `stat`, `comparison` | a contested `kind`, a target out of schema | labels, text, opponents, deltas, dice ranges |

`comparison` is the one field we really took away from the LLM. Its failure was systematic: it **copied the operator as written**. When the sentence puts the stat first ("if your stat is equal to or greater than the roll"), the engine evaluates `roll OP stat`, and both branches flip silently.

The fix: **don't ask for the operator, ask about the world.**
1. "Must the dice total exceed the stat?"
2. "Does a tie count?"

Neither answer depends on word order. LLM: wrong on all 3 such cases. Jev: 34/34.

## Measuring against real ground truth

Comparing two models tells you little (91.2% agreement, disagreements fall on both sides). Better: a book where **human corrections are in git**. The original version is a commit, the fixes are a diff.

Jev replayed on the original, 401 paragraphs, $0.0376:

| | |
|---|---|
| Human fixes it could express | **12/12** |
| New defects, still in the data | **6** |
| False alarms | **4** |

18 right out of 22. The false alarms all sit on the same border: player choice mixed with a mechanism ("or do you have an item you could use?" is a `choice`, not an `inventory`). That's why `kind` **flags instead of fixing**.

## On an unseen book

354 paragraphs, 613 branches, $0.026:

- 96.1% agreement;
- 13 flags, 7 worth handling, **3 executable bugs**:
  - an inventory gate whose "otherwise" branch was left as `choice`;
  - "if your stat is 6 or more → A, otherwise → B" typed `choice` twice: the player picks their own stat tier, and the server routes it.

Our rule-based validator flagged **none** of these. That's exactly the class of error rules don't see.

## Three design mistakes, all ours

Each time, the candidate list didn't match the real enumeration:

- filtering stats on what the text names: OCR noise turned a stat name into garbage, so the right answer wasn't offered;
- completing with an uppercase regex: garbage served as a candidate, and picked at 0.54;
- building the item catalog from collectibles only: an item never picked up wasn't in the list.

> Never offer less, or anything other, than the schema's enumeration.

## Escape hatch

A closed enum **forces an answer**. On dice-vs-dice comparisons that no `kind` fits, Jev picked `die` at 0.99 and filled overlapping ranges. The LLM left the field empty and got caught by a validator: **failing visibly is better**.

Fix: a `none` label ("no mechanism in the schema covers this"), with the same 0.8 threshold. Without the threshold, 29 low-confidence "none" drowned the 2 real ones.

## Method: two mistakes not to repeat

- **Freeze the sample before iterating on criteria.** A bug fix changed the stratification; "95.3% → 97.5%" compared two different populations.
- **Don't take examples from the disagreements you then evaluate.** Only a book never seen gives an honest number.

## Takeaways

- Where LLM output drives execution, add a cheap second opinion on the closed fields.
- Ask about the world, not about the syntax.
- Flag on fuzzy borders, fix only where the model is solid.
- Always offer an escape label, with a threshold.
- Measure against human corrections, not against another model.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
