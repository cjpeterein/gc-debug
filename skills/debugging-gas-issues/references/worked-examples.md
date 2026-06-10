# Worked examples

Four real traces. Each shows the **layer hop** (symptom layer ≠ owner layer); #2 also shows the two gates *after* the trace — the **dogfood (scale)** gate and the **consumer/semantics** gate; #3 shows how a **chain of refuted theories** ends the moment you read the component's own logs; #4 shows a **volume problem that is an amplifier, not a bloated table** — a wake loop broader than the work it checks for. Read them for the *shape* of the method, not the specifics.

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

## 3. The pipeline park — symptom: agents won't wake; owner: gc (wake-budget starvation)

| Step | What happened |
|---|---|
| **Symptom** | Idle agents (witness, outrider, architect) don't restart to pick up routed work; the *same* agents "park" until manually revived; a P1 sat unpicked. |
| **Refuted theory chain** (the trap) | Three plausible RCAs, each disproved before the next, all reasoned from the *symptom*: (a) **paste→Enter debounce race** — refuted: a bare Enter didn't submit, *and* a fresh inject with the same 500 ms debounce did; (b) **stale busy-detector** (`WaitForIdle` greps for "esc to interrupt"; current Claude shows a spinner instead) — a real bug, but a *queued* nudge auto-resolves, so it's latency, not a park; (c) **multiline bracketed-paste won't submit** — refuted: multiline + one Enter submits fine. |
| **What ended it** | Grep the owning component's own log: `session lifecycle: op=start session=…architect outcome=deferred_by_wake_budget`. One line named the root — after hours of theorizing. |
| **Trace** | `executePlannedStarts` spends `daemon.max_wakes_per_tick` over a **stable dependency topo-order** (`topoOrder`), front-to-back, with **no fairness**. When wake-demand > budget the *same* back-of-order sessions defer every tick. Because it's a *dependency* sort, devpipeline agents (which depend on infra/pool) sort **last** → a routine pool worker wakes ahead of a P1-holder. Tightening the budget for CPU made it strictly worse. |
| **Owner** | **gc** (`gastownhall/gascity`, reconciler scheduling). The CPU saturation only *amplified* it — measured 93% used / 0% steal, and the "starved" agent was asleep in `ep_poll` at 2% CPU (idle, **not** CPU-starved). |
| **Fix + upstream** | Sort each dependency wave's ready candidates **least-recently-woken first** (`last_woke_at` → `CreatedAt`) before spending the budget; winning a wake updates `last_woke_at` → rotation → no permanent starvation. Red→green test; dogfooded that `last_woke_at` is populated/updated on the real fleet. Upstream: **gc PR #3059**. |
| **Transferable lessons** | (1) **A chain of refuted hypotheses means you're theorizing, not measuring** — read the owning component's decision-logs *early*; the root was one `deferred_by_wake_budget` line. (2) Read **process state** (`ep_poll`/`S`/2% CPU = idle, not starved) — don't infer "CPU-starved" from load. (3) When a **budget/limit defers work, the selection method is the bug surface** — a *stable* order starves the same victims; fairness (least-recently-woken / round-robin) fixes it. |

## 4. The wake-amplifier — symptom: dolt CPU; owner: gc (event relevance too broad)

| Step | What happened |
|---|---|
| **Symptom** | The `dolt sql-server` pinned ~114% CPU; system load 8.7 but **56% CPU idle** — vCPU contention, not saturation. |
| **Naive read** | "It's the bloated `hq` store again — shrink the history." (Nearly identical to #2; the pull toward *re-running the prior fix* was the trap. The bloat was real but it was the *multiplicand*, not the bug.) |
| **The contradiction that broke it** | Duration-share read said ~856 assembly-queries/s; 45 ms × 856/s ≈ 38 core-seconds/s is impossible on ~1 core. Re-measured by **distinct connection-ids** (toolkit: rate-vs-duration) → ~110/s. The arithmetic contradiction was the thread: queries were *queuing*, and the question became "what fires ~110 queries/s," not "why is each query slow." |
| **Trace** | Per-agent serve loop (`runWorkflowServeFollow`) re-runs its work query on every wake. `workflowEventRelevant()` (`cmd/gc/dispatch_runtime.go:630`) keys on `evt.Type` **only** (`BeadCreated/Updated/Closed`), ignoring `evt.Subject`/assignee/rig/`gc.routed_to`. So **any** bead event anywhere wakes **every** agent's serve loop and resets `idleSweeps=0` (line 566), defeating the 1s→30s exponential backoff (`followSleepDuration`). Order churn (~27 orders firing every 1–2 min; each order-run tracking-wisp lifecycle = create→update→update→close ≈ 4 events) → ~2.9 events/s on `.gc/events.jsonl` → all ~6 serve loops re-query at the 1s floor → ~110 assembly-query exec/s against the bloated `hq` (12k wisps / 45k labels), 39–96 ms each → ~5+ core-seconds/s demand vs ~1 core → dolt pinned. |
| **Owner** | **gc** (`gastownhall/gascity`, `cmd/gc/dispatch_runtime.go`) — **not** dolt/gms; the query plan is fine. |
| **Fix shape** | Scope event relevance to events that could match **this** agent's work query (subject/assignee/rig/`gc.routed_to`); the backoff then engages and the query rate collapses ~10×. **Non-destructive** — does *not* touch the order-run ledger (the #2 / gc#2929 landmine). Filed as gascity bead **gc-6av**. |
| **Transferable lesson** | A volume problem is **not always a bloated table** — it can be an **amplifier**: an event/poll loop whose wake condition is *broader* than the work it actually checks for. Measure the event-bus rate and correlate it to the query rate; a globally-scoped wake condition multiplies a cheap per-item cost by N loops. (And: resist re-running the previous fix just because the symptom matches — #2's bloat was present here too but was the multiplicand, not the bug.) |
