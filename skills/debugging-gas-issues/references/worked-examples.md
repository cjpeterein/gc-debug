# Worked examples

Two real traces. Each shows the **layer hop** (symptom layer ≠ owner layer) and what the dogfood gate caught. Read them for the *shape* of the method, not the specifics.

## 1. The fork-storm OOM — symptom: dolt; owner: bd

| Step | What happened |
|---|---|
| **Symptom** | The whole box OOMs; the `dolt sql-server` is killed repeatedly; system meltdown under normal pipeline load. |
| **Naive read** | "Dolt has a memory leak" / "it's the jsonl auto-import." Both wrong. |
| **Trace** | strace/source the *live* write path (parent + CWD), not a synthetic one. `bd` forks `dolt remote -v` — a process that loads the **whole DB (~2 GB)** — on **every write**, via multi-store federation/credentials peer-resolution. It's called with non-dolt CWDs, so it errors *after* forking 2 GB; errors were never cached, so it re-forked ~25×/min × 2 GB → OOM. |
| **Owner** | **bd** (`gastownhall/beads`), not dolt. `dolt.auto-push`/`export.auto` were irrelevant. |
| **Fix + upstream** | Skip/cache the fork when the path isn't a dolt repo; negative-cache errors. Upstream: **beads#3948**. |
| **Transferable lesson** | Verify system-wide, not on a narrow best-case path. "Fixed" off a synthetic measurement was declared twice and was wrong both times. The OOM *surfaced* in dolt; it was *owned* by bd. |

## 2. The order-tracking-bead leak — symptom: dolt CPU; owner: gc

| Step | What happened |
|---|---|
| **Symptom** | The `dolt sql-server` pins ~2.4 cores; routine `bd list` reads are slow. |
| **Naive read** | "Load average is 22, CPU is maxed." **Wrong** — load ≠ CPU; PSI/mpstat showed the guest was a 6-vCPU cap on a 20-thread host, not host-saturated. (Corrected on the spot.) |
| **Trace** | `SHOW PROCESSLIST` → ~1k QPS of full-row reads on `hq.issues`. Row counts → 39.5k rows, **~95% closed `order:*` tracking beads**. Source-trace `order_dispatch.go` → the sweep **closes** tracking beads but **never deletes** them; every order leaves one behind forever. `Slow_queries=0`, indexes present → a *volume*/bloat problem, not a slow query. |
| **Owner** | **gc** (`gastownhall/gascity`, the sweep lifecycle) — *not* dolt/gms. The identical symptom *could* have been a gms plan miss; `EXPLAIN` + `Slow_queries` ruled that out. |
| **Fix + upstream** | The sweep deletes stale closed tracking beads (bounded, issues-tier, batched). Upstream: **gc PR #2929**. |
| **What the dogfood gate caught** (all green unit tests, all wrong at scale) | (a) **unbounded list** — v1 listed all 37.7k each run, *recreating* the full-scan it was fixing; (b) **aggregation miss** — the across-stores sum dropped `trackingDeleted`, misreporting "deleted 0" and stalling the drain; (c) **wrong tier** — `TierBoth`+newest-first kept deleting *regenerating ephemeral* wisps and never reached the issues-tier backlog. |
| **Transferable lesson** | Dogfooding on real data before the PR caught three bugs unit tests structurally couldn't. Measure the bottleneck before optimizing (the small-N delete timing already showed batching wouldn't help). The dolt-CPU symptom was *owned* by gc. |
