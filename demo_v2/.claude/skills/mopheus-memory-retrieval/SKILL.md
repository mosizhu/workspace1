---
name: mopheus-memory-retrieval
description: "Retrieve relevant memories from the memory bank. / 从记忆库检索相关记忆。"
user-invocable: true
allowed-tools: Bash(mopheus memory *)
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Memory Retrieval

Retrieve relevant memories from the memory bank for prior context, past decisions, or known resolutions.

## Relationship to the runtime workflow

The runtime workflow already triggers memory retrieval at the right moments — before answering in chat, during ticket analysis, on job runs. **You do not need to invoke this skill for routine work.** Follow the runtime instructions directly.

Invoke this skill only when you need **additional, targeted retrieval** beyond what the workflow already does:

- A broad keyword search came back empty and you want to try different keywords or narrower filters
- You need to deep-dive on a specific topic mid-task (e.g., "all past decisions about connection pooling")
- Earlier retrieval is now stale after new information surfaced in the ticket

## Build a search query

Extract the key concepts from the task:
- The type of problem (error, configuration, performance, etc.)
- The target environment (database type, version, etc.)
- Any specific error codes or messages

## Search memories

```bash
mopheus memory retrieve "PostgreSQL JSONB type casting error" \
  --context-tags '{"db_type":"postgres"}' \
  --top-k 3 \
  --output json
```

Or search by memory type:

```bash
mopheus memory retrieve "connection timeout" \
  --types "error_resolution,failure_scenario" \
  --top-k 5 \
  --output json
```

Always pass `--output json` so the results are machine-parseable.

## Use retrieved memories

If memories are found, incorporate them into your approach:
- **error_resolution** — apply the known fix directly
- **failure_scenario** — avoid the known failure pattern
- **success_playbook** — follow the proven approach
- **environment_fact** — account for the environment specifics
- **human_correction** — follow the corrected approach
- **preference** — respect the stated preference

## No memories found

If the CLI returns empty, proceed with your best judgment. New memories will be extracted from this ticket after completion.

## Tips

- Use specific keywords for better results
- Filter by `context-tags` to narrow to the relevant environment
- Use `--types` to focus on specific memory categories
- Higher similarity scores indicate closer semantic match; read the content alongside the score — there is no fixed cutoff
