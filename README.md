# naDir

**Every action an agent takes leaves a traceable record — what it did, where it came from, where it went.**

naDir ("native directory") is infrastructure for multi-agent systems. When many agents work across many projects — each in their own area of expertise, each in their own conversation thread — naDir gives every unit of work a *place* with an *entry point* and an *exit point*. Double-entry bookkeeping, but the books record provenance: which agent, which source, which destination.

No server. No database. Just directories.

## The problem

You have agents working in parallel. Agent A researches, Agent B writes code, Agent C deploys. Work flows between them — but where did that data come from? Which agent touched it? What did the project look like before and after?

Right now, that provenance evaporates. It lives in conversation threads, in terminal scrollback, in nobody's memory. When something breaks, you can't follow the thread. When you want to understand what happened, you reconstruct it by hand.

## The idea

Every unit of agent work opens a ledger entry that records **provenance** — not just what happened, but where it came from and where it's going:

```
ledger/open/20261003-1423-research/
  intent.md     ← what the agent was asked to do
  meta.json     ← agent id, started_at, heartbeat_at, source, project
  inputs/       ← references to what was consumed (prior entries, files, data)
  output.log    ← running output
```

When the work completes, the entry closes with its destination:

```
ledger/closed/20261003-1423-research/
  intent.md
  meta.json
  output.log
  outputs/      ← references to what was produced
  result.md     ← success | failed, summary
```

The critical fields in `meta.json`:

```json
{
  "agent": "researcher-3",
  "project": "syzygy",
  "started_at": "2026-10-03T14:23:00Z",
  "heartbeat_at": "2026-10-03T14:45:00Z",
  "source": "ledger/closed/20261003-1401-brief",
  "source_agent": "coordinator-1"
}
```

Every entry points backward to where its inputs came from and forward to where its outputs go. The ledger isn't a list — it's a **provenance graph** written as directories.

## What this enables: follow anything

Because every entry records its source and destination, pipelines *emerge* from the ledger. You don't declare them upfront; you discover them by tracing.

**Follow the data** (the fuel's journey):
A dataset enters as Agent A's input, becomes a report in Agent B's hands, becomes a commit via Agent C, becomes a deploy. Trace `outputs → inputs` chains and you see the exact path that specific data took through your system — like following a gallon of fuel through the engine to exhaust.

**Follow the agent** (who did what):
Filter by `agent`. Every action `researcher-3` took, across all projects, in order. Useful for review, for debugging ("which agent introduced this?"), for understanding specialization patterns.

**Follow the project** (the boat's journey):
Filter by `project` and `time`. The same events as the fuel's journey, but now you're watching the *system's* state change over that period — not the data's. Same ledger, different abstraction. The boat moving, not the fuel burning.

**Follow the time** (cross-section):
What was every agent doing at 14:30? Which entries were open? The ledger is a snapshot of your entire fleet at any moment — queryable with `ls` and `grep` today, with spreadsheet projections tomorrow.

## The agent contract

If you're building an agent that writes to naDir, here's the entire API — shell commands, no library:

```bash
# Open: when the agent starts work
NAME="$(date +%Y%m%d-%H%M)-<task-slug>"
mkdir -p ledger/open/$NAME/inputs
cat > ledger/open/$NAME/intent.md <<EOF
# $TASK_DESCRIPTION
EOF
cat > ledger/open/$NAME/meta.json <<EOF
{
  "agent": "$AGENT_ID",
  "project": "$PROJECT",
  "started_at": "$(date -u +%FT%TZ)",
  "heartbeat_at": "$(date -u +%FT%TZ)",
  "source": "$SOURCE_ENTRY",
  "source_agent": "$SOURCE_AGENT_ID"
}
EOF

# During: heartbeat and output
echo "$(date -u +%FT%TZ)" > /tmp/hb  # update meta.json heartbeat periodically
echo "working..." >> ledger/open/$NAME/output.log

# Close: when the agent finishes
cat > ledger/open/$NAME/result.md <<EOF
success — <one-line summary>
EOF
mkdir -p ledger/open/$NAME/outputs
# ... record output references in outputs/ ...
mv ledger/open/$NAME ledger/closed/$NAME
```

Three moments: open, heartbeat, close. That's the contract. Any agent that can run shell commands can implement it in minutes.

## Keeping the books honest

- **Heartbeat.** `meta.json` → `heartbeat_at`, updated while work is in flight. Stale = heartbeat older than your threshold.
- **Reaper.** Something scans `open/` for stale entries, moves them to `ledger/abandoned/` with a note. Every entry gets closed, even the dead ones.
- **Unique names.** Timestamp + slug + agent id. Check before creating.

## Why directories, not a database?

- **Zero install.** `mkdir` and you're running. Any agent, any machine.
- **Portable.** `cp -r` an entry to another machine and the *entire context* — intent, inputs, partial output, provenance — goes with it. Handoff is addressing.
- **Git-versionable.** Commit the ledger (not the logs) and you have time-travel over everything your agents did.
- **Human-readable.** The human operator debugs the fleet by looking at directories.

## What this isn't

- Not a runner, scheduler, or queue. Your agents already do the work.
- Not a dashboard. Projections (spreadsheet views, web UIs) read the ledger; the ledger is the source of truth.
- Not a replacement for any single system's native tracking. It's the *cross-system* layer — the one place where work from all your agents, all your projects, is visible together.

## Start

```bash
mkdir -p ledger/open ledger/closed ledger/abandoned
echo "*.log" > ledger/.gitignore
```

Point your first agent at it. Watch the provenance graph grow.
