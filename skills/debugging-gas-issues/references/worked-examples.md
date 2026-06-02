# Worked examples

Two real traces. Each shows the **layer hop** (symptom layer ≠ owner layer); #2 also shows the two gates *after* the trace — the **dogfood (scale)** gate and the **consumer/semantics** gate. Read them for the *shape* of the method, not the specifics.

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
| **Fix attempted** | The sweep **deletes** stale closed tracking beads (bounded, issues-tier, batched). Upstream: **gc PR #2929 — CHANGES_REQUESTED.** |
| **What the dogfood gate caught** (all green unit tests, all wrong at scale) | (a) **unbounded list** — v1 listed all 37.7k each run, *recreating* the full-scan it was fixing; (b) **aggregation miss** — the across-stores sum dropped `trackingDeleted`, misreporting "deleted 0" and stalling the drain; (c) **wrong tier** — `TierBoth`+newest-first kept deleting *regenerating ephemeral* wisps and never reached the issues-tier backlog. |
| **What the maintainer review caught** (the dogfood gate structurally could not) | The closed `order-tracking` bead is **overloaded**: it's a single-flight marker **and** the durable **order-run ledger**. `gc order history`, the API `/orders/history` (+ `gate_stdout/stderr` detail), cooldown/cron **last-run** state, and the `seq:*` **event cursor** all read it. Deleting it GCs the bloat *and erases that state* — exec orders look `never run`, cursors are lost. Tests covered deletion **mechanics**, not the **semantics** the data serves. |
| **Clean shape** | Don't delete a record that serves two lifecycles — **separate them**: a durable order-run ledger + GC-able single-flight markers, migrate the history / last-run / cursor reads to the ledger, *then* GC the markers. (The maintainer's requested reshape.) |
| **Transferable lessons** | (1) Dogfooding on real data caught three scale bugs unit tests structurally couldn't. (2) **Scale-proof ≠ correct:** before a fix DELETES or MUTATES persistent state, enumerate **every consumer** — "dead bloat" another subsystem queries is a **ledger**, not garbage — and test the *semantics* it serves, not just the change's mechanics. The dolt-CPU symptom was *owned* by gc; a clean *fix* also preserves the data's other contracts. |
