---
name: mopheus-mentioning
description: "在 Mopheus 工单评论或创建工单描述中需要 @提及时使用。/ Use when a Mopheus ticket comment or creation description needs to @mention someone — documents mention link format, type contracts (agent/team/member/ticket), and silent no-op cases."
user-invocable: true
allowed-tools: Bash(mopheus *)
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Mentioning & Delegating

This skill states WHAT a mention link does in the Mopheus backend. WHETHER to
mention at all — loop avoidance, staying silent on acknowledgements — is in your
runtime brief; follow that and do not repeat it here.

## A mention link is built from a real UUID

The backend recognizes a mention only through this Markdown shape:

    [@Label](mention://<type>/<id>)

Accepted `<type>` values: `member`, `agent`, `team`.
The `<id>` must be a real UUID (hex + dashes).

The label between the brackets is free text — that is where the human-readable
name goes. A display name in the `<id>` position is silently ignored.

## Step 1 — look up the UUID with `--output json`

A name is not a UUID. Look it up first:

- a member  → `mopheus workspace member list --output json` → use `userId`
- an agent  → `mopheus agent list --output json` → use `id`
- a team    → `mopheus team list --output json` → use `id`

Match by display name. If the name is ambiguous or absent, do not guess — say
so in your comment instead of emitting a broken link.

## Step 2 — the types and what each enqueues

| To…                    | type     | id source      | What the backend does                                      |
| ---------------------- | -------- | -------------- | ---------------------------------------------------------- |
| trigger an agent       | `agent`  | agent.id       | enqueues a run for that agent                              |
| hand work to team      | `team`   | team.id        | resolves the team's leader agent and enqueues a run for it |
| link a member          | `member` | member.userId  | renders a link; enqueues NOTHING                           |
| cross-reference ticket | `ticket` | ticket.id      | renders a link to the ticket; enqueues NOTHING             |

`member` and `ticket` mentions do NOT trigger any agent run — they are rendered links only.

## Ticket cross-references (shorthand)

You do not need to build a `mention://ticket/` link manually. The backend auto-expands two shorthand formats when you post a comment:

- `W0-12` — workspace identifier prefix + number (e.g. `W0-12` → `[W0-12](mention://ticket/<uuid>)`)
- `#12` — hash + number (e.g. `#12` → `[#12](mention://ticket/<uuid>)`)

Both are skipped inside code blocks and inline code. Existing `mention://` links are also skipped to prevent double-wrapping.

If you prefer the explicit form (e.g. to control the label), build it yourself:

```
[W0-12](mention://ticket/<uuid>)
```

To get a ticket's UUID from its number, look it up via `mopheus ticket list --output json` or `mopheus ticket get <id> --output json`.

To find a ticket by title or description, use `mopheus ticket list --search "payment retry" --status todo`. The `--search` filter combines with other ticket-list filters.

## Using mentions when creating tickets

Mention links are not limited to comments — you can embed them directly in a
ticket description at creation time. The backend parses the same
`[@Label](mention://<type>/<id>)` shape inside `--description`,
`--description-stdin`, and `--description-file`.

```bash
# Mention an agent in the ticket description so the context is visible from the start
mopheus ticket create --title "Fix payment flow" --agent-id <agent-uuid> \
  --description "Root cause analysis needed. [@Backend Bot](mention://agent/<agent-uuid>) please investigate."
```

When using a heredoc (`--description-stdin`), place mention links inline — the
backend expands them the same way it does in comments. Remember the heredoc
rule: `--description-stdin` must be the last flag, immediately before `<<'EOF'`.

For the full list of `ticket create` flags, see the `mopheus-creating-tickets`
skill.

## Silent no-ops

The backend silently skips a mention when:
- The UUID is malformed or unknown.
- The target agent already has a pending or running task on this ticket.
- The agent is archived or not visible to the caller (private agent, no access).

In these cases no error is surfaced in the comment — the link renders but
nothing is enqueued. If you expect a task to fire and it does not, verify the
UUID and the agent's status with `mopheus agent list --output json`.

## Negative examples — what does NOT dispatch an agent

These shapes render or post fine but **do not enqueue any agent run**. The
ticket looks "dispatched" and then stalls silently. If you catch yourself
about to write any of these, stop and copy the matching mention link from
the Team Roster section of your runtime brief instead.

| Bad shape                                | Why it fails                                       |
| ---------------------------------------- | -------------------------------------------------- |
| `@Code Writer please handle this`        | Plain `@Name` — no mention link, no run queued.    |
| `@Code Writer 接手这个 bug`                | Plain `@Name` with CJK — same silent failure.      |
| `[Code Writer](mention://)`              | Truncated link — backend requires a real UUID.     |
| `[Code Writer](mention://agent/cw)`      | Non-UUID placeholder — link renders, no dispatch.  |
| `<@agent/UUID>` (Slack/Discord style)    | Non-platform shorthand — silently ignored.         |
| Plain text `Code Writer` with no `@`     | Human-readable only — no dispatch.                 |

The only dispatch shape is the platform mention link:

    [@Label](mention://agent/<uuid>)

Copy the link verbatim from the Team Roster. Re-typing the UUID is the most
common cause of "I dispatched but nothing happened".

### Pre-post bare-`@` scan

Before every `mopheus ticket comment add` that delegates work,
copy the complete mention link from the Team Roster. The complete link matches
`[@Label](mention://<type>/<id>)`; a bare `@Name` does not match this shape
and dispatches nothing. Replace a bare name with the roster link before
posting. If the target is absent from the roster, use the Step 1 lookup and
do not guess the UUID.
