---
name: mopheus-github-webhook
description: "收到 GitHub webhook 事件后处理 PR/Issue 同步和工单关联时使用。/ Use when processing GitHub webhook events — covers PR/Issue sync, ticket linking, auto-close, and auto-create flows."
user-invocable: true
allowed-tools: Bash(mopheus *)
features:
  - git
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Webhook Event Handler

This skill manages mirror records and ticket links via `mopheus` CLI only. Available commands: `mopheus repo list`, `mopheus repo pr/issue list|sync`, `mopheus repo link|unlink|links`.

## Pre-flight: Repo Identification

**Every event starts here.**

```bash
mopheus repo list --output json
```

Match the payload's `repository.html_url` or `repository.clone_url` against the repo table to get the repo URL. If no match is found, skip the event.

## Unsupported Actions

Only `opened`, `closed`, and `reopened` are handled for both `issues` and `pull_request` events. All other actions (`labeled`, `assigned`, `unlabeled`, `edited`, `milestoned`, `demilestoned`, `synchronize`, etc.) should be **skipped** — do not attempt to process them through any workflow path.

---

# Issue Workflow

Payload: `issues` event with `action` = opened / closed / reopened.

## Check Before Sync

```bash
# Check if issue already exists
mopheus repo issue list --repo <url> --number <N> --output json

# Check if already linked to a ticket
mopheus repo links --repo <url> --type git_issue --number <N>
```

- Issue does not exist → run sync
- Issue exists and `state` matches → skip sync
- Issue exists but `state` differs → run sync to update
- Already linked to ticket → skip link
- Not linked → run link after sync

## issues.opened

```bash
mopheus repo issue sync \
    --repo <url> --number <N> --title "..." --state open \
    --author <login> --auto-link
```

`--auto-link` matches ticket identifiers (e.g. MOPHEUS-42) in the issue title/body. When matched, association is created automatically.

**No match found** — optionally create a ticket and link manually:

```bash
# 1. Create the ticket
mopheus ticket create --title "..." --description "..." --priority medium

# 2. Link using the entity UUID from the sync output above
mopheus repo link --ticket <new-ticket-id> --entity-type git_issue --entity <issue-uuid>
```

## issues.closed

```bash
mopheus repo issue sync --repo <url> --number <N> --state closed --auto-link --output json
```

After sync, evaluate ticket auto-done:
1. Linked ticket is not in a terminal status (done / cancelled)
2. No associated PR is still open/draft — check with:
   ```bash
   mopheus repo links --ticket <ticket-id> --type git_pull_request --output json
   ```
3. This issue is closed

All three met → `mopheus ticket status <ticket-id> done`.

## issues.reopened

```bash
mopheus repo issue sync --repo <url> --number <N> --state open --auto-link
```

Do NOT revert ticket status.

---

# Pull Request Workflow

Payload: `pull_request` event with `action` = opened / closed / reopened.

## Check Before Sync

```bash
# Check if PR already exists
mopheus repo pr list --repo <url> --number <N> --output json

# Check if already linked to a ticket
mopheus repo links --repo <url> --type git_pull_request --number <N>
```

- PR does not exist → run sync
- PR exists and `state` matches → skip sync
- PR exists but `state` differs → run sync to update
- Already linked to ticket → skip link
- Not linked → run link after sync

## pull_request.opened

```bash
mopheus repo pr sync \
    --repo <url> --number <N> --title "..." --state open \
    --author <login> --head <ref> --base <ref> \
    --auto-link --output json
```

`--auto-link` matches ticket identifiers (e.g. MOPHEUS-42) in the PR title/body. When matched, association is created automatically.

**No match found** → optionally create a ticket and link manually:

```bash
# 1. Create the ticket
mopheus ticket create --title "..." --description "..." --priority medium

# 2. Link using the entity UUID from the sync output above
mopheus repo link --ticket <new-ticket-id> --entity-type git_pull_request --entity <pr-uuid>
```

## pull_request.closed (merged)

```bash
mopheus repo pr sync \
    --repo <url> --number <N> --title "..." --state closed \
    --author <login> --head <ref> --base <ref> \
    --merged --close-keyword \
    --additions <N> --deletions <N> --changed-files <N> \
    --auto-link
```

`--close-keyword` indicates closing keywords detected in the PR body. The sync command automatically sets `ticket_entity_link.close_intent` to true when this flag is present.

After sync, evaluate ticket auto-done:
1. Linked ticket is not in a terminal status (done / cancelled)
2. No open/draft PR remains on this ticket — check with:
   ```bash
   mopheus repo links --ticket <ticket-id> --type git_pull_request --output json
   ```
3. At least one merged PR has `close_intent=true`

All three met → `mopheus ticket status <ticket-id> done`.

## pull_request.closed (not merged)

```bash
mopheus repo pr sync \
    --repo <url> --number <N> --title "..." --state closed \
    --author <login> --head <ref> --base <ref> \
    --auto-link --output json
```

Add a comment on the linked ticket noting the PR was closed without merge:

```bash
mopheus ticket comment add <ticket-id> --content "PR #<N> was closed without merge."
```

## pull_request.reopened

```bash
mopheus repo pr sync --repo <url> --number <N> --state open --auto-link
```

Do NOT revert ticket status.

---

# Reference

## Auto-link Regex

Matches ticket identifiers: `[A-Z][A-Z0-9]{1,9}-\d+` (e.g. MOPHEUS-42).
Also matches `#N` format (repo-scoped issue/PR number).

## Closing Keywords

Pattern: `(?i)\b(?:close[sd]?|fix(?:e[sd])?|resolve[sd]?)[:\s]+(?:#\d+|[A-Z][A-Z0-9]{1,9}-\d+)`

Matches phrases like "Fixes #123", "Closes MOPHEUS-42", "Resolved #7".

## Manual Link

When auto-link fails to match, use the entity UUID from the sync output (`--output json`):

```bash
mopheus repo link --ticket <id> --entity-type git_pull_request --entity <pr-uuid> [--close-intent]
mopheus repo link --ticket <id> --entity-type git_issue --entity <issue-uuid>
```

If the entity was synced in a previous run and you don't have the UUID, retrieve it:

```bash
mopheus repo pr list --repo <url> --number <N> --output json
mopheus repo issue list --repo <url> --number <N> --output json
```

## Auto-create Tickets (Optional)

Controlled by workspace setting `auto_create_tickets` (default: off). When enabled and no ticket identifier is matched by auto-link:

1. Create a ticket from the webhook payload:
   ```bash
   mopheus ticket create \
       --title "<issue/PR title>" \
       --description "<issue/PR body>" \
       --priority medium
   ```
2. Link the newly created ticket to the entity using the UUID from the sync output:
   ```bash
   mopheus repo link --ticket <new-ticket-id> --entity-type git_issue --entity <entity-uuid>
   ```

Derive fields from the payload:
- `--title`: `issue.title` or `pull_request.title`
- `--description`: `issue.body` or `pull_request.body`
- `--priority`: default `medium` (workspace convention)

## Error Handling

- If `mopheus repo list` returns empty or no match → skip the event silently.
- If `mopheus repo issue sync` or `mopheus repo pr sync` fails → log the error and skip. Do not retry transient failures in the webhook handler; the event will not be re-delivered.
- If required payload fields are missing (`repository.html_url`, `issue.number`, etc.) → skip the event.
