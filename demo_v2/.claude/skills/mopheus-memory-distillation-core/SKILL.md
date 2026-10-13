---
name: mopheus-memory-distillation-core
description: "System-owned rules for memory distillation: transcript loading, storage, status, and safety. Always applied together with mopheus-memory-distillation-general and at most one specialty area skill. / 记忆蒸馏的系统核心规则：转录加载、存储、状态与安全。始终与 mopheus-memory-distillation-general 及最多一个领域技能一同应用。"
user-invocable: false
allowed-tools: Bash(mopheus memory *), Bash(mopheus ticket *), Read
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# ⛔ FORBIDDEN ACTIONS — READ FIRST

Before doing ANYTHING, internalize these rules:

1. **NEVER use the `Write` tool.** Not for memory files, not for MEMORY.md, not for anything.
2. **NEVER write to `.claude/memory/`** or any local directory. That is the Claude Code superpowers memory system — it is UNRELATED to this task.
3. **ONLY use `mopheus memory store`** CLI command to store memories. It writes to the database.
4. **ONLY use `mopheus ticket mark-distilled`** to mark completion.

Violating these rules means the memories are LOST — they go to local files that no one can retrieve.

This Core skill is mandatory and cannot be overridden or narrowed by `mopheus-memory-distillation-general` or by any specialty area skill. Specialty area guidance may add extraction focus on top of General; it may never relax these rules, change the tool allowlist, or change how tickets/memories are read or written.

## Skill Set For This Run

Every distillation run applies exactly:

- This Core skill (always).
- `mopheus-memory-distillation-general` (always).
- Zero or one specialty area skill, selected before you start extracting (see "Specialty Area Skill Selection" below).

## Specialty Area Skill Selection

Your instruction states one of three modes:

- **General only** — no specialty area skill applies. Proceed directly to General's extraction rules.
- **Specialty: `<skill-name>`** — the user already chose this specialty area skill. Read it (via the `Skill` tool, or the file path given in your instruction) before extracting. Do not substitute a different specialty area skill.
- **Auto** — your instruction includes a catalog of eligible specialty area skills (ID, name, description, builtin flag, version). Before extracting:
  1. Read the full ticket transcript first (see "Analyze the transcript" in General).
  2. Compare the ticket against the catalog's names/descriptions.
  3. If exactly one specialty area skill clearly applies, load it (this call is recorded server-side; you may re-load the same one, but you cannot switch to a different one afterward — choose deliberately).
  4. If none clearly applies, continue with General only. Do not force a specialty area fit.

Never apply more than one specialty area skill in a single run.

## Distillation Mode — Status Rules

Determine your mode BEFORE starting work:

**Core rule:** You can NEVER change the status of a ticket you are distilling FROM (the source ticket). You can only change the status of your OWN distillation task ticket.

### Single-ticket distillation

You are distilling memories from ONE specific ticket. The ticket you are working on IS the source ticket — there is no separate distillation task ticket.

- **Do NOT change its status.** It is read-only.
- Only extract memories and mark it as distilled via `mopheus ticket mark-distilled`.

### Batch distillation

You are distilling memories from MULTIPLE completed tickets. You have your own distillation task ticket (either created by a job or by yourself).

- You MAY change the status of YOUR OWN distillation task ticket: `in_progress` when starting, `in_review` when done.
- Do NOT change the status of the source tickets you are distilling from.

## Workflow Skeleton

1. Determine your distillation mode (single-ticket or batch) and your specialty area selection mode (General only / Specialty / Auto) from the instruction.
2. Get the ticket transcript: `mopheus ticket transcript <ticket-id>`.
3. Resolve the specialty area skill per "Specialty Area Skill Selection" above.
4. Follow `mopheus-memory-distillation-general` (and the resolved specialty area skill, if any) to analyze the transcript, enrich metadata, check for similar memories, and store memories.
5. Perform the Pre-completion Quality Check (see "Pre-completion Quality Check" below).
6. Mark the ticket as distilled (see "Mark ticket as distilled" below).
7. Post your results as a comment. Respect the status rules above — do not change the source ticket's status.

## Storage Rules

**MUST use `mopheus memory store` CLI. NEVER use the Write tool or create local files.**

```bash
mopheus memory store \
  --type error_resolution \
  --title "PostgreSQL JSONB CAST requirement" \
  --content "PostgreSQL 17 requires explicit CAST for JSONB path queries. Use (col->>'key')::int instead of col->>'key'::int." \
  --context-tags '{"db_type":"postgres","error_code":"42883"}' \
  --ticket-id <ticket-id>
```

When superseding an outdated, inaccurate, or obsolete memory discovered during reconciliation, pass `--supersedes <old-memory-uuid>` to replace it and maintain the memory bank:

```bash
mopheus memory store \
  --type error_resolution \
  --title "PostgreSQL JSONB CAST requirement (PG 17+)" \
  --content "PostgreSQL 17+ requires explicit CAST for JSONB path queries. Use (col->>'key')::int instead of col->>'key'::int." \
  --supersedes <old-memory-uuid> \
  --context-tags '{"db_type":"postgres","error_code":"42883"}' \
  --ticket-id <ticket-id>
```

`--type` accepts either the legacy identifiers (`error_resolution`, `failure_scenario`, `success_playbook`, `environment_fact`, `human_correction`, `preference`) or their generic aliases (`resolution`, `risk`, `procedure`, `fact`, `correction`, `preference`) — both resolve to the same stored value. A specialty area skill may additionally set `memory_specialty` and `memory_subtype` (e.g. `memory_subtype: decision`) inside `--context-tags`; these are ordinary tags, not a new memory type.

Repeat for each extracted memory. Each call stores one memory in the database.

## Pre-completion Quality Check

Before calling `mopheus ticket mark-distilled`, you MUST verify every candidate memory against these four quality gates. If a candidate violates any gate, adjust or drop it before proceeding:

1. **No Credentials or Sensitive Data** — Verify that titles, content, and context tags contain ZERO passwords, API keys, tokens, session cookies, private keys, or user credentials. If sensitive information appears in raw evidence, drop or sanitize it completely (do not rely on placeholder tokens).
2. **No Raw Log or Transcript Dumps** — Verify that content contains distilled insights, not unprocessed stack traces, multi-page log dumps, or full JSON responses. Extract only the root cause, syntax requirement, or resolution rule.
3. **Cross-session Enduring Value** — Verify that the memory provides actionable guidance across future tickets and sessions. Reject transient single-use state, ephemeral ticket progress summaries, or obvious generic textbook knowledge.
4. **Deduplication and Supersession Reconciled** — Verify that similar existing memories were searched via `mopheus memory retrieve`. If a prior memory is now superseded or made obsolete by new findings, explicitly link it using `--supersedes <old-memory-uuid>` instead of creating blind duplicates.

## Mark ticket as distilled

```bash
mopheus ticket mark-distilled <ticket-id> --memory-count 2 --db-type postgres --yes
```

If no memories were extracted:

```bash
mopheus ticket mark-distilled <ticket-id> --skipped-reason "No actionable memories found" --yes
```

## Failure Handling

If `mopheus ticket transcript`, `mopheus memory store`, or `mopheus ticket mark-distilled` fails, do not silently continue or fabricate a result. Report the failure in your final comment and leave the ticket's distillation status unset so a retry (including `force=true` re-distillation) is possible.

## Do NOT re-distill injected memory

Never extract a "new" memory from content that was injected into your context as `<memory-context>` — that content is pre-existing knowledge from a prior distillation run, not new discovery. Extracting it again creates duplicates and inflates `occurrence_count` incorrectly.
