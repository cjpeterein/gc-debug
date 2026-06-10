# gc/bd/dolt diagnostic toolkit

Concrete commands. Read-only / dry-run unless noted. See `safety-floor.md` before anything destructive.

## Confirm the binary matches the source

```bash
git -C <source> log -1 --format='%H %ci %s'    # source HEAD
<binary> version                                # running build
```
If they diverge, your source trace may not reflect the running binary. Rebuild or check out the matching commit first.

## Binary symbol-grep (when config/schema docs lie)

The compiled binary carries its struct tags and Go type names. `grep -ao` works even where `strings` is filtered.

```bash
grep -aoE 'toml:"[^"]+"' "$(which gc)" | sort -u | grep -iE 'patch|rig|agent'   # TOML tags
grep -aoE 'json:"[^"]+"' "$(which gc)" | sort -u                                 # JSON/API tags
grep -aoE '[A-Z][a-zA-Z]+Patch(es)?[a-zA-Z]*' "$(which gc)" | sort -u            # Go type names
grep -aoE '/v[0-9]+/city/[^ ]*' "$(which gc)" | sort -u                          # API path templates
```

## CPU vs load (they are not the same)

Load average counts runnable **plus** uninterruptible-sleep (I/O-wait) tasks. A box can show load 22 at 26% CPU. Measure utilization, not load:

```bash
cat /proc/pressure/cpu /proc/pressure/io      # PSI: cpu `some` high + io `some` low = CPU-bound
mpstat 1 2                                     # %usr+%sys vs %idle, %iowait, %steal
vmstat 1 3                                     # r (run-queue) vs b (blocked); wa, st
nproc; lscpu | grep -E 'On-line|Off-line|Model name'   # vCPU cap vs host (a 6-vCPU guest on a 20-thread host = ~30% of host when pinned)
```
`%steal` high → the hypervisor is throttling. `%iowait` high → I/O-bound, not CPU. Run-queue ≫ cores at low iowait → genuinely CPU-bound.

## Find the real consumer

```bash
pidstat 3 1 | awk '/^Average:/ && $3 ~ /^[0-9]+$/ {cpu[$NF]+=$8} END{for(c in cpu) if(cpu[c]>=2) printf "%7.1f%%  %s\n", cpu[c], c}' | sort -rn
```
Aggregates sustained CPU by command. (`ps` %CPU is lifetime-average — misleading for "now".)

### Userspace vs syscall attribution (is dolt CPU in SQL execution or connection/IO?)

Per-thread `utime`/`stime` answer whether the CPU is burned **executing SQL** (userspace) or **handling connections/IO** (syscall). Sum field 14 (utime) vs 15 (stime) across the process's threads:

```bash
PID=<dolt-pid>
awk '{u+=$14; s+=$15} END{printf "utime=%d stime=%d  utime%%=%.0f\n", u, s, 100*u/(u+s)}' \
  /proc/$PID/task/*/stat
```
(Live: ~92% utime → execution-bound, which **ruled out** connection-churn as the sink.) Complements host-level PSI/mpstat and per-process `pidstat` with per-thread granularity.

### Measure what a connection pool would actually save

Before hypothesizing "add pooling," measure it: compare dolt CPU jiffies for N queries via **fresh-conn-per-query** vs a **single persistent conn** — read `/proc/<dolt-pid>/stat` fields 14+15 (utime+stime) before and after each run:

```bash
J(){ awk '{print $14+$15}' /proc/$PID/stat; }
b=$(J); for i in $(seq 1 200); do DSQL "<query>"; done;        echo "fresh-conn:  $(( $(J) - b )) jiffies"   # new conn each call
b=$(J); DOLT_CLI_PASSWORD='' dolt --host 127.0.0.1 --port "$PORT" --user root --no-tls sql <<<"$(yes '<query>;' | head -200)"; echo "persistent: $(( $(J) - b )) jiffies"
```
(Live: 19.5 s vs 16.55 s for 200 queries → pooling saves ~15%; query *execution* is the other 85%.) Note `bd` **already** has a `database/sql` pool — defeated by **process-per-CLI-invocation**, so "add a pool" actually means "make `bd` a daemon." Stops an ~11–15% lever from being mistaken for the fix.

## dolt server introspection

```bash
PORT=$(cat <rig>/.beads/dolt-server.port 2>/dev/null || echo 3307)   # resolution: --port > city dolt.port > port-file > legacy
DSQL(){ DOLT_CLI_PASSWORD='' dolt --host 127.0.0.1 --port "$PORT" --user root --no-tls sql -q "$1"; }

DSQL "SELECT id,db,command,time,LEFT(info,150) FROM information_schema.processlist WHERE command<>'Sleep' ORDER BY time DESC"  # what's running NOW
DSQL "SHOW GLOBAL STATUS WHERE Variable_name IN ('Questions','Com_select','Slow_queries','Threads_connected','Uptime')"        # volume vs slow
DSQL "USE \`<db>\`; EXPLAIN <query>"; DSQL "USE \`<db>\`; SHOW INDEX FROM <table>"                                              # plan + indexes
```
Sample `Questions` twice to get QPS. **`Slow_queries`=0 + huge `Com_select` = a volume problem (gc/bd), not a slow-query problem (gms).**

### Rate vs duration — "always caught running" is not a frequency

A query that *dominates* `PROCESSLIST` samples is measuring **duration-share**, not frequency: one slow query holds a sample even at low call-rate. Sampling `Questions` gives **all-query** QPS, not per-shape. For the true rate of a *specific* query shape, count **distinct connection-ids** running it over a window:

```bash
for i in $(seq 1 100); do
  DSQL "SELECT id FROM information_schema.processlist WHERE info LIKE '%<query-sig>%'"
done | sort -u | wc -l    # distinct conns over the window / window-seconds = per-shape rate
```
(Live: 1102 distinct conns in 10 s → ~110/s — versus the ~856/s a naive duration-share read implied.)

Map which processes hold connections (find the culprit client):
```bash
ss -tnp 2>/dev/null | awk -v p=":$PORT$" '$5 ~ p' | grep -oE '"[^"]+",pid=[0-9]+' | sort | uniq -c | sort -rn
```

## Bead store: layout, tiers, bloat

Beads live in two tiers: **issues** (durable, non-ephemeral) and **wisps** (ephemeral, TTL-reclaimed by wisp-compact). Bloat is usually closed rows never deleted.

```bash
DSQL "USE \`<db>\`; SELECT issue_type, COUNT(*) n, SUM(status='closed') closed FROM issues GROUP BY issue_type ORDER BY n DESC"
DSQL "USE \`<db>\`; SELECT COUNT(*) FROM issues WHERE status='closed' AND title LIKE 'order:%'"   # leaked order-tracking
gc dolt-cleanup --probe --json     # orphan dbs / stale procs (NEVER --force without go-ahead)
```
Disk reclaim after deleting rows is a **dolt** GC: `CALL DOLT_GC('--full')` (online-safe; quiesce writers first). `gc dolt compact` gates on commit count and skips low-commit/high-churn dbs.

## doltlite (SQLite backend)

No server/port/PROCESSLIST. Inspect the file:
```bash
du -sh <store>/.doltlite 2>/dev/null
sqlite3 <store-file> 'SELECT issue_type, COUNT(*) FROM issues GROUP BY issue_type;'   # adapt to the actual schema/path
```
The CPU-vs-load, binary-grep, store-layout, and dogfood techniques still apply; the server techniques do not. **Check the backend before reaching for `SHOW PROCESSLIST`.**
