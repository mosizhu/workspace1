---
name: mopheus-creating-tickets
description: "需要创建工单（ticket）时使用，覆盖所有标志、多行描述、优先级和指派方式。/ Use when creating a Mopheus ticket — covers all flags, multi-line descriptions, priority, and assignee options."
user-invocable: true
allowed-tools: Bash(mopheus *), Read, Write
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Creating Mopheus Tickets

## Basic Create

```bash
mopheus ticket create --title "Fix login bug"
```

`--title` is the only required flag. Everything else is optional.

## All Flags

| Flag | Required | Notes |
| --- | --- | --- |
| `--title` | yes | Ticket title |
| `--description` | no | Short description; decodes `\n` `\r` `\t` `\\` — use `--description-stdin` for literal backslashes or multiline |
| `--description-stdin` | no | Read description from stdin (heredoc) |
| `--description-file` | no | Read description from a UTF-8 file |
| `--priority` | no | `none` `low` `medium` `high` `urgent` |
| `--status` | no | `backlog` `todo` `in_progress` `in_review` `done` `blocked` `cancelled` — defaults to `todo` |
| `--agent-id` | no | Assign to an agent UUID (shortcut: sets `--assignee-type=1` automatically) |
| `--assignee-id` + `--assignee-type` | no | Generic assignee — 0=member, 1=agent, 3=team |
| `--project` | no | Project UUID |
| `--parent-id` | no | Parent ticket UUID — creates a sub-ticket (only when explicitly requested by human user) |
| `--due-date` | no | RFC3339, e.g. `2026-12-31T00:00:00Z` |
| `--label` (`-l`) | no | Label name or UUID (repeatable or comma-separated; `--labels` alias supported) |
| `--attachment` | no | Local file path to attach; repeat for multiple files |

## Multi-line Description (heredoc)

**Rule: `--description-stdin` must be the last flag, immediately followed by `<<'EOF'`. Every other flag — including any shell redirections — must appear before `<<'EOF'`. Anything written on a line after the closing `EOF` becomes a separate shell command and does NOT affect the `mopheus` invocation.**

```bash
# ✅ Correct
mopheus ticket create --title "Investigate outage" --description-stdin <<'EOF'
## Context

Customers reported 500 errors starting at 14:00 UTC.

## Steps

1. Check error logs
2. Identify root cause
EOF

# ❌ Wrong — flags after EOF are silently dropped
mopheus ticket create --title "..." --description-stdin <<'EOF'
content
EOF
--priority high   ← never reaches mopheus

# ❌ Wrong — 2>/dev/null after EOF runs as a no-op shell command; mopheus is unaffected
mopheus ticket create --title "..." --description-stdin <<'EOF'
content
EOF
2>/dev/null   ← does nothing useful
```

## Assign to an Agent

```bash
# Shortcut — agent-id implies assignee-type=1
mopheus ticket create --title "..." --agent-id <agent-uuid>

# Generic form
mopheus ticket create --title "..." --assignee-id <agent-uuid> --assignee-type 1
```

## Assign to a Team

```bash
mopheus ticket create --title "..." --assignee-id <team-uuid> --assignee-type 3
```

## Sub-tickets Policy

**Do NOT create sub-tickets autonomously.** Consolidate all work, discussion, and deliverables on the current primary ticket. Sub-tickets create information silos (parent context and attachments are isolated) and risk recursive task deadlocks.

Use `--parent-id` ONLY if a human user explicitly requests creating a child or sub-ticket:

```bash
mopheus ticket create --title "Child Task" --parent-id <parent-uuid> --status todo
```

## Status and Agent Enqueueing

- **`todo`** — agent task fires immediately at create time.
- **`backlog`** — assignee is recorded but no task fires; promote to `todo` when ready.
- All other statuses at create time follow the same rule as `todo` (the agent is enqueued) except `backlog` and `cancelled`.

## Mention Links in Descriptions

You can embed `[@Label](mention://<type>/<id>)` mention links inside any
description field (`--description`, `--description-stdin`, `--description-file`).
The backend parses them the same way it does in comments — agent/team mentions
enqueue runs, member/ticket mentions render as links.

Use this to provide rich context at creation time:

```bash
mopheus ticket create --title "Fix payment flow" --agent-id <agent-uuid> \
  --description-stdin <<'EOF'
## Context

Users report failures at checkout. [@Support Bot](mention://member/<user-uuid>)
flagged this in #12.

## Assigned

[@Backend Bot](mention://agent/<agent-uuid>) — please investigate.
EOF
```

For mention link format, type contracts, and lookup commands, see the
`mopheus-mentioning` skill.

## Full Example

```bash
mopheus ticket create \
  --title "Refactor auth middleware" \
  --priority high \
  --status backlog \
  --agent-id <uuid> \
  --project <project-uuid> \
  --due-date 2026-12-31T00:00:00Z \
  --description-stdin <<'EOF'
## Goal

Replace the legacy session middleware with JWT validation.

## Acceptance Criteria

- All existing tests pass
- New unit tests cover edge cases
EOF
```
