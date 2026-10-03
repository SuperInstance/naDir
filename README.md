# naDir

**A running job is just a directory. When it's done, the directory moves.**

naDir ("native directory") is a way to track async work — builds, agent tasks, background jobs, anything that starts now and finishes later — using nothing but directories. No server. No database. No dashboard to install.

## The problem

You start a job. It disappears into the void — a background process, an agent's context, a queue somewhere. To know what happened, you poll, you grep logs, you check a dashboard, or you forget about it entirely. Dispatched work is invisible. It has no place.

## The idea

Every unit of work gets a directory that *records* it. When you start something, you open an entry:

```
ledger/open/build-47/
  intent.md     ← what was asked, by whom, when
  output.log    ← growing log, written as it runs
  meta.json     ← started_at, heartbeat_at, owner (required)
```

When it finishes, the directory moves:

```
ledger/closed/build-47/
  intent.md     ← carried over
  output.log    ← final log
  meta.json     ← carried over
  result.md     ← success | failed, exit code, one-line summary
```

That's the whole system. Two folders: `open/` and `closed/`. Starting work creates a directory. Finishing work moves it. If you started ten things and eight are closed, the two still in `open/` are your in-flight work — visible at a glance.

To be precise about the tagline: the directory is the job's *record*, not the job itself. Copying the directory doesn't teleport a running process. What it does — and this is the part that matters — is package the *entire context* of the work (intent, partial output, metadata) into something you can hand to another machine, another agent, another person. The handoff is the address.

## Keeping the books honest

The ledger only works if `open/` means "actually running." Three conventions keep it honest:

1. **Heartbeat.** Every open entry has `meta.json` with `heartbeat_at`, updated periodically by whatever is doing the work. An entry whose heartbeat is older than your threshold (an hour? a day? — your call) is *stale*, not running.
2. **Reaper.** Something — a cron job, a wrapper script, you — periodically scans `open/` for stale entries and moves them to `ledger/abandoned/` with a note. The books balance because *something* closes every entry, even the dead ones.
3. **Unique names.** Two writers picking `build-47` will collide. Use timestamps, UUIDs, or namespaced names (`agent1-build-47`). Check before creating.

Without these, `open/` accumulates corpses and the "visible at a glance" promise rots. The format is simple; the discipline is load-bearing.

## Why directories, not a database?

The obvious question. Four reasons:

- **Zero install.** `mkdir -p ledger/open ledger/closed` and you're running. No service to configure, no schema to migrate.
- **Unix-native.** `ls`, `cat`, `grep`, `find` — every tool you already have works on the ledger. No query language to learn.
- **Git-versionable.** Commit the ledger (minus the logs — see below) and your async history is time-travelable.
- **Human-readable.** You can debug the system by looking at it. No opaque binary format between you and the truth.

A database would give you atomicity and queries. naDir trades those for simplicity and transparency. If you need transactions, you already have a database — naDir isn't replacing it.

## Sharp edges

Honest limitations, so you can work around them:

- **Not atomic by default.** Write `result.md` to a temp file, then atomically rename it into place, *then* move the directory. Crash between steps and you'll have an entry that's half-closed. The filesystem gives you the tools; the discipline is yours.
- **Not concurrent-safe.** One writer per ledger, or unique names plus atomic creates. Two processes writing to the same entry will step on each other.
- **Don't version the logs.** `output.log` will grow. Git-commit the ledger structure (`intent.md`, `meta.json`, `result.md`), `.gitignore` the logs, or your repo becomes unpushable.
- **naDir doesn't run anything.** Your existing tools do the work; naDir holds the record. The writers that automate open/close don't exist yet — for now, it's a discipline, and disciplines need the heartbeat/reaper conventions above to survive contact with reality.

## Why this matters

Because the record is just files, file operations become job operations:

- **See what's running** — `ls ledger/open/`
- **Check on a job** — `cat ledger/open/build-47/output.log`
- **Hand a job's context to another machine** — `cp -r ledger/open/build-47 /other/machine/ledger/open/` and something there picks it up
- **Snapshot everything in flight** — `tar -czf snapshot.tgz ledger/open/`
- **Find failures** — sort `ledger/closed/` by `result.md`. Failures have *places*.
- **Audit** — `git log` on the ledger is your async history

## The format

An open entry (all fields required unless noted):

```
ledger/open/<unique-name>/
  intent.md    — what was asked, by whom, when
  meta.json    — {"started_at": "...", "heartbeat_at": "...", "owner": "..."}
  output.log   — running output (optional, appended as it runs)
```

A closed entry adds:

```
  result.md    — success | failed, exit code, one-line summary
```

Abandoned entries (moved by the reaper):

```
ledger/abandoned/<unique-name>/
  intent.md, meta.json, output.log (if any)
  abandoned.md — when it was reaped, last heartbeat age
```

## What this isn't

- Not a job runner. naDir doesn't execute anything.
- Not a dashboard. The directory *is* the interface. (Projections read the ledger; they don't replace it.)
- Not a queue. No scheduling, no priorities, no workers.

## What's coming

- **Writers** — shell wrappers and hooks so tools open/close entries automatically instead of by hand.
- **Projectors** — spreadsheet and web views rendering the ledger as a live grid. Sort by status, filter by staleness, zoom into any entry.
- **Handoff** — an agent reads an open entry on one machine and resumes the work on another. The directory is the context transfer.

## Start

```bash
mkdir -p ledger/open ledger/closed ledger/abandoned
echo "*.log" > ledger/.gitignore
```

That's the install. You're running naDir.
