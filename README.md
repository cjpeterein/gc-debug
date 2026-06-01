# gc-debug

A Gas City / Gas Town debugging skill pack. Trace a complex Gas\* issue across the dependency stack to its real root, prove the fix on real data, and land it as a mergeable upstream PR.

## The idea

Most "gc problems" are actually **bd, dolt, go-mysql-server, vitess, doltlite, driver, or pack** problems. **The layer that shows the symptom is rarely the layer that owns the bug.** This pack ships one skill — `debugging-gas-issues` — that walks you *down* the stack to the owning layer, makes you prove the fix at production scale (a green unit test proves *behavior*, not *scale*), and finishes at a clean PR to the *right* repo.

It's the methodology used to trace two real performance bugs — a bd→dolt fork-storm OOM and a gc order-tracking-bead leak that pinned the Dolt sql-server at ~2.4 cores — to root and PR them. It builds on the `systematic-debugging` and `test-driven-development` disciplines, applied to the Gas\* stack with concrete tooling.

## Install

```
gc import add github.com/Wldc4rd/gc-debug
```

Loads as the `debugging-gas-issues` skill in your city's agents.

## When to use

High CPU, slow `bd`/queries, OOM, store/noms bloat, stalls, full-table scans, or intermittent failures anywhere in gc · bd · dolt · gms · doltlite. **Before** theorizing, working around, or escalating.

## What's inside

| File | What it covers |
|---|---|
| `SKILL.md` | The arc (capture state → evidence → **find the owning layer** → source-trace → RCA → diagnose-where-time-goes → TDD → **dogfood on real data** → upstream PR), the iron laws, and a red-flags table where every entry is a real mistake from a live trace. |
| `references/gas-stack-map.md` | The dependency map, a "which layer owns this symptom" table, the upstream repo per layer, the go.mod-is-source-of-truth rule, and the dolt-vs-doltlite split. |
| `references/gc-diagnostic-toolkit.md` | Binary symbol-grep, dolt `SHOW PROCESSLIST` + connection mapping, CPU-vs-load (PSI/mpstat), bead-store/tier layout, doltlite inspection. |
| `references/upstreaming-fixes.md` | patch → prove → dogfood → rebase → PR, with a PR-body template. |
| `references/worked-examples.md` | The two traces, end to end, each showing the layer hop. |
| `references/safety-floor.md` | Capture-state-first; the don't-without-go-ahead list. |

## Status

**v0.1** — early, and meant to improve with use. Feedback and PRs welcome — especially more worked traces, doltlite-specific diagnostics, and additional "which layer owns this" heuristics. The point is to get better at tracing Gas\* issues to mergeable PRs, together.
