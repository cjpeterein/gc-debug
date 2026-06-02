# Upstreaming a Gas* fix

A local patch unblocks you now; the durable home is upstream. The sequence that produces a clean, accept-able PR:

## 1. Patch + prove (TDD)

Write the failing test first, in the **owning** repo (`gas-stack-map.md`), and watch it fail for the *right* reason — a feature-missing failure, not a typo. Then the minimal fix. A test written after the fix passes immediately and proves nothing.

## 2. Dogfood on real / production-scale data — BEFORE the PR

**This is the gate that catches what unit tests can't.** A green unit test proves *behavior on a few rows*; it says nothing about *scale*. Build the fix and run it against real data:

- Does it actually do the thing at production volume? (Watch the live numbers move.)
- Does it introduce a new cost? (A "cleanup" that scans the bloat it's cleaning; a "batch" that doesn't help because the bottleneck is elsewhere.)
- Diagnose *where* time goes on the real run; don't extrapolate from the toy case.

Add a bound/scale test for whatever the dogfood exposes. Only then is the fix PR-ready.

## 2b. Audit the consumers — for any fix that DELETES or MUTATES persistent state

Dogfooding proves the *symptom* is gone at scale; it does **not** prove you didn't break something *else* that reads the data you changed. Before the PR:

- Enumerate **every reader** of the state you delete/mutate — grep the code *and the docs* for the labels / queries / columns / cursors involved.
- A record that looks like dead bookkeeping may be load-bearing: a history **ledger**, a **last-run** marker, an event **cursor**. Reclaiming it can GC the bloat *and* erase that state.
- For each consumer, preserve its contract or migrate it first. If the record serves two lifecycles, **separate them** (durable record + GC-able marker) rather than blunt-delete.
- Add regression tests for each consumer's **semantics** — not just that the rows went away.

(gc PR #2929 passed the scale gate but was CHANGES_REQUESTED here: the deleted `order-tracking` beads were also the order-run ledger, the cooldown/cron last-run state, and the `seq:*` event cursor — see `worked-examples.md`.)

## 3. Rebase onto the owning repo's main

The repo evolves. Base on current `main`, not your working checkout:

```bash
git remote add upstream git@github.com:<org>/<repo>.git
git fetch upstream main
git rebase --onto upstream/main <your-base-commit> <your-branch>   # replays only your commits onto main
```
If `main` diverged in the same function, **read main's current version** and re-apply your change onto *it* — main may have already touched the area (e.g. added a `limit` param). Don't blind-resolve; integrate. Re-run tests + `go vet` against main (`GOTOOLCHAIN=auto` if the toolchain bumped).

## 4. File a thorough PR to the RIGHT repo

**Search the repo first — by your fix / root-cause.** A matching open *or closed* PR may carry prior art, the maintainer's preferred shape, or the reason it wasn't done; link it or build on it rather than filing a duplicate. Then PR to the layer that owns the bug (`gas-stack-map.md`), squashed to one clean commit. Body template:

```markdown
## Summary
<one paragraph: the symptom, the impact, the scale (real numbers)>

## Root cause
<the exact mechanism, named function/line — what the source trace found>

## Fix
<what changed and why; note bounding/safety choices; what behavior is unchanged>

## Tests
- <failing-then-passing test of the behavior>
- <bound/scale test the dogfood motivated>
`go test ...` and `go vet` pass.

## Validation
<what running it against real data showed — the actual effect>
<for a delete/mutate fix: the consumers you audited — each reader of the changed state — and evidence each still holds>
```

## Notes

- **Don't manual-delete / hand-fix the symptom** when the goal is a fix that helps everyone — make the cleanup a *natural effect of the fixed code*, then drain via that mechanism. But **only reclaim what's provably unreferenced** — if the bloat is also read elsewhere (a ledger / cursor / last-run), the fix is to *separate the lifecycle* (durable record + GC-able marker), not to delete.
- Local-patch-only (no upstream) is reserved for genuinely work-stopping cases; otherwise upstream is the home.
- Keep the local patch + a state backup until the PR lands and the upgrade is verified.
