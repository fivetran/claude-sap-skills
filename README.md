# Claude SAP Skills

Shared [Claude Code](https://claude.ai/code) custom skills for SAP CDS view work.

## Available Skills

| Skill | Description |
|-------|-------------|
| `/sap-cds` | SAP CDS view expert: 8-phase extraction pipeline, HANA-to-ANSI SQL translation, Fivetran connector, 4 custom ABAP function modules (Z_CDS_SQL_VIEWS, Z_VIEW_DDL, Z_CDS_DEPENDENCIES, Z_VIEW_GET_META) |
| `/sap-compat-view` | SAP compatibility view builder for fivetran/dbt_sap: staging models, column macros, compatibility view SQL, YAML docs |

## Installation

Clone this repo, then symlink or copy the skills into your Claude Code commands directory:

```bash
# Option A: symlink (auto-updates when you pull)
ln -s /path/to/claude-sap-skills/.claude/commands/sap-cds.md ~/.claude/commands/sap-cds.md
ln -s /path/to/claude-sap-skills/.claude/commands/sap-compat-view.md ~/.claude/commands/sap-compat-view.md

# Option B: copy
cp .claude/commands/*.md ~/.claude/commands/
```

Then in any Claude Code session, type `/sap-cds` or `/sap-compat-view` to load the skill.

## Project-Level Usage

If you want the skills available to everyone working in a specific repo, copy the `.claude/commands/` folder into that repo:

```bash
cp -r .claude/commands/ /path/to/your-repo/.claude/commands/
```

Skills in a project's `.claude/commands/` are automatically available to all contributors.
