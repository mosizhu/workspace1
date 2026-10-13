---
name: mopheus-channel-messaging
description: "Use the Mopheus CLI to inspect channel installations, user bindings, chat sessions, and send messages through WeCom, DingTalk, or Feishu. / 使用 Mopheus CLI 查看渠道安装、用户绑定和渠道会话，并通过企业微信、钉钉或飞书发送消息。"
user-invocable: true
allowed-tools: Bash(mopheus *)
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Channel Messaging

Use this skill for WeCom, DingTalk, and Feishu installations, user bindings, channel chat sessions, and outbound messages.

## Workflow

1. List available agents and teams first:

   ```bash
   mopheus agent list --output json
   mopheus team list --output json
   ```

2. List available installations. Filter by either an `agent_id` or a `team_id`:

   ```bash
   mopheus channel list --agent-id <agent-id> --output json
   mopheus channel list --team-id <team-id> --output json
   ```

3. List Mopheus users bound to an installation:

   ```bash
   mopheus channel bindings <installation-id> --output json
   ```

4. List channel sessions for an installation:

   ```bash
   mopheus channel chat <installation-id> --output json
   ```

5. Send through the existing `chat_session`:

   ```bash
   mopheus channel send --session <chat-session-id> --content "message content"
   ```

   The legacy command remains compatible:

   ```bash
   mopheus chat send-channel <chat-session-id> --content "message content"
   ```

   For an installation-targeted message, use the installation ID and Mopheus user ID:

   ```bash
   mopheus channel send --installation <installation-id> --user <mopheus-user-id> --content "message content"
   ```

## Safety Checks

- Confirm that the installation belongs to the current workspace and is active before sending.
- Confirm that the target user appears in the installation's binding list before sending.
- Never guess or construct external platform identifiers.
