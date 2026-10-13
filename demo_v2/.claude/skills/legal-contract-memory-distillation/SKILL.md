---
name: legal-contract-memory-distillation
description: "Specialty area memory distillation for legal and contract review tickets: clause positions, negotiated terms, and approval rules. / 法务与合同审查工单的领域记忆蒸馏：条款立场、谈判结果与审批规则。"
user-invocable: true
capabilities:
  - memory_distillation
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Memory Distillation — Legal & Contract Specialty Area

Applies on top of `mopheus-memory-distillation-core` and `mopheus-memory-distillation-general` when the ticket is a legal, contract drafting, or contract review task. This skill specializes General's extraction rules; it never overrides Core.

Use this specialty area when the ticket involves: contract clause negotiation, legal risk review, compliance requirements, approval thresholds, or standard-terms decisions.

## Additional Extraction Focus

- **`correction` (`human_correction`)** — a legal/compliance reviewer overriding a proposed clause or position is almost always this type; capture the rejected position, the reason, and the accepted position.
- **`preference`** — standing negotiation positions and fallback terms (e.g. "our floor for liability cap is 1x annual contract value", "always require a 30-day termination-for-convenience clause").
- **`procedure` (`success_playbook`)** — a review or negotiation sequence that reached signature/approval; record it as steps, not as the specific deal's facts.
- Store decisions (e.g. "accepted clause X with fallback Y") with `--context-tags '{"memory_subtype":"decision"}'` under `procedure` or `correction`, not as a new type.
- Recommended `context_tags` keys: `contract_type`, `clause_type`, `jurisdiction`, `memory_specialty: legal`.

## Sensitivity

Never store counterparty names, deal values, specific contract IDs, or other confidential commercial terms in `title`/`content`/`context_tags`. Extract the reusable position or rule only (e.g. "liability cap negotiations for SaaS contracts default to 1x ACV" — not "Acme Corp agreed to a $2M cap").

## Out of Scope

Do not use this specialty area for engineering, HR, or sales content. Route unrelated content to General or skip it.
