---
name: dbt-model-planner
description: |
  Plan and build NEW dbt models from schema requirements, dashboard specs, or reporting briefs.
  Trigger when: (1) user provides a target schema/CSV spec/dashboard requirements and asks to
  build a dbt model, (2) user needs to map dimensions and metrics to existing dbt source tables,
  (3) user asks "which source tables should I use for this report/dashboard/model?", (4) user
  wants to reverse-engineer transformations to produce a target dataset from dbt sources,
  (5) user has a brief/spec and wants a structured plan before writing SQL.
  Trigger phrases: "plan a dbt model", "map columns to source tables", "create a planning
  spreadsheet", "which staging models should I join", "plan source mappings for this fact table",
  "I have a target schema for a new dbt model", "reverse-engineer the transformations needed",
  "validate source fields before writing SQL", "build a dbt model from dashboard requirements",
  "plan a dim/fact model from this brief", "data sources schema", "source mapping",
  "transformation logic for a dbt model", "field mapping for a dbt model",
  "document a plan for a dbt model".
  Do NOT trigger for: SQL query optimization or performance tuning, data analysis or insight
  generation, ETL/ELT pipeline orchestration (Airflow, Prefect, Dagster), dbt CI/CD or GitHub
  Actions setup, dbt project migration between warehouses, reviewing or debugging existing dbt
  model SQL, database schema design for applications or microservices, dbt unit testing or
  test-writing, Python data cleaning or transformation scripts, Looker/Tableau/BI tool
  definitions, churn analysis or customer analytics, ORM data modeling (Prisma, SQLAlchemy),
  creating new warehouse tables without dbt context.
author: Claude Code
version: 2.3.0
date: 2026-03-25
consumes_from: ["target schema (CSV/spreadsheet/brief)", "dbt project codebase"]
hands_off_to: ["dbt test frameworks", "dbt-incremental-missing-columns", "snowplow-event-order-id-leak"]
output_contract: ["planning spreadsheet (CSV/Excel)", "investigation SQL file", "dbt model .sql + .yml"]
---

# dbt Model Planner

Turn schema requirements into production dbt models through structured, iterative planning.
Nothing gets built before mappings are validated -- catching missing columns, taxonomy mismatches,
and grain misunderstandings early.

## Composability

**Input:** Target schema (CSV, spreadsheet, brief, inline) + dbt project codebase.

**Output:** Planning spreadsheet, validation SQL, dbt model `.sql` + `.yml`.

**Dependencies:** Requires git (for codebase exploration) and a dbt project with `dbt_project.yml`. Supported versions: dbt-core >= 1.0, dbt-utils >= 0.8. Works with Databricks/Spark v3+, Snowflake, BigQuery, Postgres, and Redshift dialects.

**Error handling:** Missing source fields: flag as blockers, suggest alternatives. Wrong grain: loop to Phase 1. No codebase access: defer exploration.

**Idempotency:** Safe to re-run. Creates new files only (Phase 6). Scoped to `dbt-model-planner` namespace.

**Handoff:** For existing dbt model debugging, then use `dbt-incremental-missing-columns` instead. For Snowplow fan-out, then use `snowplow-event-order-id-leak`. After model creation, hand off to dbt test frameworks. For ETL pipelines, use Airflow/Prefect.

## Complexity Gate

| Criteria | Lightweight (skip to Phase 6) | Full (all 7 phases) |
|----------|-------------------------------|---------------------|
| Fields | < 8 target fields | 8+ fields or any ambiguity |
| Sources | All known and obvious | Multiple systems, different naming |
| Grain | Clear and confirmed | Needs discussion |
| Complexity | No taxonomy mappings needed | Complex joins, reusable prior work |

## The Process

### Phase 1: Understand the Requirements

Read the target schema. For each field, extract:
- **Column name** and description
- **Example values** (reveal data types and formats)
- **Implied grain** (per-event, per-session, per-user, per-order?)
- **Relationships** (e.g., "transaction_id only populated for order events")

Determine the **model grain** early -- it drives every source selection. Ask if ambiguous.

### Phase 2: Context Gathering & Codebase Exploration

#### Step 1: Project Configuration

Auto-read: `dbt_project.yml`, `packages.yml`, `.sqlfluff`, `.gitignore`. Check `profiles.yml` (often at `~/.dbt/`; absent in dbt Cloud).

Extract: warehouse dialect, materialization defaults, schema naming, catalog/database, project vars (e.g., `surrogate_key_treat_nulls_as_empty_strings`), package versions.

#### Step 2: Convention Mining

Read convention docs (CLAUDE.md, CONTRIBUTING.md), memory files, PR templates, recent merged PRs. Compile a **Convention Brief**: branch naming, SQL style, model naming (fact_*, dim_*, stg_*), testing conventions, PR process, "never do this" rules.

#### Step 3: Source Model Exploration

Investigate (use subagents for parallel exploration when available):
1. **Existing models** in the target domain
2. **Core/staging models** with source fields -- read SQL for columns, joins, logic
3. **Previous work** -- reusable transformation patterns (channel mappings, bucketing, etc.)
4. **Schema files (.yml)** -- column descriptions and data types

#### Step 4: Source Lineage & Risk Flags

For each source model, check:

| Check | Why |
|-------|-----|
| Materialization (`incremental`/`microbatch`?) | Schema drift risk -- SQL columns may not exist in production |
| Grain (one row per what?) | Document explicitly |
| Join keys | What links models together |
| Freshness (`_row_updated_at_`, `updated_at`, etc.) | Stale data detection |

#### Step 5: Stakeholder Alignment

Prompt the user to confirm with the end consumer:
1. What is the grain? (one row per what?)
2. What are the goal/conversion events?
3. How will they use the output? (screenshot of downstream format is ideal)
4. Will they transform the output further or consume it directly?

If grain is wrong later, loop back to Phase 1 -- do not patch Phase 3.

### Phase 3: Create the Planning Document

Produce a CSV/Excel mapping with these columns:

| Column | Purpose |
|--------|---------|
| **Target Column** | Field name from requirements |
| **Description** | What the field represents |
| **Example Values** | From the requirements doc |
| **Source Model(s)** | Which dbt model(s) to source from |
| **Source Field(s)** | Specific column name(s) (e.g., `core.order.gross_revenue`) |
| **Transformation Logic** | SQL pseudocode to derive target from source |
| **Reusable From Previous Work** | Reference with line numbers (e.g., `fact_campaign_order.sql:94`) |
| **Questions / Caveats** | Specific questions, not vague flags |
| **Review Feedback** | **LEAVE BLANK** for human reviewer |
| **Status** | CONFIRMED / ACTION NEEDED / PENDING |

**Rules:** Be specific about source fields. Ask explicit questions ("Which revenue: gross_revenue incl shipping or recipe_revenue excl shipping?"). List Option A / Option B with trade-offs when multiple sources exist.

**Format:** Default CSV. Excel/Sheets if user prefers. Markdown for quick inline reviews.

Present summary: high-confidence mappings, fields needing decisions, problematic fields.

### Phase 4: Human Review Loop

Present the plan. Handle feedback:

| Feedback | Action |
|----------|--------|
| "looks good" | Mark CONFIRMED |
| "ignore" | Mark out-of-scope |
| "check if X exists" | Write investigation query (Phase 5) |
| "use X instead of Y" | Update source/logic, mark CONFIRMED |
| "not sure" | Keep ACTION NEEDED |
| "add a fallback" | Add COALESCE pattern |
| Correction to assumption | Update plan, check cascading effects |
| Grain/scope change | **Loop to Phase 1** |

Iterate until all fields are CONFIRMED or out of scope.

### Phase 5: Schema Validation & Investigation Queries

**Step 1: 3-part source validation** (for each source table)

```sql
-- 1. Column existence (dialect-specific — see references/dialect-patterns.md)
-- 2. Column population:
SELECT COUNT(*) AS total_rows, COUNT(<field>) AS non_null,
    ROUND(COUNT(<field>) * 100.0 / COUNT(*), 1) AS pct_populated
FROM <table> WHERE <date_col> >= '<recent_date>';
-- 3. Data freshness:
SELECT MAX(<timestamp_col>) AS last_updated, COUNT(*) AS total_rows FROM <table>;
```

All three must pass. Flag fields that exist in dbt SQL but not in production.

**Step 2: Investigation queries** for ACTION NEEDED items

| Pattern | Query |
|---------|-------|
| Coverage | `SELECT COUNT(*), COUNT(field), COUNT(DISTINCT field) FROM table` |
| Distribution | `SELECT field, COUNT(*) GROUP BY field ORDER BY 2 DESC` |
| Fan-out | Compare `COUNT(*)` before/after LEFT JOIN |
| Filter impact | Count rows excluded by WHERE clause |

When user runs queries manually: combine into UNION ALL with `query_id` discriminator. One query = one download.

**Step 3: Incorporate results** -- update planning doc, resolve ACTION NEEDED items, add Source Validated column. Missing fields: try upstream raw fields, related fields, or flag as blocker requiring full-refresh.

### Phase 6: Build the dbt Model

Structure as CTEs with refs at top:

```sql
{{ config(materialized='table', tags=['domain_tag']) }}

with source_a as (
    select * from {{ ref('source_model_a') }}
),
transformed as (
    select
        source_a.field_1,
        case when source_a.category in ('a', 'b') then 'Group 1'
            else 'Unmapped'
        end as category_group
    from source_a
    left join source_b on source_a.key = source_b.key
),
final as (
    select * from transformed
)
select * from final
```

**Conventions checklist:**

| # | Rule |
|---|------|
| 1 | **Refs at top**: All `{{ ref() }}` source CTEs at top with `select *`. Transformation logic in later CTEs only |
| 2 | **Surrogate PK**: `generate_surrogate_key()` on grain dimensions. Test: `unique` + `not_null` on key, `unique_combination_of_columns` on component dimensions |
| 3 | **GROUP BY grain only**: Use `ANY_VALUE()`/`first()`/`MIN()` for 1:1 dimensions (see `references/dialect-patterns.md`) |
| 4 | **No unnecessary dedup**: Check upstream uniqueness before adding `QUALIFY`/`DISTINCT` |
| 5 | **Linting**: Follow project `.sqlfluff` rules |
| 6 | **CASE needs ELSE**: Always `ELSE 'Unmapped'` or `ELSE original_field` for taxonomy mappings |
| 7 | **Deterministic tie-breakers**: `ROW_NUMBER()` ORDER BY needs 2+ columns, second being a unique ID |

See `references/dialect-patterns.md` for warehouse-specific SQL syntax.

**Project-specific**: Match existing patterns for materialization, tags, schema naming. Use `{{ ref() }}` always. Add `_row_updated_at_` if project convention. Write companion `.yml` with column descriptions and tests.

### Phase 7: Verify and QA

**Step 1: Standalone verification query**
- Replace `{{ ref() }}` with fully-qualified table names
- Replace `generate_surrogate_key()` with dialect-specific hash
- Add date filter + `LIMIT 100`

**Step 2: Data quality inspection**

| Bug | Check |
|-----|-------|
| NULL fields expected populated | Source field name mismatch? Need COALESCE? |
| Fields on wrong rows | URL-extracted ID fan-out? (see below) |
| Unexpected enum values | CASE without ELSE? |
| Row count inflation | `COUNT(*)` before vs after each LEFT JOIN |

**Step 3: QA queries** (self-contained, no dependency on materialized output)
- NULL rates per column
- Value distribution for computed fields
- Join fan-out validation
- Data freshness check
- Single-entity journey trace

## Common Challenges

### Taxonomy Mismatches
Build CASE mapping explicitly in the plan. Always include ELSE. Do not defer to implementation -- taxonomy decisions need human review.

### Incremental Model Column Drift
Incremental/microbatch models silently omit columns added after initial materialization. This cascades downstream.

**Diagnosis:** Run schema inspection -> grep dbt SQL -> check `materialized` config -> check git log for column addition date.

**Resolution:** (1) Full refresh in dependency order (preferred), (2) Replicate upstream logic from raw fields, (3) Preventive: `on_schema_change='sync_all_columns'` (only applies to future runs).

### Multiple Source Options
Document trade-offs in the plan. Recommend COALESCE fallback. Let the human decide priority order.

### Grain Mismatches
Finer target than source: join back to fine-grained table. Coarser: define aggregation strategy. Multiple grains in requirements: split into separate models. Split if >200 lines or mixed detail/aggregate grains.

### URL-Extracted Join Keys

URL-extracted IDs (e.g., order_id from `/menu/{8-digit-id}`) appear on ALL page views in the order flow, not just conversions. Joining order tables on these IDs silently fans out order dimensions to non-conversion events.

**Fix -- 3-step gating:**

```sql
-- 1. Identify actual conversion event
order_events as (
    select event_id, event_order_id from events
    where event_order_id is not null and order_placed_in_session_flag = 1
    qualify row_number() over (
        partition by session_id, event_order_id
        order by collector_tstamp desc, event_id desc
    ) = 1
),
-- 2. Join through gating CTE, not directly on URL field
-- 3. Gate ALL order fields: case when order_events.event_id is not null then ...
```

**Verify:** `COUNT(transaction_id)` should approximate `COUNT(DISTINCT transaction_id)`.

### COALESCE Across Event Name Fields
Multiple name fields (e.g., `se_action`, `event_action`, `event_name`) have different coverage. COALESCE to unify, but audit each separately first. Add `event_name_source` companion column to preserve provenance.

### Event Noise Filtering

For event-level models, raw tables mix user interactions with telemetry. Follow this process:

1. **Audit** each event name field separately (UNION ALL with `field_source` discriminator)
2. **Classify** as engagement vs telemetry. Use a **blacklist** (not whitelist) -- ~20 high-volume noise events typically removes 15-20% volume
3. **Review doc** with Blacklist/Keep/Implementation/Legend tabs for stakeholder sign-off. Keep tab must list ALL events
4. **Implement** as dbt seed table (`data/mta_event_blacklist.csv`) with anti-join. If CI uses `dbt run` (not `dbt build`), ensure `dbt seed` runs first
