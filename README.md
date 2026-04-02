# dbt Model Planner
[![GitHub release](https://img.shields.io/github/v/release/wan-huiyan/dbt-model-planner)](https://github.com/wan-huiyan/dbt-model-planner/releases) [![Claude Code](https://img.shields.io/badge/Claude_Code-skill-orange)](https://claude.com/claude-code) [![license](https://img.shields.io/github/license/wan-huiyan/dbt-model-planner)](LICENSE) [![last commit](https://img.shields.io/github/last-commit/wan-huiyan/dbt-model-planner)](https://github.com/wan-huiyan/dbt-model-planner/commits)

A Claude Code skill that turns schema requirements into production dbt models through a structured, iterative planning process with human review checkpoints.

## What It Does

Takes a target schema (CSV spec, dashboard brief, or requirements doc) and guides you through 7 phases to build a validated dbt model:

1. **Understand Requirements** — parse the spec, determine model grain
2. **Context Gathering** — auto-read project config, mine conventions, explore source models, check with stakeholders
3. **Planning Document** — structured CSV/Excel mapping every target field to source tables with transformation logic
4. **Human Review Loop** — iterate with the team until all fields are confirmed
5. **Schema Validation** — DESCRIBE production tables (3-part check: existence + population + freshness), write investigation queries
6. **Build the Model** — CTE-structured SQL with recommended conventions and a dialect reference table
7. **Verify and QA** — standalone verification query + self-contained QA queries

## Key Features

- **Complexity gate** — lightweight path for simple models (<8 fields), full 7-phase path for complex ones
- **Multi-dialect support** — dialect reference table for Spark/Databricks, Snowflake, BigQuery, and Postgres/Redshift
- **Human review checkpoints** — blank "Review Feedback" column in the planning doc, feedback-to-action mapping table
- **Schema drift detection** — catches incremental model column drift (columns in dbt code but missing from production)
- **URL-extracted join key leak prevention** — detects and fixes silent data quality bugs where URL-extracted IDs (e.g., Snowplow's `event_order_id`) cause dimension fields to fan out across all events instead of just conversion events
- **Event noise filtering** — blacklist-based telemetry filtering for event-level models, with Excel review doc generation
- **Battle-tested tips** — COALESCE across event name fields, taxonomy mismatch handling, deterministic tie-breakers

## Installation

### Claude Code

```bash
# Plugin install (recommended)
/plugin marketplace add wan-huiyan/dbt-model-planner
/plugin install dbt-model-planner@wan-huiyan-dbt-model-planner

# Git clone (always works)
git clone https://github.com/wan-huiyan/dbt-model-planner.git ~/.claude/skills/dbt-model-planner
```

### Cursor (2.4+)

```bash
# Per-project rule (most reliable)
mkdir -p .cursor/rules
# Create .cursor/rules/dbt-model-planner.mdc with SKILL.md content + alwaysApply: true

# npx skills CLI
npx skills add wan-huiyan/dbt-model-planner --global

# Manual global install
git clone https://github.com/wan-huiyan/dbt-model-planner.git ~/.cursor/skills/dbt-model-planner
```

## Usage

The skill triggers automatically when you:
- Provide a target schema and ask to build a dbt model
- Ask "which source tables should I use for this report?"
- Mention "data sources schema", "source mapping", "field mapping"
- Ask to "document a plan" for a dbt model
- "Help me plan a dbt model" / "map these columns to source tables"
- "Create a planning spreadsheet for this model" / "which staging models should I join"
- "Plan the source mappings for this fact table" / "I have a target schema for a new dbt model"
- "Reverse-engineer the transformations needed" / "validate source fields before writing SQL"
- "Build a dbt model from these dashboard requirements" / "plan a dim/fact model from this brief"

## Conventions

The skill includes 7 recommended conventions verified against official dbt docs:

| Convention | Source |
|---|---|
| Refs at top as import CTEs | [dbt style guide](https://docs.getdbt.com/best-practices/how-we-style/2-how-we-style-our-sql) |
| Surrogate key with `_sk` / `_key` naming | [dbt surrogate key guide](https://www.getdbt.com/blog/guide-to-surrogate-key) |
| Group only by grain dimensions | [dbt-labs/corp style guide](https://github.com/dbt-labs/corp/blob/main/dbt_style_guide.md) |
| No unnecessary deduplication | Best practice |
| Linting compliance (check .sqlfluff) | Project-specific |
| ELSE clause in all CASE taxonomy mappings | [SQL best practices](https://www.getgalaxy.io/learn/glossary/case-when-with-null-handling-in-sql) |
| Deterministic tie-breakers for ROW_NUMBER | Best practice |

## Related Skills

- [agent-review-panel](https://github.com/wan-huiyan/agent-review-panel) — Multi-agent adversarial review (used to review this skill)

## Version History

| Version | Changes |
|---------|---------|
| 2.2.0 | Enrich trigger description, add eval suite, add composability metadata |

## Acknowledgements

Trigger accuracy and eval suite improved using [schliff](https://github.com/Zandereins/schliff) — an autonomous skill scoring and improvement framework.

## License

MIT
