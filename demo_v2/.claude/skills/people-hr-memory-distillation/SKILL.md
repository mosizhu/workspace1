---
name: people-hr-memory-distillation
description: "Specialty area memory distillation for HR and people-operations tickets: policy interpretation, onboarding/offboarding steps, and escalation rules. / 人力资源与员工事务工单的领域记忆蒸馏：政策解读、入离职流程与升级规则。"
user-invocable: true
capabilities:
  - memory_distillation
features:
  - memory
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Memory Distillation — People & HR Specialty Area

Applies on top of `mopheus-memory-distillation-core` and `mopheus-memory-distillation-general` when the ticket is an HR, people-operations, or employee-relations task. This skill specializes General's extraction rules; it never overrides Core.

Use this specialty area when the ticket involves: policy interpretation questions, onboarding/offboarding procedures, leave/benefits administration, or escalation handling for employee requests.

## Additional Extraction Focus

- **`procedure` (`success_playbook`)** — onboarding/offboarding checklists, benefits-enrollment steps, or escalation paths that were followed successfully. Record as a canonical sequence, not this specific employee's timeline.
- **`preference`** — standing policy interpretations and org-specific rules (e.g. "remote employees get a one-time $500 equipment stipend", "PIP escalations require director sign-off before HRBP notification").
- **`correction` (`human_correction`)** — an HR lead correcting a proposed policy application; capture the wrong interpretation, why it was wrong, and the correct one.
- Recommended `context_tags` keys: `policy_area` (e.g. `leave`, `benefits`, `onboarding`), `region` (policies often vary by jurisdiction), `memory_specialty: hr`.

## Sensitivity

Never store employee names, ID numbers, salary figures, medical/disability details, or other personally identifiable or protected information in `title`/`content`/`context_tags`. Extract only the reusable policy or process knowledge (e.g. "leave-of-absence extensions beyond 12 weeks require legal review" — not any specific employee's case details).

## Out of Scope

Do not use this specialty area for engineering, legal/contract, or sales content. Route unrelated content to General or skip it.
