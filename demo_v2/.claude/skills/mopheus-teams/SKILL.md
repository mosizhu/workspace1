---
name: mopheus-teams
description: "需要使用 mopheus team 命令创建或管理团队时使用。/ Use when working as part of a Mopheus team — covers leader vs member roles, delegation via comment @mentions, and the no_action exit contract."
user-invocable: true
allowed-tools: Bash(mopheus *)
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Mopheus Teams

This skill covers the team operating model. Your runtime brief's Team Operating
Protocol section takes precedence — read it first. This skill fills in the
backend contracts that the brief does not fully encode.

## Leader vs member roles

Every team has exactly one leader agent. When a ticket is assigned to a team,
the platform enqueues a task for the **leader** — never directly for member
agents. Member agents only receive tasks when the leader explicitly delegates.

## Read teams

```bash
mopheus team list --output json                     # active teams
mopheus team list --include-archived --output json  # include archived
mopheus team get <id> --output json                 # full details
mopheus team member list <team-id> --output json    # members + roles (0=member, 1=lead)
```

## Create and update teams

```bash
# Create — leader is required
mopheus team create \
  --name "Team Name" \
  --leader <agent-name-or-id> \
  --description "..." \
  --output json

# Update name, description, or leader instructions
mopheus team update <id> \
  --name "New Name" \
  --description "..." \
  --instructions "..." \
  --output json

# Update instructions from stdin (safe for multi-line)
mopheus team update <id> --instructions-stdin <<'EOF'
Always start with a failing test.
EOF
```

## Manage team membership

```bash
# Add a regular member (role 0=member only; if team has no leader yet, role 1 is accepted)
mopheus team member add <team-id> \
  --member-id <uuid> \
  --member-type 1    # 1=agent, 0=workspace member

# Change leader — demotes existing leader, promotes target member
mopheus team member promote <team-id> <member-id>

# Remove a member (leader cannot be removed)
mopheus team member remove <team-id> <member-id>
```

## Archive and restore

```bash
mopheus team archive <id>   # soft-disable; team no longer receives new tickets
mopheus team restore <id>   # undo archive
```

## Delegating to team members

The leader delegates by posting a comment on the current ticket that @mentions the member agent:

```bash
mopheus ticket comment add <current-ticket-id> --content-stdin <<'EOF'
[@Agent Name](mention://agent/<member-agent-uuid>) Please analyze the repository and draft the plan.
EOF
```

Copy the target's complete mention link from the Team Roster; never use a bare `@Name`.

**Do NOT create sub-tickets autonomously.** Keep all work, conversation, and deliverables unified on the current primary ticket. Sub-tickets create context silos (attachments and prior discussion are isolated) and risk recursive task deadlocks. Only create sub-tickets if explicitly requested by a human user.

## Reporting outcomes with mopheus team activity

When your outcome is `no_action`, you MUST report it via the team activity
command and then exit immediately — do NOT post a comment:

```bash
mopheus team activity <ticket-id> no_action --reason "..."
```

For other outcomes, post your results as a normal ticket comment. The team
activity command is only required for `no_action`.

If you delegated in this turn, the outcome is `action`, not `no_action`.
Copy the target's complete mention link from the Team Roster;
never use a bare `@Name`. For a true `no_action` outcome, do not post a comment.

## Checking team member progress

Review the comments and activity timeline on the current ticket to track member progress:

```bash
mopheus ticket comment list <ticket-id> --output json
mopheus ticket get <ticket-id> --output json
```

Verify that delegated work and discussions in the ticket comments are complete before
marking the ticket done.

