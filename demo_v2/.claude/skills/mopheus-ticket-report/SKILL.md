---
name: mopheus-ticket-report
description: "需要对已完成或关闭的工单生成归档报告时使用，覆盖数据采集、报告结构和归档方式。/ Use when a Mopheus ticket needs an archive report — covers transcript fetching, report structure, and archiving."
user-invocable: true
allowed-tools: Bash(mopheus *), Write, Read
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Mopheus Ticket Archive Report

Generate a complete archive report from a ticket's full history and post it as a comment for long-term reference.

## Step 1 — Fetch the full transcript

```bash
mopheus ticket transcript <ticket-id>
```

The transcript returns the ticket's full timeline in one call: metadata, status changes, comments, agent task runs, and run messages in chronological order.

If the command is unavailable, assemble the same data manually:

```bash
mopheus ticket get <ticket-id>
mopheus ticket comment list <ticket-id>
mopheus ticket runs <ticket-id> --recursive
# For each run-id returned above:
mopheus ticket run-messages <agent-task-id>
```

## Step 2 — Analyze the data

Read the transcript carefully before writing anything. For each section of the template, identify the evidence first:

- **结论** — did the ticket fully achieve its goal? what was the root cause? any residual risk?
- **原因分析** — why did this ticket exist, what caused each blocker and how was it resolved
- **数据依据** — for each claim in the conclusion and metrics, identify the exact command/query that produced the evidence, the key output, and one sentence explaining what it proves; omit ticket metadata fields that add no analytical value

## Step 3 — Write the report

Follow the structure in [report-template.md](templates/report-template.md). Fill every section with concrete facts from the transcript — no filler, no hedging.

Write the completed report to a local file first:

```bash
# Use the Write tool to create the file
# Path: /tmp/<ticket-id>-report.md
```

## Step 4 — Post and archive

Determine the next version number by counting existing report comments on the ticket (comments whose content starts with "Archive report"), then increment by one. Use today's date in `YYYYMMDD` format.

Name the file: `<ticket-id>-report-v<N>-<YYYYMMDD>.md`

Upload the report file as an attachment and post a new comment:

```bash
mopheus ticket comment add <ticket-id> \
  --content "Archive report v<N> (<YYYY-MM-DD>) attached." \
  --attachment /tmp/<ticket-id>-report-v<N>-<YYYYMMDD>.md
```

Each report generation always creates a new versioned comment. Never edit or delete previous report comments.

## Rules

- **Pull facts, never invent.** Every claim in the report must trace back to the transcript.
- **Prefer concrete values over vague summaries.** Duration in minutes, PR numbers, exact error messages — not "some work was done".
- **Omit empty sections.** If there were no blockers, drop that section entirely.
- **One report per ticket.** If a report comment already exists (search comments for "# Ticket Report"), update it instead of adding a duplicate.
