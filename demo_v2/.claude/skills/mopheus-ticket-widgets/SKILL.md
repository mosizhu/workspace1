---
name: mopheus-ticket-widgets
description: "在工单评论与描述中嵌入交互式决策组件（单选、多选、确认操作卡片）时使用。/ Use when embedding interactive decision widgets (single-select, multi-select, confirm-action) in ticket comments or descriptions."
user-invocable: true
allowed-tools: Bash(mopheus ticket comment *), Bash(mopheus ticket create *), Bash(mop *)
features:
  - widget
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Interactive Decision Widgets (Ticket Widgets)

Use interactive decision widgets to present structured, clickable choices or approvals directly within ticket comments or descriptions.

When you need the user or team to choose between implementation paths, select parameters, pick target services, or confirm high-risk actions, **do NOT rely only on raw text bullet points**. Embed an interactive widget code fence (` ```widget ... ``` `). The Mopheus Web UI renders it into a native, interactive decision card where users can click or submit choices without manual typing.

## 1. Widget Types & JSON Specifications

Every widget block must use valid JSON with `v: "1"`, a unique `id`, `type`, `label`, and `props`.

### Single Select (`single-select`)
For mutually exclusive choices (e.g. selecting an architectural plan, deployment strategy, or rollback target):

````markdown
```widget
{
  "v": "1",
  "id": "w_plan_choice",
  "type": "single-select",
  "label": "选择执行方案",
  "props": {
    "options": [
      { "value": "plan_a", "label": "方案 A (推荐)", "description": "在线热迁移，无服务中断", "intent": "positive" },
      { "value": "plan_b", "label": "方案 B (维护窗口)", "description": "停机重建索引，耗时 10 分钟", "intent": "danger" }
    ],
    "default": "plan_a"
  }
}
```
````

### Multi Select (`multi-select`)
For selecting one or more options with optional selection bounds:

````markdown
```widget
{
  "v": "1",
  "id": "w_services_choice",
  "type": "multi-select",
  "label": "选择目标服务",
  "props": {
    "options": [
      { "value": "auth-svc", "label": "Auth Token Service" },
      { "value": "gateway-svc", "label": "API Gateway Service" },
      { "value": "worker-queue", "label": "Async Worker Queue" }
    ],
    "minSelected": 1,
    "maxSelected": 2
  }
}
```
````

### Confirm Action (`confirm-action`)
For critical approvals, production releases, destructive commands, or branching flows:

````markdown
```widget
{
  "v": "1",
  "id": "w_confirm_deploy",
  "type": "confirm-action",
  "label": "确认发布生产环境",
  "props": {
    "description": "变更将执行生产 DDL 结构调整，请确认维护窗口放行。",
    "actions": [
      { "intent": "positive", "label": "确认执行" },
      { "intent": "negative", "label": "终止操作" },
      { "intent": "feedback", "label": "提出修改意见", "placeholder": "请说明需要调整的参数..." }
    ]
  }
}
```
````

## 2. Posting in Ticket Comments

Post comments using `--content-stdin` with heredoc (remember: `--content-stdin` must be immediately before `<<'EOF'`):

```bash
mopheus ticket comment add <ticket-id> --content-stdin <<'EOF'
## 方案分析完毕

我已完成两种修复策略的对比分析，请确认最终执行路径：

```widget
{
  "v": "1",
  "id": "w_fix_strategy",
  "type": "single-select",
  "label": "选择修复策略",
  "props": {
    "options": [
      { "value": "patch", "label": "原地打补丁 (推荐)", "description": "仅热更新路由中间件", "intent": "positive" },
      { "value": "rollback", "label": "回滚上一版本", "description": "回滚至 v2.4.1 并排查", "intent": "danger" }
    ],
    "default": "patch"
  }
}
```
EOF
```

## 3. Embedding in Ticket Creation

You can also embed widgets when creating a ticket that requires user choices upfront:

```bash
mopheus ticket create --title "生产发布申请" --priority high --description-stdin <<'EOF'
## 发布概述

准备发布版本 v2.5.0，请选择目标维护时段：

```widget
{
  "v": "1",
  "id": "w_release_window",
  "type": "single-select",
  "label": "选择发布时段",
  "props": {
    "options": [
      { "value": "tonight", "label": "今晚 02:00-04:00 (推荐)", "intent": "positive" },
      { "value": "weekend", "label": "周末凌晨 02:00-04:00" }
    ],
    "default": "tonight"
  }
}
```
EOF
```

## 4. Perceiving User Decisions (Receipts)

When the user clicks an action or submits a selection in the Web UI:
1. The widget transitions to an immutable completed state showing who decided and when.
2. Mopheus automatically posts a structured receipt comment in the ticket timeline (containing `action`, `selected`, and any user feedback).
3. The agent reads the confirmed choices by querying the latest comments:

```bash
mopheus ticket comment list <ticket-id> --output json
```

Inspect the newest receipt comments to obtain the user's decision before continuing execution.
