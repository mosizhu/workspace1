# Task Memory Navigation Index

This is the workspace memory repository index for the current task. Skim primary anchors and matched recommendations first, then navigate to detailed cards via relative paths.

## Primary Anchors
- Team Conventions & Guardrails: refer to `conventions/` for mandatory operational rules.
- User & Environment Preferences: refer to `conventions/preferences/` for operational preferences.

## Knowledge Map (OKF Topology)
- `runbooks/`: Operational SOPs and disaster recovery (`errors/`), pitfalls & incident postmortems (`pitfalls/`), verified execution playbooks (`playbooks/`).
- `facts/`: IT asset topology and configuration baselines (`specs/`).
- `conventions/`: Human corrections (`corrections/`) and user/operational preferences (`preferences/`).
- `docs/`: Ingested system architecture manuals and long-form official reference documentation.

## Top Matched Recommendations
> **Notice**: Showing top 10 prioritized recommendations of 11 locally mounted knowledge cards. Additional 1 cards are cataloged below.

| Type | Title | Relative Path | Occurrence | Last Seen |
| :--- | :---- | :------------ | :--------- | :-------- |
| environment_fact | DM实例172.20.23.54:5236 OS巡检矛盾发现-2026-10-13 | [facts/specs/c11815a2-c850-40be-adcb-f4ac1d6f9638.md](facts/specs/c11815a2-c850-40be-adcb-f4ac1d6f9638.md) | 1 | 2026-10-13T08:22:03+08:00 |
| environment_fact | 告警处理状态: DM实例不可达 #189/#191 | [facts/specs/f3b5235d-0cb4-46bc-a65b-e1717145234e.md](facts/specs/f3b5235d-0cb4-46bc-a65b-e1717145234e.md) | 1 | 2026-10-13T08:13:10+08:00 |
| environment_fact | DM实例172.20.23.54:5236 DMInstanceDown P0告警处理记录 | [facts/specs/095367c2-4d7f-4a10-a09a-4f5768c665cf.md](facts/specs/095367c2-4d7f-4a10-a09a-4f5768c665cf.md) | 1 | 2026-10-13T04:15:56+08:00 |
| environment_fact | Oracle表空间基线-2026-10-12 | [facts/specs/bdea8702-3481-45ec-b3e0-b8a77334ae09.md](facts/specs/bdea8702-3481-45ec-b3e0-b8a77334ae09.md) | 1 | 2026-10-12T22:11:52+08:00 |
| environment_fact | AWR快照配置与采集能力确认 | [facts/specs/6b688258-5e6c-4d66-bbf8-5844d2423468.md](facts/specs/6b688258-5e6c-4d66-bbf8-5844d2423468.md) | 1 | 2026-10-12T22:09:51+08:00 |
| environment_fact | Oracle容量趋势分析结果-2026-10-12 | [facts/specs/0cf1c24f-3061-406a-8dbb-405ac3d0c847.md](facts/specs/0cf1c24f-3061-406a-8dbb-405ac3d0c847.md) | 1 | 2026-10-12T22:09:34+08:00 |
| environment_fact | Oracle 容量预测排除系统表空间 | [facts/specs/0d429418-a0c8-4130-8f05-1cb943e2705e.md](facts/specs/0d429418-a0c8-4130-8f05-1cb943e2705e.md) | 1 | 2026-10-10T20:06:56+08:00 |
| environment_fact | swissQL 巡检连接数异常 20/22 | [facts/specs/0d7a8a35-2cda-4feb-8b0d-3039829b7693.md](facts/specs/0d7a8a35-2cda-4feb-8b0d-3039829b7693.md) | 1 | 2026-10-09T07:04:00+08:00 |
| environment_fact | 2026-10-09 数据库连通性巡检失败汇总 | [facts/specs/385841ff-6359-49ce-84da-3f739e740073.md](facts/specs/385841ff-6359-49ce-84da-3f739e740073.md) | 1 | 2026-10-09T07:03:54+08:00 |
| reference_doc | 授权Oracle_DM数据库网络不可达 | [docs/d291d0d8-acad-4a0d-b37f-350d137c6825.md](docs/d291d0d8-acad-4a0d-b37f-350d137c6825.md) | 1 | 2026-10-13T10:04:00+08:00 |

## Additional Mounted Cards (1)
The following background cards are also mounted in local OKF directories and can be inspected via your preferred file viewing tool or `grep`:

- [Prometheus-Alert-Handler 黑名单](docs/cbaee207-82fb-46a3-9c15-ce775cdf882f.md) (`reference_doc`)

## Usage Guidelines
- **Verify Deeply**: Table summaries are triage hints, not conclusive evidence. Always inspect the full card via your preferred file viewing tool before relying on it.
- **Explore Local Cards**: In addition to the top recommendations, all mounted cards listed above are immediately available in the local directory tree.
- **Dynamic Retrieval**: If this index does not cover what you need, run `mopheus memory retrieve "<query>"` with descriptive natural language, or `mopheus memory get <id> --save` to fetch full docs into `.memory/`.
- **Durable Persistence**: This directory is an ephemeral read-only projection for this task run. Never write memory files directly to disk; use `mopheus memory store` to persist reusable facts into the workspace.
