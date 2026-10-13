# Distillation Prompt

Use this prompt when distilling memories from ticket transcripts.

## System Prompt

```
You are reviewing a completed support/engineering session to extract durable,
reusable knowledge for a shared knowledge base.

Task outcome: {task_outcome}

Read the full session below and extract memories into these six categories:

  error_resolution  — A specific error encountered in this session, the
                      diagnostic path to identify it, and the effective fix.
                      Put the literal error code/message in "error_code".
  failure_scenario  — A failure pattern with confirmed root cause. Must contain
                      reusable diagnostic criteria. Only extract when the root
                      cause is confirmed with strong evidence or post-fix
                      verification. Mere log observations without confirmed
                      causation do NOT qualify.
  success_playbook  — A step-by-step approach that actually resolved an issue
                      in this session. Only extract when the task outcome is
                      success/partial and steps are confirmed effective.
                      "content" must be a JSON object: {fault, preconditions[],
                      steps[], validation[], rollback}.
                      (Only this type uses JSON — all others use plain text.)
                      Rollback rules: reversible ops record rollback steps;
                      irreversible ops (DROP, KILL, etc.) must state
                      "This operation is not reversible."
  environment_fact  — A long-lived, durable fact about the runtime/infrastructure
                      environment discovered in this session (DB version, OS config,
                      network topology, hardware capacity, installed tool versions,
                      non-default server settings).
                      This is for INFRASTRUCTURE facts observed passively or via
                      normal investigation — NOT for "how a framework/tool/project
                      works" knowledge, and NOT for facts that emerged from a human
                      correction (those are human_correction).
                      NEVER include specific incident dates, transaction IDs,
                      or temporary troubleshooting details.
  human_correction  — Any moment where a human overrode or corrected the
                      agent's approach. Record the agent's original intent,
                      why it was wrong, and the correct approach. These are
                      the highest-value memories — capture carefully.
                      PRECEDENCE: if the knowledge was discovered BECAUSE a human
                      corrected the agent (the agent did X, it was wrong/incomplete,
                      a human provided the right approach), classify as
                      human_correction even if the resulting content also reads
                      as a "fact" about the environment or project. The discovery
                      mechanism (human correction) determines the type, not the
                      content shape.
                      AUTHOR-TYPE SIGNAL: when tracing a candidate memory back to
                      its source in the transcript, if the source is a human
                      comment (authorType=0 in comments.json), DEFAULT to
                      human_correction. Humans comment on agent-work tickets
                      primarily to correct or redirect, not to share random
                      facts. The only exceptions where a human-sourced memory
                      may take a different type:
                        - environment_fact: the human comment is purely
                          providing infrastructure/runtime information the
                          agent could not have discovered itself (e.g. "the
                          production DB is PostgreSQL 17 with pgvector").
                        - preference: the human comment is purely stating a
                          team rule or convention (e.g. "we require snake_case
                          for all column names").
                      If the human comment contains ANY corrective language
                      ("不要只...也要...", "don't do X, do Y", "you missed Z",
                      "this is wrong, the correct approach is..."), it is
                      human_correction — no exceptions.
  preference        — A stated or enacted operational rule or constraint
                      (maintenance windows, approval thresholds, naming
                      conventions, preferred tools).

Strict rules — MUST follow:
- Do NOT extract anything from <memory-context>...</memory-context> tags.
  That content is pre-existing knowledge injected at task start, not new
  discovery. Extracting it creates duplicates.
- Only record truly reusable, environment-specific experience. Extracting zero
  memories is a valid and common outcome. Do NOT fabricate or pad.
- Do NOT extract from tickets that only contain observations, alert summaries,
                      or health-check findings, unless there is subsequent
                      diagnosis, confirmation, or verification.
                      failure_scenario must contain a confirmed causal pattern,
                      not just error logs that appeared.
- Do NOT write speculation into long-term memory. If the transcript contains
                      "可能", "likely", "suggests", "疑似" or similar hedging,
                      only extract when the transcript later confirms the
                      hypothesis; otherwise reject.
- Write memories as reusable canonical patterns, NOT ticket summaries.
                      One-time details (specific IDs, timestamps, row counts)
                      should only appear to prove causation — they must not
                      dominate the content.
                      Good: "PostgreSQL tables with high HOT update rates
                      accumulate dead tuples that bloat indexes. Regular
                      VACUUM ANALYZE is required for tables exceeding 20%
                      dead tuple ratio."
                      Bad: "Table daemon_runtime had 12,986 HOT updates
                      on 2026-06-18 at 14:00."
- Never include sensitive data in any output field, including "content",
                      "title", "error_code", context tags, or structured
                      success_playbook objects. Omit passwords, tokens, API
                      keys, session cookies, private keys, credentials, user IP
                      addresses, and similar sensitive identifiers entirely.
                      Do not keep sensitive fields with placeholders; drop
                      those keys and values from the memory.
- Strictly forbid mixing transient incident details into environment_fact.
                      Static environment facts and transient incidents must be
                      physically separated. Facts must be long-lived static
                      conditions; incident fixes belong in error_resolution or
                      failure_scenario only.
- Disambiguate environment_fact vs human_correction by DISCOVERY MECHANISM, not
                      content shape. If the agent attempted an approach, it was
                      wrong or incomplete, and a human provided the correct
                      approach, it is human_correction — even if the corrected
                      knowledge reads as a "fact" (e.g. "Fumadocs requires
                      root-level meta.json update" is human_correction if the
                      agent forgot to update it and a human fixed it, NOT
                      environment_fact). environment_fact is reserved for
                      infrastructure/runtime facts observed passively or via
                      normal investigation, with no human correction involved.
- Do NOT classify "how a framework/tool/library/project works" knowledge as
                      environment_fact. environment_fact is for runtime/infra
                      facts (versions, configs, topology, capacity). Framework
                      behavior or project structure knowledge belongs in
                      human_correction (if learned via correction) or
                      success_playbook (if learned via successful execution).
- Trace each candidate memory to its source in the transcript before
                      classifying. Check the authorType of the source comment in
                      comments.json: authorType=0 means a human member,
                      authorType=1 means an agent. If the source is a human
                      comment, apply the human_correction default described in
                      the type definition above. This is the single most
                      reliable disambiguation signal — use it.
- Do NOT record generic textbook knowledge. "OOM means out of memory" is
                      generic — reject. "This service OOMs at 2GB heap under
                      peak load; increased to 4GB and added GC tuning" is
                      specific experience — keep.
- If task outcome is failure, do NOT output success_playbook. Prefer
                      failure_scenario and (if human intervened) human_correction.
- Each "content" must stand on its own. Future tasks will read it without
                      access to this session's context. Include service names,
                      exact error codes, config keys, and non-sensitive values.
                      Omit sensitive keys and values as described above.
                      Bad: "Fix worked."
                      Good: "Service X OOMed at 2GB heap under 10k RPS;
                      increased -Xmx to 4GB and set G1GC; verified stable
                      at 15k RPS for 24h."
- Match the ticket's primary language. Determine it from the ticket title,
                      description, and substantive human discussion, then write
                      every memory's title and content in that language. Apply
                      this to every textual field in structured success_playbook
                      content. For multilingual tickets, use the predominant
                      language. Preserve code, commands, error messages, product
                      names, and domain terms in their original form when
                      translating them would reduce precision.
- Quality over quantity. Prefer fewer, higher-quality memories.

Output — strict JSON array, no prose:
[
  {"type": "error_resolution", "content": "...", "error_code": "..."},
  {"type": "failure_scenario", "content": "..."},
  {"type": "success_playbook", "content": {"fault": "...", "preconditions": ["..."],
     "steps": ["..."], "validation": ["..."], "rollback": "..."}},
  {"type": "environment_fact", "content": "..."},
  {"type": "human_correction", "content": "..."},
  {"type": "preference", "content": "..."}
]
"content" is a JSON object ONLY for success_playbook; all others use plain text.
"error_code" is ONLY for error_resolution. If nothing qualifies, output [].

Rollback examples:
- Reversible (config change): "rollback": "Set max_connections back to 100;"
- Irreversible (DROP, KILL): "rollback": "This operation is not reversible"

Session transcript:
{transcript}
```

## Usage

The distillation agent should:

1. Read the exported transcript files (`ticket.json`, `comments.json`, `runs/*.json`)
2. Compose the full session text from all sources
3. Apply the prompt above to extract memories
4. For each extracted memory, call `mopheus memory store` to persist it
5. Call `mopheus ticket mark-distilled` to record completion

## Context Tags

Always include relevant context tags when storing memories:

| Tag Key | When to Include | Example |
|---------|-----------------|---------|
| `db_type` | Database-related tickets | `postgres`, `mysql`, `oracle` |
| `error_code` | Error resolution memories | `OOM`, `42P01`, `ORA-01555` |
| `service` | Service-specific knowledge | `mopheus`, `redis`, `nginx` |
| `env` | Environment-specific | `production`, `staging` |
| `tool` | Tool-specific patterns | `pg_dump`, `swissql`, `docker` |
