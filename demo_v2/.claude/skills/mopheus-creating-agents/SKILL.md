---
name: mopheus-creating-agents
description: "需要使用 mopheus agent 命令创建或管理智能体时使用。/ Use when working with mopheus agent commands — covers create/update/delete/archive/restore, skill management, and env vars."
user-invocable: true
allowed-tools: Bash(mopheus *), Read, Write
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Managing Agents

## Before Creating an Agent — Check Available Runtimes

**Always check for an online runtime before creating an agent.** An agent without a runtime cannot execute any tasks.

```bash
mopheus runtime list --output json
```

A runtime is usable when its `status` is `online` or `busy`. If the list is empty or every runtime is `offline` / `error`, **stop immediately** and tell the user:

Do not create the agent until at least one runtime is available.

## Create an Agent

Once a usable runtime exists, create the agent and **always pin it to that runtime** with `--runtime-id`:

```bash
# 1. Pick the runtime ID from the list above
RUNTIME_ID="<id-from-runtime-list>"

# 2. Create the agent pinned to that runtime
mopheus agent create \
  --name "Agent Name" \
  --description "What this agent does" \
  --instructions "System prompt / identity" \
  --runtime-id "$RUNTIME_ID" \
  --max-concurrent 1 \
  --visibility workspace \
  --output json
```

| Flag | Required | Description |
| --- | --- | --- |
| `--name` | yes | Display name |
| `--description` | no | Short description |
| `--instructions` | no | System prompt (agent identity) |
| `--runtime-id` | **yes** | Pin to the online runtime (required — do not omit) |
| `--max-concurrent` | no | Max parallel tasks (default 1) |
| `--visibility` | no | `workspace` (default) or `private` |

## Update an Agent

```bash
mopheus agent update <agent-id> \
  --name "New Name" \
  --description "..." \
  --instructions "..." \
  --max-concurrent 2 \
  --visibility workspace
```

All flags are optional — only provided flags are updated.

## Delete, Archive, Restore

```bash
mopheus agent delete <agent-id>    # permanent
mopheus agent archive <agent-id>   # soft-disable; tasks no longer enqueued
mopheus agent restore <agent-id>   # undo archive
```

An archived agent keeps its history but receives no new tasks. Prefer archive over delete when you may want to restore later.

## Manage Skills

```bash
# List skills attached to an agent
mopheus agent skills list <agent-id>

# Add a single skill
mopheus agent skills add <agent-id> --skill-id <skill-uuid>

# Remove a single skill
mopheus agent skills remove <agent-id> --skill-id <skill-uuid>

# Replace all skills at once
mopheus agent skills set <agent-id> --skill-ids <uuid1>,<uuid2>,...
```

Skills are additional instruction blocks injected into the agent's context at task claim time. Built-in skills (prefixed `mopheus-`) are appended automatically and do not need to be managed here.

## Configure Env Vars

```bash
# View current env (values redacted for secrets)
mopheus agent env get <agent-id>

# Set via inline JSON
mopheus agent env set <agent-id> --env '{"API_KEY":"value","BASE_URL":"https://..."}'

# Set via stdin (useful for large or multiline values)
echo '{"API_KEY":"value"}' | mopheus agent env set <agent-id> --env-stdin

# Set via file
mopheus agent env set <agent-id> --env-file /path/to/env.json
```

Env vars are merged — existing keys not present in the new payload are preserved. To remove a key, set it to `null` in the JSON.

## List / Get Agents

```bash
mopheus agent list --output json
mopheus agent get <agent-id> --output json
```

Use `agent list` to look up UUIDs when creating mentions or assigning tickets.
