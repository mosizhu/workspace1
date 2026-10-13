---
name: sales-customer-memory-distillation
description: "Specialty area memory distillation for sales and customer-facing tickets: objection handling, pricing rules, and escalation playbooks. / 销售与客户工单的领域记忆蒸馏：异议处理、定价规则与升级方案。"
user-invocable: true
capabilities:
  - memory_distillation
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Memory Distillation — Sales & Customer Specialty Area

Applies on top of `mopheus-memory-distillation-core` and `mopheus-memory-distillation-general` when the ticket is a sales, customer-success, or account-management task. This skill specializes General's extraction rules; it never overrides Core.

Use this specialty area when the ticket involves: pricing/discount decisions, objection handling, renewal negotiation, or customer escalation resolution.

## Additional Extraction Focus

- **`procedure` (`success_playbook`)** — an objection-handling or escalation sequence that led to a successful outcome (renewal, upsell, resolved complaint). Record as a reusable pattern, not this specific customer's story.
- **`preference`** — standing pricing/discount rules and approval thresholds (e.g. "discounts above 20% require VP approval", "enterprise tier requires a signed DPA before contract execution").
- **`correction` (`human_correction`)** — a sales lead correcting a proposed pricing or messaging approach; capture the rejected approach, why it was wrong, and the accepted one.
- Recommended `context_tags` keys: `deal_stage`, `product_tier`, `memory_specialty: sales`.

## Sensitivity

Never store customer names, account IDs, deal values, or other confidential commercial terms in `title`/`content`/`context_tags`. Extract only the reusable rule or pattern (e.g. "multi-year commitments unlock an additional 10% discount tier" — not "Acme Corp got a 30% discount for a 3-year deal").

## Out of Scope

Do not use this specialty area for engineering, legal/contract, or HR content. Route unrelated content to General or skip it.
