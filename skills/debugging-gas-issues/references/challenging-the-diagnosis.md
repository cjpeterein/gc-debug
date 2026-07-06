# Challenging the diagnosis

Debugging fails less from a wrong *observation* than from a right observation promoted too early to "the cause." You find something fishy, adopt it, and pour hours down a rabbit trail chasing a symptom or a real-but-secondary factor. **A diagnosis is a suspect, not a verdict.** The moment a cause looks found is the moment to attack it — not adopt it. This is the cheapest step in the arc and the one most often skipped.

## The failure mode

- **Premature closure** — the first plausible cause ends the search.
- **Confirmation bias** — you look for evidence that *fits*, not evidence that *breaks* it.
- **Correlation mistaken for causation** — "it appeared alongside the symptom" ≠ "it caused the symptom."
- **The refuted-theory chain** — a *second*, *third* "cleaner theory" is the tell: you're pattern-matching the symptom, not measuring the cause.

## The challenge checklist

Before you act on a root-cause claim, state it in one line, then clear all five:

| Gate | The question | If you can't answer it |
|---|---|---|
| **Disconfirming prediction** | What would I observe if this were **NOT** the cause? Now go look for exactly that. | You haven't tested the hypothesis, only admired it. |
| **Un-excluded alternatives** | What else could produce this symptom that I haven't ruled out? | Rule them out (cheaply) or your "cause" is one of several. |
| **Causation, not correlation** | Does *removing this specific thing* actually resolve it — and does it **reproduce** on demand? | You have a correlate, not a cause. |
| **Whole story, or a piece?** | Could there be a *second* path/cause producing the same symptom? (Fixing one path leaves the other.) | Assume there is until you've proven the coverage. |
| **Proven past the window** | Does the fix hold **past the failure's own recurrence window**, not just at first green? | Green ≠ cured. A stopgap re-breaks after the first clean interval. |

The single most useful prompt: **"Name the one observation that would refute this."** If nothing could, it isn't a diagnosis.

## The skeptic pass (the highest-value half)

A motivated debugger rarely challenges itself convincingly — it wants to be done. A **fresh adversary** does it in one cheap pass. Spawn a *second* agent whose only job is to **argue against** the diagnosis:

- Give it the evidence and the claim; instruct it to **refute**, not to agree — default to "not proven" when uncertain.
- Have it attack each checklist gate: propose the disconfirming test, name an un-excluded alternative, question causation, look for a second path.
- If it can't break the diagnosis after a real attempt, your confidence is earned. If it can, you just saved the rabbit trail.

This is the same discipline as the opposing reviewer in a good code review, or the adversarial-verify step in a multi-agent workflow — applied to the *diagnosis* before you invest in the fix. For a high-stakes call, run more than one skeptic and require a majority to *fail* to refute.

## The confidence ladder

Track where a claim actually sits — and refuse to shortcut a rung:

```
suspicion  →  hypothesis  →  tested hypothesis  →  proven root cause
(fishy)       (falsifiable)   (disconfirming        (causation shown +
                               test run)             reproduces + holds
                                                     past the window)
```

Only the top rung earns the word "fixed." Everything below is a lead.

## Calibrate to the cost of acting

The gauntlet scales with how expensive it is to be wrong (see `safety-floor.md`'s cost asymmetry):

- **Reversible poke** (a read-only probe, a dry-run) — a quick self-challenge is enough; act and observe.
- **Irreversible / shared-state / live-primary change** (swapping a binary, deleting rows, a store rebuild) — full checklist **and** an independent skeptic pass before you touch it. Here a wrong diagnosis costs hours or data.

## Worked mini-example — the doltlite guard

The store-bloat root cause was declared "done" three times before it was:

1. Commit-path race found (PR #1540) → *declared the fix*. **Missed gate: whole story?** — there was a second code path (the refresh path) the fix never touched.
2. Refresh path guarded → **"resolved / 15-of-15."** **Missed gate: proven past the window?** — the honest handoff later reframed it as a **stopgap**, with a slower second cause ("RC#2") still open.

A skeptic pass at each "done" — *"what would we see if this weren't the whole cause? does it hold past the recurrence window?"* — turns two premature verdicts into two more rungs of the ladder, before anyone builds on them.

---

**Diagnose freely; act on a diagnosis only after you've tried to break it.** The challenge is minutes; the rabbit trail is hours.
