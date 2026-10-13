---
name: mopheus-memory-distillation-general
description: "Domain-independent memory extraction rules and the shared Memory Type definitions, applied after mopheus-memory-distillation-core. / 领域无关的记忆提取规则与共享的记忆类型定义，紧跟 mopheus-memory-distillation-core 应用。"
user-invocable: false
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Memory Distillation — General

Applies after `mopheus-memory-distillation-core`. Extract structured memories from completed tickets and store them in the database memory bank. These rules are domain-independent; a specialty area skill (if any is active for this run) specializes them but never replaces them.

## Memory Types

| Type (legacy identifier) | Generic name | Description | When to Extract |
|---|---|---|---|
| `error_resolution` | `resolution` | Error encountered and how it was resolved | Agent hit an error and found a fix |
| `failure_scenario` | `risk` | A failure pattern to watch for | Something failed repeatedly or in a surprising way |
| `success_playbook` | `procedure` | A proven approach that worked well | A multi-step solution that succeeded |
| `environment_fact` | `fact` | A runtime/infrastructure fact about the target environment | DB version, OS config, network topology, hardware capacity — NOT framework behavior or project structure |
| `human_correction` | `correction` | A human corrected the agent's approach | User said "don't do X, do Y instead" — DEFAULT classification when the source comment's authorType=0 (human). Takes precedence over `fact` unless the comment is purely providing infra info or stating a team rule |
| `preference` | `preference` | A user or team preference | "Always use snake_case", "Prefer CTEs over subqueries" |

`decision` is not a first-class type. Store decisions under the appropriate type above (usually `procedure` or `correction`) with `--context-tags '{"memory_subtype":"decision"}'`.

See `reference/distillation-prompt.md` for the full classification prompt, including edge cases and precedence rules between types.

## Extraction Rules

1. **Do NOT extract from `<memory-context>` content** — those are already memories.
2. **Do NOT extract textbook knowledge** — only extract environment-specific or project-specific knowledge.
3. **Quality over quantity** — extract 1-3 high-value memories per ticket, not every detail.
4. **One fact per memory** — each memory should contain exactly one piece of knowledge.
5. **Be specific** — include actual error messages, statements, config keys and safe values, etc.
6. **Tag context** — extract only the genuinely useful context from the ticket's metadata and labels into `context_tags` as ordinary keys (e.g. `db_type`, `error_code`, `labels`). Do NOT copy the full ticket metadata. Prefer `mopheus memory store --context-tags '{"db_type":"postgres","error_code":"42883"}'`. When `error_code` is set, it MUST carry its identifying prefix — vendor+number for vendor-specific errors (`ORA-00942`, `MYSQL-1062`, `MSSQL-8134`, `SQL0803N`) or the 5-character SQLSTATE for PostgreSQL (`42P01`, `42883`). NEVER store a bare vendor number (`1062`, `942`, `8134`): without the prefix the source system is ambiguous and retrieval cannot match it.
7. **Trace source before classifying** — for each candidate memory, identify which comment or agent run it came from. Check `authorType` in `comments.json`: 0=human, 1=agent. If the source is a human comment, default to `human_correction`/`correction` (see `reference/distillation-prompt.md` for the full rule and exceptions).
8. **Omit sensitive data entirely** — never store passwords, tokens, API keys, session cookies, private keys, credentials, user IP addresses, or similar sensitive identifiers in titles, content, context tags, error codes, or structured fields. Do not preserve sensitive fields with placeholders; drop those keys and values from the memory.
9. **Title is a one-line summary of the content** — the title must tell future readers what this memory is about and when it applies. It should be a precise headline of the fact below, not a category or label. Bad: `Error fix`, `DB issue`, `Login bug`. Good: `PostgreSQL 17 requires explicit CAST on JSONB path queries`, `MySQL 8 ignores index on JSON columns with functions`. If the title alone does not convey what the content will tell you, rewrite it.
10. **Drop transient source IDs from title, content, and context_tags** — never store ticket IDs, comment IDs, run IDs, message IDs, attachment IDs, or any other source-scoped identifier inside the memory body. They are meaningless outside the source ticket, pollute retrieval, and bloat the index. The `--ticket-id` flag already records provenance. If a specific comment or run matters, describe what was said or done, not its ID.
11. **Match the ticket's primary language** — determine the primary language from the ticket title, description, and substantive human discussion, then write every memory's title and content in that language. Apply this to all textual fields inside structured `success_playbook` content as well. For multilingual tickets, use the predominant language. Preserve code, commands, error messages, product names, and domain terms in their original form when translating them would reduce precision.
12. **Maintain while touching** — Treat distillation as an active maintenance cycle. Before storing a candidate memory, check whether existing memories are made obsolete, incomplete, or inaccurate by what you learned. When new findings supersede prior guidance (e.g. newer library versions, deprecated configs, corrected procedures), pass `--supersedes <old-memory-uuid>` to replace the outdated entry rather than letting contradictory records accumulate.
13. **Cognitive Promotion** — Strive to distill root causes rather than isolated symptoms. If an error resolution reveals a broader architectural rule, preferred pattern, or repeatable procedure, promote it to a higher-level type (`success_playbook` or `preference`) rather than leaving it as a narrow, one-off `error_resolution`.

## Workflow

### Single-ticket distillation

1. Analyze the transcript (see "Analyze the transcript" below).
2. Enrich the ticket's metadata (see "Enrich ticket metadata" below).
3. Check for similar memories (see below).
4. Store extracted memories (Core owns the storage command).
5. Mark the ticket as distilled (Core owns this step).

### Batch distillation

For each undistilled ticket found via `mopheus ticket list --undistilled`, repeat steps 1-4 above, then mark the source ticket as distilled. Do not change the source ticket's status.

### Analyze the transcript

Read ALL exported files: `ticket.json`, `comments.json`, `agent_tasks.json`, and `runs/*.json`.
The `runs/` directory contains the full agent conversation logs — these are the primary source for memory extraction.

Apply the distillation prompt from `reference/distillation-prompt.md` to analyze the session.
Follow its strict rules: quality over quantity, canonical patterns not ticket summaries,
no textbook knowledge, no speculation, no mixing incidents into environment facts.

### Enrich ticket metadata

During ticket handling the agent may not have recorded structured signals into metadata. Backfill the ones you can derive from the transcript — recall matches on metadata scalar pairs, so any structured key you write here makes this ticket's memories more retrievable later. This is optional and not exhaustive.

**Only extract core, genuinely useful structured facts — never pad with low-value or redundant keys.** Quality over quantity: a few high-signal keys beat a dump of everything.

```bash
mopheus ticket metadata list <ticket-id>          # read current keys first to avoid clobbering
mopheus ticket metadata set <ticket-id> --key <key> --value <value>
```

Rules:
- Set ONLY keys you derived from the transcript — never copy the whole metadata, never guess.
- Values must be primitives (string / number / bool).
- Skip if the metadata already has the key, or the transcript has nothing structured to add.

### Check for similar memories

Always search existing memories before writing new ones:

```bash
mopheus memory retrieve "the key insight" --context-tags '{"db_type":"postgres"}' --top-k 3
```

Reconciliation Matrix:

| Finding | Verdict | Action |
|---|---|---|
| Existing memory is made obsolete, deprecated, or corrected by this ticket | **Supersede** | Call `mopheus memory store --supersedes <old-uuid>` with the new insight. The system links and supersedes the old record, preventing stale guidance. |
| Existing memory is partially incomplete, missing an edge case, or needs expansion | **Consolidate** | Formulate a unified, comprehensive memory covering both cases and store it with `--supersedes <old-uuid>` so the knowledge base stays tidy. |
| Existing memory already accurately and fully captures this exact knowledge | **Skip** | Do not store. Omit duplicate insertion to avoid polluting retrieval and wasting space. |
| Existing memory is about a different domain, context, or complementary situation | **Store New** | Store as an independent new memory without `--supersedes`. |

### Cognitive Promotion

Do not settle for shallow incident logs. When distilling, evaluate where the knowledge sits on the cognitive ladder and promote it to the highest durable tier:

1. **Incident Resolution (`error_resolution` / `resolution`)**
   - *Scope:* A specific error or barrier encountered and resolved during execution.
   - *Example:* "PostgreSQL 17 rejects unquoted JSON path queries; wrap path in explicit CAST."
   - *When:* The learning is tightly bound to a specific runtime quirk, error code, or tool behavior.

2. **Standard Operating Procedure (`success_playbook` / `procedure`)**
   - *Scope:* A validated multi-step operational sequence, debugging protocol, or migration workflow.
   - *Promotion trigger:* If resolving the ticket required an orderly multi-step sequence that another agent or engineer will repeat in similar situations, promote the insight from isolated error fixes into a structured playbook with prerequisites, step-by-step actions, and verification commands.
   - *Example:* "PostgreSQL 15 to 17 Major Version Upgrade and Schema Cast Verification Playbook".

3. **Universal Convention (`preference`)**
   - *Scope:* A team standard, architectural guardrail, code style, or invariant boundary.
   - *Promotion trigger:* If human feedback or recurring troubleshooting demonstrates that a certain pattern should *always* or *never* be used project-wide, promote it into a project preference.
   - *Example:* "Always use typed error codes from errx package instead of raw HTTP 403 strings in handlers."
