# Gas* stack map — which layer owns this bug?

The symptom surfaces high in the stack (a `gc` command feels slow, an agent stalls), but the bug usually lives lower. Walk *down* until the behavior is owned, then trace and PR *there*.

## The stack

| Layer | Repo | What it owns | Local source / where it lives |
|---|---|---|---|
| gc / gastown | `gastownhall/gascity`, `gastownhall/gastown` | Supervisor, controllers, sessions, hooks, reconcilers, orders, formulas, pack imports, the bd/dolt bridge (`internal/beads`) | your gascity source checkout; confirm vs the running `gc` (see below) |
| bd / beads | `gastownhall/beads` | The issue tracker, jsonl↔dolt sync, federation/credentials, the `bd` CLI | your beads source checkout; **the consuming `go.mod` pins which fork+version is real** |
| packs | `gastownhall/gascity-packs` (+ community pack repos) | Agent prompts, formulas, orders, skills, scripts | `.gc/system/packs/*`, `packs/*`, imported pack repos under `~/.gc/cache/repos/*` |
| dolt | `dolthub/dolt` | The data plane: storage (noms/chunks), commits, GC, the sql-server, `dolt` CLI | the module in the consumer's build cache; the running `dolt sql-server` |
| go-mysql-server (gms) | `dolthub/go-mysql-server` | The SQL engine embedded in dolt: query planning, index selection, joins, expression eval | a dolt dependency (check dolt's `go.mod`) |
| vitess | `dolthub/vitess` | MySQL wire protocol + SQL parser used by gms | a gms dependency |
| driver | `dolthub/driver` | The Go `database/sql` driver gc/bd use to talk to a dolt server | a gc/bd dependency |
| doltlite | `dolthub/doltlite` (`doltlite-python`) | SQLite-backed version-controlled store — an alternative to dolt; **no mysql sql-server, no port** | the consumer's build cache; an on-disk SQLite file |

## Which layer owns this symptom?

| Symptom | Likely owner(s) | Confirm by |
|---|---|---|
| `dolt`/`bd` process pinning CPU | **dolt/gms** (heavy/looping query) OR **gc/bd** (query *volume* — too many calls, or scanning a bloated store) | `SHOW PROCESSLIST` (what query); global `Questions`/`Com_select` over time (volume); `pidstat` (the real consumer); map clients with `ss` |
| Slow `bd list`/query | **gc/bd** (no filter / unbounded / bloated table / no projection) → **gms** only if EXPLAIN shows a scan despite an index | `EXPLAIN`; `SHOW INDEX`; row counts; whether bd pushes the filter to SQL vs filters in-process |
| OOM / RSS climb | **dolt** (server memory) OR **bd** (forking the dolt CLI per op — loads the whole DB) | `pgrep`/`pidstat` for forks; guard logs; which process's RSS climbs |
| Store/noms bloat on disk | **gc/bd** churn (rows written faster than reclaimed) — reclaim is **dolt** (`DOLT_GC`) | row counts by type/status; noms dir size; commit count vs `gc dolt compact`'s gate |
| Stalls / "controller stalled" | **gc** (reconciler/config-load) OR host (CPU/IO saturation) — *not* the data plane by default | load **vs** CPU% (PSI/mpstat); run-queue vs iowait; per-import `git status` cost |
| Wrong/duplicated query plans, index ignored | **gms** (planner) — a genuine upstream gms issue | `EXPLAIN` shows a full scan with an index present + stats populated |

**Rule of thumb:** a *volume* problem (too many cheap queries, a bloated table) is almost always **gc/bd**; a *per-query* problem (one query is expensive/mis-planned) points at **dolt/gms**. Prove which with `Slow_queries`, `Questions`-rate, and `EXPLAIN` before you blame the engine.

## The go.mod-is-the-source-of-truth rule

Forks and mirrors exist (e.g. `bd` appears under both `gastownhall/beads` and a personal fork; local "patched" binaries drift from any repo). **The consuming project's `go.mod` names the exact module + version that is actually linked.** Trace and PR *that* one. Before trusting any local source checkout, confirm it matches the running binary:

```bash
git -C <source> log -1 --format='%H %ci %s'   # source HEAD
<binary> version                               # running version/commit
# mismatch -> your trace may not reflect the binary; rebuild or check out the right commit
```

## dolt vs doltlite (don't assume mysql)

- **dolt**: a MySQL-protocol `sql-server` on a TCP port. Diagnose with `SHOW PROCESSLIST`, `information_schema.processlist`, global status, `ss`/`lsof` on the port. Port resolution: `--port` flag > city `dolt.port` config > `<rig>/.beads/dolt-server.port` file > legacy default.
- **doltlite**: a version-controlled **SQLite** file. **No server, no port, no PROCESSLIST.** Diagnose with SQLite tooling against the file (`.dolt`/`.doltlite` dir), file size on disk, and the consuming process's own profiling. The CPU-vs-load, binary-grep, bead-store-layout, and dogfood techniques still apply; the *server* techniques do not. Check which backend the city/store actually uses before reaching for `SHOW PROCESSLIST`.

See `gc-diagnostic-toolkit.md` for the concrete commands per backend.
