---
name: mopheus-jobs
description: "需要使用 mopheus job 命令创建或管理 Job 自动化时使用。/ Use when working with mopheus job commands — covers schedule, event, and webhook triggers, event-filter specification, action config, and run history."
user-invocable: true
allowed-tools: Bash(mopheus *), Read, Write
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Jobs

Jobs trigger automated actions on a schedule, on an event, or via webhook.

## Create a Job

```bash
# 1. Schedule job with its initial cron trigger
mopheus job create \
  --name "Daily Standup" \
  --trigger-type schedule \
  --action-type create_ticket \
  --action-config '{"title":"Daily standup","agentId":"<uuid>"}' \
  --instruction "Optional extra instructions for the agent" \
  --cron "0 0 9 * *" \
  --timezone "Asia/Shanghai" \
  --output json

# 2. Event job (real-time platform events)
mopheus job create \
  --name "Bug Ticket Auto-Triage" \
  --trigger-type event \
  --action-type assign_agent \
  --action-config '{"agentId":"<agent-uuid>"}' \
  --instruction "A new bug ticket #{{ticket.number}} ({{ticket.title}}) was created. Description: {{ticket.description}}. Triage and reproduce." \
  --event-filter '{"event":"ticket","actions":["created"],"conditions":{"labels":["bug"]}}'
```

### `--trigger-type`

| Value | When it fires |
| --- | --- |
| `schedule` | On a cron schedule; pass `--cron` to create its initial trigger |
| `event` | On a platform event (add via `trigger-add` or `--event-filter`) |
| `webhook` | On an HTTP POST to the job's webhook URL |

### `--action-type` and `--action-config`

| `--action-type` | `--action-config` keys | Effect |
| --- | --- | --- |
| `create_ticket` | `title`, `priority`, `projectId`, `agentId` | Creates a ticket; if `agentId` is set, enqueues a run |
| `assign_agent` | `agentId` | Assigns the agent to an existing ticket |
| `send_notification` | `message` | Sends a notification to workspace members |

`--action-config` must be valid JSON.

### Schedule flags on `job create`

For `--trigger-type schedule`, `--cron` creates the job's first schedule trigger.
Without `--cron`, the job is created without a trigger and one can be added later with
`job trigger-add`.

- `--cron-dialect standard` uses `minute hour day-of-month month day-of-week`, for
  example `0 9 * * 1-5` for weekdays at 09:00.
- `--cron-dialect quartz` uses `second minute hour day-of-month month day-of-week year`,
  with weekday numbers `SUN=1, MON=2, TUE=3, WED=4, THU=5, FRI=6, SAT=7`,
  for example `0 0 9 ? * 2-6 *` for weekdays at 09:00. Use `?` for either
  day-of-month or day-of-week when the other field is specified.
- `--timezone` selects the IANA timezone. `--start-at` is the earliest eligible run
  (default: now); `--expires-at` is an exclusive deadline, so occurrences at or after
  it do not run.

---

## Event-Type Jobs & `event-filter`

Event filters determine which domain events trigger the job.

### Structure of `event-filter`

```json
[
  {
    "event": "ticket",
    "actions": ["created", "updated"],
    "conditions": {
      "labels": ["bug"],
      "priority": [3, 4],
      "status": 1
    }
  }
]
```

- **`event`** (required): `ticket`, `comment`, `agent_task`, `runtime`
- **`actions`** (optional): Filter specific actions. Empty/omitted = any action.
- **`conditions`** (optional): Match payload properties. Supports scalar comparison, slice/OR matching (`[3, 4]`), and array tag containment (`labels: ["bug"]`).

### Supported Domain Events & Variables

- **`ticket`**: actions (`created`, `updated`, `deleted`), variables (`id`, `number`, `title`, `description`, `status`, `priority`, `assigneeId`, `projectId`, `labels`)
- **`comment`**: actions (`created`, `updated`, `deleted`), variables (`id`, `ticketId`, `ticketTitle`, `ticketNumber`, `ticketStatus`, `authorType`, `content`, `type`)
- **`agent_task`**: actions (`created`, `updated`, `completed`, `failed`, `interaction`), variables (`id`, `ticketId`, `agentId`, `runtimeId`, `status`, `failureReason`, `instruction`)
- **`runtime`**: actions (`offline`, `register`, `updated`), variables (`id`, `daemonId`, `name`, `provider`, `status`, `enabled`)

### Introspect Event Schemas

```bash
mopheus job event-list
mopheus job event-schema ticket
mopheus job event-schema comment
mopheus job event-schema agent_task
mopheus job event-schema runtime
```

---

## Add a Trigger

```bash
# 1. Schedule trigger (cron)
mopheus job trigger-add <job-id> \
  --kind schedule \
  --cron "0 9 * * 1-5" \
  --timezone "Asia/Shanghai" \
  --label "Weekday 9am"

# 2. Event trigger
mopheus job trigger-add <job-id> \
  --kind event \
  --label "Human comments on tickets" \
  --event-filter '{
    "event": "comment",
    "actions": ["created"],
    "conditions": {
      "authorType": 0,
      "type": 0,
      "ticketStatus": [1, 2, 3]
    }
  }'

# 3. Webhook trigger
mopheus job trigger-add <job-id> --kind webhook --label "External hook"
```

## Update a Trigger

```bash
# Update schedule trigger
mopheus job trigger-update <job-id> <trigger-id> \
  --enabled true \
  --cron "0 10 * * 1-5" \
  --timezone "Asia/Shanghai"

# Update event trigger filter
mopheus job trigger-update <job-id> <trigger-id> \
  --event-filter '[{"event":"ticket","actions":["created"],"conditions":{"priority":[3,4]}}]' \
  --enabled true
```

## Delete a Trigger

```bash
mopheus job trigger-delete <job-id> <trigger-id>
```

## Webhook URL

```bash
# Rotate the webhook URL (invalidates old URL)
mopheus job trigger-rotate-url <job-id> <trigger-id>
```

## Manually Fire a Job

```bash
mopheus job trigger <job-id>
```

## Run History

```bash
mopheus job runs <job-id> --limit 20 --offset 0
```

## Update a Job

```bash
mopheus job update <job-id> \
  --name "New Name" \
  --description "..." \
  --instruction "..." \
  --enabled true \
  --action-config '{"title":"Updated title","agentId":"<uuid>"}'
```

## Delete a Job

```bash
mopheus job delete <job-id>
```

## List Jobs

```bash
mopheus job list --output json
mopheus job get <job-id> --output json
```
