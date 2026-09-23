# "Which" beats "whether": what a decision model can and can't decide

*Les Grimoires devlog · September 2026*

**TL;DR**

- We use Jev, a decision model (no text, typed answers, calibrated probabilities), in play and at build time.
- **"Which one of these?"** questions: near perfect. 44/44, 51/51, 20/20, 11/11.
- **"Should we do X?"** questions: failed. 0/13 recall on the one we cared about.
- Same model, same calls, same data. The difference is the shape of the question.

## The three primitives

Jev ([docs](https://docs.typesafe.ai/primitives)) takes a JSON `state` and typed questions, all in one call:

| Type | You give | You get |
|---|---|---|
| `choice` | `criteria`: **object** `{label: description}` | `choice`, `probabilities`, `confidence` |
| `noul` | `instructions` only | probability of "yes". **No `confidence`** |
| `score` | `criteria`: **array** of scale levels | `score`, `legend`, `probabilities`, `confidence` |

## "Which": it works

**Picking a tool and its argument.** Five-item inventory, real item ids in the `state`, two `choice` questions per call (~330 ms):

| Player says | Tool | Item |
|---|---|---|
| "I draw my weapon" | `inventoryEquip` 0.98 | `sword` 0.93 |
| "I heal myself" | `inventoryUse` 1.00 | `potion-of-healing` 0.98 |
| "I light the lantern" | `inventoryUse` 0.87 | `lantern` 1.00 |
| "What's in my bag?" | `none` 1.00 | `none` 0.99 |

**11/11** on clear cases, tool **and** id. "I heal myself" finds the potion without the word "potion".

**Classifying branches at build time.** Which of 7 mechanisms does this branch use? On two reviewed books, 1,520 branches:

| Mechanism | Agreement |
|---|---|
| `die` | 44/44 |
| `inventory` | 51/51 |
| `stat_test` | 20/20 |
| `continuation` | 279/290 |
| `choice` | 915/965 |

## "Whether": it doesn't

Same calls, four yes/no checks added (`noul`), one book, 398 paragraphs:

| Check | Result |
|---|---|
| "Does this paragraph impose a rule outside the data?" | **0/13** recall at 0.8. At 0.3: 5/13 for 43 false alarms |
| "Does combat start immediately?" | median 0.68 on yes, **0.66 on no**. No separation |
| "Is this effect conditional?" | 6/7 recall, 0 false alarms at 0.5. Only 7 positives: proves nothing |
| "Is a collectible missing?" | 0 positives, 0 alarms. Silence, not success |

**Excellent for "which", weak for "whether".** If we come back to these, it'll be as a `choice` over described categories, never a yes/no.

## Limits we hit

- **Mono-label per question.** "I drink the potion and draw my sword" → `inventoryUse` at **0.60**. Not an error, a drop in confidence. The threshold catches it; no setting will represent it.
- **No invented ids.** "I use the magic item" → tool 0.92, item `none` 0.78. It doesn't make up an item.
- **Uncertainty is honest.** "I throw my sword away" (no tool does that) → 0.73. The hesitation says it.

## Cost gotchas

- Price follows the **`state`**, not the number of questions. +4 `noul` = +237 tokens, no extra latency.
- But `choice` **criteria are billed per question**. Longer descriptions: **+29% tokens**. Moving shared definitions into the `state`: **-28%**, same quality (95.3% vs 95.5%).
- A bloated `state` also **lowers precision**. Don't dump the whole character sheet.

## API gotchas

- `criteria` is an **object** for `choice`, an **array** for `score`, **absent** for `noul`. Wrong shape = 400 with an unreadable zod error.
- `noul` returns no `confidence`.
- `score` returns the **expected index** on your scale, not a 0 to 1 value. `0.95` on a 4-level scale means "level 1", not "95%".

## Takeaways

- Ask "which of these?" over a closed, real list. Avoid "should I?".
- Always give an escape label (`none`): a closed list forces an answer.
- Read `confidence`, not just the label.
- Put shared definitions in the `state`, not in every question.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
