---
name: mopheus-working-on-tickets
description: "使用 mopheus ticket 命令处理工单时使用。/ Use when working on a Mopheus ticket with mopheus ticket commands — covers status side-effects, ticket comment collaboration, and metadata key conventions."
user-invocable: true
allowed-tools: Bash(mopheus *), Bash(git *), Bash(gh *)
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Working on Mopheus Tickets

Product contracts the runtime brief does not fully encode: status side-effects,
comment collaboration behavior, and metadata key conventions.

## Status changes have server-side effects

A status change is not cosmetic — the server enqueues or skips agent work based
on it.

- **`backlog`** parks an agent-assigned ticket: the assignee is set but no task
  fires. Moving `backlog → todo` (or any non-done/non-cancelled status) enqueues
  the assigned agent.
- **`todo`** on an agent-assigned ticket enqueues a run immediately (at create
  time or on promotion from backlog).
- **`in_progress`** — set this when you begin working; the runtime workflow
  instructs you to run `mopheus ticket status <id> in_progress` at step 4.
- **`in_review`** — set this when work is done and ready for review.
- **`done`** — marks completion. For a task-token-bound child ticket, entering
  `in_review` or `done` records a pending parent wake; the parent AgentTask is
  considered only after this source run reaches a final state. An effective
  parent-ticket comment trigger suppresses the duplicate wake. Human status
  updates do not auto-wake the parent in this phase.
- **`blocked`** — set when you cannot proceed; always post a comment explaining why.
- **`cancelled`** — stops outstanding work; treat as a user-driven decision.

## Sub-tickets Policy

**Do NOT create sub-tickets autonomously.** Consolidate all work, discussion, and deliverables on the current primary ticket. Collaborate with team members by posting comments with `@mentions`. Sub-tickets isolate context and increase concurrency deadlock risk.

Only create child tickets if the human user explicitly instructs you to do so:
- `--status todo` — start immediately.
- `--status backlog` — park until promoted.

## Metadata: high-signal keys only

Metadata is durable ticket state. Reading is always safe. Writing should be tied
to a concrete task requirement — most runs write zero new keys.

```bash
mopheus ticket metadata set <ticket-id> --key <stale-key> --value 225 --type number
mopheus ticket metadata delete <ticket-id> --key <stale-key>
```

## Creating tickets

```bash
mopheus ticket create --title "Fix login bug" \
  --description "Steps to reproduce..." \
  --priority high \
  --status todo \
  --agent-id <uuid> \
  --project <project-uuid> \
  --output json
```

| Flag | Required | Notes |
| --- | --- | --- |
| `--title` | yes | |
| `--description` | no | decodes `\n`; use `--description-stdin` for multiline |
| `--priority` | no | `none`, `low`, `medium`, `high`, `urgent` |
| `--status` | no | defaults to `todo`; use `backlog` to park agent work |
| `--agent-id` | no | shortcut for assigning to an agent |
| `--assignee-id` + `--assignee-type` | no | generic assignee (0=member, 1=agent, 3=team) |
| `--project` | no | project UUID |
| `--parent-id` | no | parent ticket UUID (creates a sub-ticket; only when explicitly requested by human user) |
| `--due-date` | no | RFC3339, e.g. `2026-12-31T00:00:00Z` |

Multi-line description via heredoc — all flags must come **before** `<<'EOF'`:

```bash
mopheus ticket create --title "Investigate outage" --description-stdin <<'EOF'
## Context

Customers reported 500 errors starting at 14:00 UTC.

## Steps

1. Check error logs
2. Identify root cause
EOF
```

## Applying labels

Labels classify tickets (e.g. severity, database type). Use label **names** directly — the CLI resolves the UUID automatically and creates the label if it does not exist yet.

```bash
# Add one or more labels
mopheus ticket label add <ticket-id> P1
mopheus ticket label add <ticket-id> PostgreSQL

# Remove a label
mopheus ticket label remove <ticket-id> P2

# See what labels are currently on a ticket
mopheus ticket label list <ticket-id>

# See all labels defined in this workspace
mopheus label list
```

Apply labels proactively: when you create a ticket, close it, or analyse its content and a label clearly fits — apply it without being asked. Multiple labels are encouraged.

## Posting comments

Use `mopheus ticket comment add` to post a result comment or status update.

**Heredoc rule: `--content-stdin` must be the last flag, immediately before `<<'EOF'`. Every other flag (`--parent`, etc.) must appear before `--content-stdin`. Anything written after the closing `EOF` line is a separate shell command and is silently ignored.**

```bash
# Short single-line comment
mopheus ticket comment add <ticket-id> --content "Work complete."

# ✅ Multi-line — all flags before <<'EOF'
mopheus ticket comment add <ticket-id> --content-stdin <<'EOF'
## Summary

Investigated the issue. Root cause: ...

Next steps: ...
EOF

# ✅ Reply to a specific comment — --parent comes before --content-stdin
mopheus ticket comment add <ticket-id> --parent <comment-uuid> --content-stdin <<'EOF'
Reply content here.
EOF

# ❌ Wrong — --parent after EOF is dropped silently
mopheus ticket comment add <ticket-id> --content-stdin <<'EOF'
Reply content here.
EOF
--parent <comment-uuid>   ← never reaches mopheus

# ❌ Wrong — 2>/dev/null after EOF is a no-op shell command; mopheus stderr is unaffected
mopheus ticket comment add <ticket-id> --content-stdin <<'EOF'
content
EOF
2>/dev/null   ← does nothing useful
```

Other content sources:
```bash
# Read from a file
mopheus ticket comment add <ticket-id> --content-file /path/to/result.md

# Attach local files (repeat --attachment for multiple)
mopheus ticket comment add <ticket-id> --content "See attached" --attachment /path/to/file.png --attachment /path/to/report.pdf
```

## Incorrect → correct

Autonomous sub-ticket creation:

```bash
# ❌ Incorrect — autonomously creating sub-tickets creates context silos and deadlocks
mopheus ticket create --title "Sub-task" --parent-id <id> --agent-id <uuid> --status todo

# ✅ Correct — collaborate directly in current ticket comments via @mention
mopheus ticket comment add <id> --content-stdin <<'EOF'
[@Agent Name](mention://agent/<uuid>) Please review the implementation and provide feedback.
EOF
```

## Tagging context for memory recall

While working, if you identify stable context about this ticket (e.g. `db_type`, `repo`, `error_code`, environment), record it on the ticket so future memory recall can match it:

```bash
mopheus ticket metadata set <ticket-id> --key db_type --value postgres
```

These metadata keys become "context tags" candidates when memories are recalled for the next task on the same context. Tag only genuinely useful, stable facts — not transient notes.
