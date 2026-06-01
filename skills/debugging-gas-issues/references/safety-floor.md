# Safety floor

Debugging a live Gas* system means most of the danger is in the *fix*, not the diagnosis. Diagnose freely; mutate carefully.

## Capture state first

Before any change, snapshot what you'll need to compare against or roll back to:

```bash
gc config show > /tmp/gc-snapshot-$(date +%s).toml
# + the live numbers you're about to change: row counts, dolt-noms size, CPU, the metric in question
```
Keep a copy of any binary you swap and any data you're about to delete until the fix is verified.

## Free: read-only / dry-run

Any inspection or `--probe`/`--dry-run`/`--check` mode is always fine: `gc config explain`, `gc doctor`, `gc events`, `gc trace show`, `gc bd query`, `gc dolt-cleanup --probe --json`, `SHOW PROCESSLIST`, `EXPLAIN`, `pidstat`, etc.

## Prefer the gc-native tool over the raw op (diagnostic-first)

There is almost always a gc-native path. Reach for it before the blunt instrument:

| Don't | Do |
|---|---|
| `pkill -f tmux` / `kill -9` a session | `gc session kill <id>` / `gc stop --force` |
| raw SQL `DELETE` against the store | the bd/gc cleanup mechanism (and make deletion a natural effect of the fix) |
| `rm -rf .beads/dolt/` | `gc dolt recover`, or quarantine the corrupt dir to a sibling path |
| `systemctl restart` first | `gc supervisor reload` → `gc reload --soft` → restart only if those don't take |

## Don't, without explicit human/owner go-ahead (in the same thread)

- `gc supervisor stop` / `uninstall`, `gc stop --force`, `gc cities unregister`
- `gc dolt-cleanup --force` (the probe is fine), `gc dolt recover` unless already wedged and it's the documented next step
- `bd delete` / `bd close` on beads you don't own; any `rm -rf` against `.beads`/`.gc`
- `kill -9` a process not under your immediate control
- editing machine-wide config (`~/.gc/supervisor.toml`)
- `git push` from a rig; swapping a live binary into the running supervisor

## The cost asymmetry

Asking costs one round-trip. A wrong destructive op costs hours. When in doubt: **capture state, mail the question with the diagnostic data, wait for the green light.**
