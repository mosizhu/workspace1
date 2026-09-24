# workspace1 · 工作区导出快照

由 `mopheus workspace export` 导出后推送至本仓库，用于跨工作区重建团队与技能绑定关系。

## 目录结构

| 目录 / 文件 | 内容 | 数量 |
|-------------|------|------|
| `agents/` | 每个 Agent 一份 JSON（含 instructions、runtime、env、visibility、技能引用） | 97 |
| `skills/` | 每个 Skill 一份 `.meta.json`（含 content、labels、config） | 49 |
| `teams/` | 每个 Team 一份 JSON（含 description、instructions、leader、成员 ID 列表） | 21 |
| `jobs/` | 调度/触发 Job（本次工作区无 Job） | 0 |
| `labels.json` | Skill 标签（每个 Skill 可绑定多标签，标签也是绑定关系的一部分） | — |
| `workspace.json` | 工作区元数据（name / slug / description） | — |

## 绑定关系（保留方式）

- **Team → Agent**：通过 `team.members[]` 中的 Agent ID 引用，`team.leaderId` 指向队长 Agent。
- **Agent → Skill**：在 `agent.skillIds[]` 中按 ID 引用；同时 Agent 的 `instructions` 中以 skill 名称调用。
- **Skill → Label**：在每个 `skills/*.meta.json` 的 `labels[]` 中保留 label ID / name 双向信息。
- **跨实体名称**：`workspace.json` 提供 `name` / `slug`，方便导入端再次校验工作区身份。

## 导入方法

```sh
mopheus workspace import <dir>
```

## 生成时间

- 导出命令：`mopheus workspace export ./` (workspace1)
- 提交时间：见 git log

## 注意事项

- 导出文件包含大量中文 instructions / description，是有意的（保留原貌）。
- 导入端需保证目标工作区存在相同 runtime（`agent.runtimeId`），否则 Agent 将降级或失败。
- jobs 目录为空，表示当前工作区没有 Job 自动化。