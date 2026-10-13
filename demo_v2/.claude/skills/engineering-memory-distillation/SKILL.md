---
name: engineering-memory-distillation
description: "Specialty area memory distillation for software engineering and database operations tickets: errors, environment facts, and operational playbooks. / 软件工程与数据库运维工单的领域记忆蒸馏：错误、环境事实与操作方案。"
user-invocable: true
capabilities:
  - memory_distillation
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Memory Distillation — Engineering Specialty Area

Applies on top of `mopheus-memory-distillation-core` and `mopheus-memory-distillation-general` when the ticket is a software engineering, infrastructure, or database operations task. This skill specializes General's extraction rules; it never overrides Core.

Use this specialty area when the ticket involves: application or database errors and fixes, deployment/infrastructure changes, performance tuning, configuration, CI/CD, or incident response.

## Additional Extraction Focus

- **`resolution` (`error_resolution`)** — capture the literal error message/code, the diagnostic path, and the fix. Always set `error_code` in `context_tags` with its vendor/SQLSTATE prefix (see General rule 6).
- **`risk` (`failure_scenario`)** — only extract when the root cause is confirmed, not merely observed. Include the reusable diagnostic signal (e.g. a log pattern, metric threshold) that lets a future agent recognize the same scenario early.
- **`procedure` (`success_playbook`)** — store as a JSON object: `{fault, preconditions[], steps[], validation[], rollback}`. Reversible operations must record rollback steps; irreversible operations (`DROP`, `KILL`, force deletes) must state `"This operation is not reversible."`.
- **`fact` (`environment_fact`)** — DB/runtime versions, OS configuration, network topology, hardware capacity, non-default server settings. Never mix in transient incident details (specific timestamps, row counts, transaction IDs) — those belong in `resolution`/`risk`.
- Recommended `context_tags` keys: `db_type`, `component`, `os_type`, `cloud`, `deploy_tool`, `error_code`.

## Out of Scope

Do not use this specialty area for HR, legal, contract, or sales content — even if it happens to appear on an engineering ticket in passing. Extract only the engineering-relevant knowledge; route unrelated content to General or skip it.
