---
name: mopheus-skills
description: "需要使用 mopheus skill 命令创建、导入或管理技能时使用。/ Use when working with mopheus skill commands — covers create, import (URL or local path), export, and file management."
user-invocable: true
allowed-tools: Bash(mopheus *)
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# Managing Skills

Skills are prompt snippets (and optional extra files) that can be assigned to agents to extend their behaviour.

## List / Get

```bash
mopheus skill list --output json
mopheus skill list --search "audit" --page 2 --per-page 25
mopheus skill get <skill-id> --output json
```

## Create

```bash
mopheus skill create \
  --name "My Skill" \
  --description "What this skill does" \
  --label "ops" \
  --content "Instruction text here." \
  --output json
```

`--content` is required. `--description`, `--label` (repeatable), and `--config` are optional. Skill labels use the `skill` resource type and are separate from ticket labels.

## Update / Delete

```bash
mopheus skill update <skill-id> --name "New Name" --content "Updated instructions."
mopheus skill delete <skill-id> --yes
```

`--config` replaces the entire config object (not a merge).

## Manage skill labels

```bash
# List labels attached to a skill
mopheus skill label list <skill-id> --output json

# Add or remove a label by name or UUID
mopheus skill label add <skill-id> "ops"
mopheus skill label remove <skill-id> "ops"

# Keep skill and ticket label dictionaries separate
mopheus label list --target-type skill --output json
mopheus label list --target-type ticket --output json
```

## Import from URL

```bash
mopheus skill import --url https://example.com/skills/my-skill
```

## Import from a local path

The path must be a directory containing `SKILL.md` (or a parent directory that is scanned recursively up to 4 levels deep).

`SKILL.md` must have a frontmatter block with at least a `name:` field:

```markdown
---
name: My Skill
description: What this skill does
---

# My Skill

Instruction content here.
```

```bash
# Import a single skill directory
mopheus skill import --path ./my-skill

# Import all skills under a parent directory
mopheus skill import --path ./skills/

# Re-import and overwrite on name conflict
mopheus skill import --path ./skills/ --update
```

Extra files alongside `SKILL.md` (e.g. `reference.md`, `examples/example.sh`) are bundled automatically (max 128 files, 1 MB each, 8 MB total).

## Export

```bash
# Export specific skills
mopheus skill export <skill-id> [<skill-id>...] --output-dir ./exported/

# Export all skills in the workspace
mopheus skill export --all --output-dir ./exported/
```

Each skill is written to `<output-dir>/<slug>/SKILL.md` plus any attached files.

For complete workspace backups, use `mopheus workspace export <dir>` and `mopheus workspace import <dir>`. The workspace bundle includes skill labels.

## Manage attached files

```bash
# List files attached to a skill
mopheus skill files list <skill-id>

# Add or update a file
mopheus skill files upsert <skill-id> --path reference/guide.md --content-stdin <<'EOF'
File content here.
EOF

# Delete a file
mopheus skill files delete <skill-id> <file-id> --yes
```

## Assign skills to an agent

Use `mopheus agent skills` — see the `mopheus-creating-agents` skill for details.
