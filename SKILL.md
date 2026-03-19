---
name: dbt-model-planner
description: |
  Plan and build dbt models from schema requirements, dashboard specs, or reporting briefs.
  Use this skill whenever: (1) someone provides a target schema, CSV spec, or dashboard
  requirements and asks to build a dbt model, (2) someone needs to map required dimensions
  and metrics to existing source tables in a dbt project, (3) someone asks "which source
  tables should I use for this report/dashboard/model?", (4) someone wants to reverse-engineer
  what transformations are needed to produce a target dataset from existing dbt sources,
  (5) someone has a brief or spec document and wants a structured implementation plan before
  writing SQL. Also trigger when the user mentions "data sources schema", "source mapping",
  "transformation logic", "field mapping", or asks to "document a plan" for a dbt model.
  Covers the full lifecycle: codebase exploration → source mapping → planning spreadsheet
  with human review → investigation queries for open questions → schema validation →
  SQL model creation.
author: Claude Code
version: 1.3.0
date: 2026-03-19
---

# dbt Model Planner

Turn schema requirements into production dbt models through a structured, iterative
planning process. This skill ensures nothing gets built before the mapping is validated
— saving rework and catching issues like missing columns, taxonomy mismatches, and
grain misunderstandings early.

## Why This Process Matters

Building a dbt model directly from a requirements doc usually leads to rework because:
- Target fields don't map 1:1 to source columns (they need transformation logic)
- Source tables may have stale schemas (especially incremental models — see below)
- Channel/category taxonomies differ between source systems
- The grain of the target model may not match available source grains
- Previous work may already solve parts of the problem

This skill front-loads all of that discovery into a planning phase with human review
checkpoints, so by the time you write SQL, every field mapping is validated.

## The Process

### Phase 1: Understand the Requirements

Read the target schema document (CSV, spreadsheet, brief, or inline description).
For each target field, extract:
- **Column name** and description
- **Example values** (these reveal data types and expected formats)
- **Implied grain** (is this per-event, per-session, per-user, per-order?)
- **Relationships** between fields (e.g., "transaction_id is only populated for order events")

Determine the **model grain** early — it drives every source selection decision.
Ask the user if ambiguous. Common grains: event-level, session-level, user × time-period,
order-level, aggregated dimension combinations.

### Phase 2: Explore the Codebase

Use subagents for parallel exploration. Investigate:

1. **Existing models in the target domain** — check if similar models already exist
   (e.g., `models/bi/campaign/` for campaign-related work)
2. **Core/staging models** that contain the source fields — read the SQL to understand
   available columns, joins, and transformation logic
3. **Previous work** — the user may point to a separate directory or branch with
   prior implementations. Read these thoroughly — they often contain reusable
   transformation patterns (channel mappings, discount bucketing, tenure groupings, etc.)
4. **Schema files (.yml)** — for documented column descriptions and data types
5. **dbt_project.yml** — for materialization strategies, schema naming conventions,
   and project structure

Key things to note during exploration:
- Which models are `materialized='incremental'` or `microbatch` (these may have
  stale schemas — columns in the SQL may not exist in production)
- Source table grain (one row per what?)
- Join keys between source tables
- Existing transformation logic that can be reused verbatim

### Phase 3: Create the Planning Document

Produce a structured mapping document (CSV or Excel) with these columns:

| Column | Purpose |
|--------|---------|
| **Target Column** | Field name from the requirements |
| **Description** | What the field represents |
| **Example Values** | From the requirements doc |
| **Source Model(s)** | Which dbt model(s) to source from |
| **Source Field(s)** | Specific column name(s) in the source |
| **Transformation Logic** | How to derive the target from source (SQL pseudocode) |
| **Reusable From Previous Work** | Reference to existing code that implements this logic |
| **Questions / Caveats** | Ambiguities, risks, or things that need human confirmation |
| **Review Feedback** | **LEAVE BLANK** — this is for the human reviewer |
| **Status** | CONFIRMED / ACTION NEEDED / PENDING |

**Writing good entries:**
- Be specific about source fields: `core.order.gross_revenue`, not just "order table"
- Include line numbers when referencing reusable code: `fact_campaign_order.sql line 94`
- In Questions/Caveats, be explicit about what you need answered — don't just flag
  vaguely. "Which revenue definition: gross_revenue (incl shipping) or recipe_revenue
  (excl shipping)?" is better than "revenue TBD"
- When multiple source options exist, list them as Option A / Option B with trade-offs

**Format choice:**
- Default to CSV (universally readable, easy to version control)
- If the user prefers Excel/Google Sheets, create a well-formatted spreadsheet
  with headers, column widths, and conditional formatting for Status column
- Markdown table is acceptable for quick inline reviews

Save the planning document and present a summary to the user highlighting:
- Fields that mapped cleanly (high confidence)
- Fields that need decisions (multiple options or ambiguous requirements)
- Fields that are problematic (no clear source, taxonomy mismatch, etc.)

### Phase 4: Human Review Loop

This is the critical step that prevents wasted implementation effort.

Present the plan to the user and ask them to add feedback in the **Review Feedback**
column. Common feedback patterns and how to handle them:

| Feedback Pattern | Action |
|------------------|--------|
| "looks good" / "agree" | Mark as CONFIRMED, no changes needed |
| "ignore" | Remove from scope or mark as out-of-scope |
| "check if X exists" / "can you verify Y?" | Write an investigation query (Phase 5) |
| "use X instead of Y" | Update source/logic, mark as CONFIRMED |
| "I'm not sure about this" | Flag for investigation, keep as ACTION NEEDED |
| "can we add a fallback?" | Update transformation logic with COALESCE pattern |
| Correction to your assumption | Update the plan, check if correction affects other rows |

After incorporating feedback, update the planning document and re-present any
fields that changed status. Iterate until all fields are CONFIRMED or explicitly
out of scope.

### Phase 5: Schema Validation & Investigation Queries

Before writing the model SQL, validate that source fields actually exist in
production. This catches a common issue with incremental dbt models where columns
are defined in SQL but not materialised.

**Step 1: Generate DESCRIBE queries for all source tables**

```sql
-- Validate source schema for each table referenced in the plan
DESCRIBE main.<schema>.<table_name>;
```

Cross-reference the DESCRIBE output against every source field in the plan.
Flag any field that exists in the dbt SQL code but NOT in the production table.

**Step 2: Write investigation queries for open questions**

For each ACTION NEEDED item, write a targeted SQL query. Common patterns:

- **Coverage check**: `SELECT COUNT(*), COUNT(field) FROM table` — what % is NULL?
- **Value distribution**: `SELECT field, COUNT(*) GROUP BY field ORDER BY 2 DESC`
- **Cross-table join validation**: Does joining A to B on key X produce expected results?
- **Filter impact**: How many rows does a WHERE clause exclude?

Save all queries in a single SQL file with:
- Comment header explaining the purpose
- Context linking back to the planning doc row
- Status marker (PENDING / DONE)

**Important: When the user runs queries manually (no MCP/direct DB connection),
combine related queries into a single UNION ALL result set** with a discriminator
column (e.g., `query_id` or `field_source`). One query = one download = one file.
Only split into separate queries when schemas are incompatible or cost is prohibitive.

**Step 3: Incorporate results**

After the user runs the queries (or you run them via MCP/tool):
- Update the planning document with findings
- Resolve ACTION NEEDED items → CONFIRMED or flag new issues
- Add a **Source Validated** column to the planning doc confirming each field
  exists in production (with data type from DESCRIBE output)
- If a source field doesn't exist in production, find alternatives:
  - Use upstream raw fields and replicate the transformation logic
  - Check if a related field exists with similar semantics
  - Flag as a blocker requiring a dbt full-refresh

### Phase 6: Build the dbt Model

Once all fields are CONFIRMED, write the SQL model following the project's conventions.

**Structure the model as CTEs:**

```sql
{{ config(materialized='table', tags=['domain_tag']) }}

-- Source CTEs
with source_a as (
    select * from {{ ref('source_model_a') }}
),

-- Transformation CTEs (one per logical step)
transformed as (
    select
        -- Direct mappings
        source_a.field_1,
        -- Computed fields (with inline comments for non-obvious logic)
        case
            when source_a.mkt_medium in ('paidsocial', 'paid') then 'Paid Social'
            ...
        end as channel,
        -- Joined fields
        source_b.revenue
    from source_a
    left join source_b on ...
),

final as (
    select * from transformed
)

select * from final
```

**Universal conventions (apply to every dbt project):**

1. **Refs at top:** List all `{{ ref('...') }}` source CTEs at the top of the file
   with `select * from {{ ref('...') }}` so data lineage is immediately visible.
   Keep all transformation logic in later CTEs — never mix source selection with
   business logic.

2. **Surrogate primary key:** Add a primary key using
   `{{ dbt_utils.generate_surrogate_key([...]) }}` composed of all grain-defining
   dimension columns. Use `CTE_name.column` format in the key definition
   (e.g., `'events.event_id'`). Name it `[model_name]_key`. In the companion YAML,
   test with `dbt_utils.unique_combination_of_columns` on the key column, plus
   column-level `unique` and `not_null` tests.

3. **Group only by grain dimensions:** In GROUP BY, include only columns that define
   the output grain. For dimensions that are 1:1 with the grain (e.g., period fields
   derived from menu_week), use `first()` in SELECT instead of adding to GROUP BY.
   This makes the intended grain explicit and prevents accidental grain changes.

4. **No unnecessary deduplication:** Don't add `qualify row_number() over (...)` or
   `distinct` on CTEs sourcing from upstream models that already guarantee uniqueness.
   Check the upstream model first — if it's already deduplicated, the extra logic adds
   cost and obscures intent.

5. **Linting compliance:** Follow sqlfluff rules: single space before `as`, explicit
   `as` for all aliases, one select target per line, consistent indentation, no SQL
   reserved keywords as column identifiers.

6. **Full CTE names in joins:** Reference columns as `cte_name.column_name` in all
   joins and SELECT clauses (e.g., `committed_orders.gross_revenue`, not
   `co.gross_revenue`). Don't alias CTEs with short names — readability matters more
   than brevity.

**Project-specific conventions (check these per project):**
- Match the project's existing patterns (check dbt_project.yml for materialization,
  tags, schema conventions)
- Use `{{ ref() }}` for all model references
- Add `current_timestamp() as _row_updated_at_` if the project convention includes it
- Write a companion .yml schema file with column descriptions and tests
- Check for project-specific model naming (e.g., period vs dim_calendar), alias
  conventions (e.g., shipping_region → region), and filter patterns (rolling windows
  vs hardcoded dates)

**After writing the model:**

### Phase 7: Verify and QA

Don't declare done after writing the SQL. Verification catches bugs that look
correct on paper but fail with real data.

**Step 1: Create a standalone Databricks verification query**
- Replace `{{ ref('...') }}` with fully-qualified 3-part table names
- Replace `{{ dbt_utils.generate_surrogate_key(...) }}` with `md5(cast(...))`
- Add a date filter (e.g., `WHERE collector_tstamp >= '2026-01-01'`) to avoid
  full table scans
- Add `LIMIT 100` for quick syntax validation

**Step 2: Inspect the output for data quality issues**
Common bugs that only surface with real data:
- **NULL fields you expected populated** — check if the source field uses a
  different name, or if COALESCE fallbacks are needed (e.g., se_action is NULL
  for page_view events, but event_action or event_name has the value)
- **Fields populated on rows they shouldn't be** — especially when joining on
  URL-extracted IDs (see "URL-Extracted Join Keys" below)
- **Unexpected values** — enum fields with values not in your CASE statement

**Step 3: Write standalone QA queries**
Each QA query should be self-contained (no dependency on materialised output).
Key checks:
- NULL rates per column
- Value distribution for computed fields (channel groupings, discount segments)
- Join fan-out validation (count with vs without order fields)
- Single-entity journey trace (visual sanity check of one user's event sequence)

## Tips for Common Challenges

### Taxonomy Mismatches
When source systems use different category names (e.g., UTM tags vs promo channel
categories), build the mapping CASE statement explicitly in the plan. Don't defer
this to implementation — taxonomy decisions need human review.

### Incremental Model Column Drift
dbt incremental models (`materialized='incremental'` or `microbatch`) silently
omit new columns added to the SQL after initial materialization. The column
exists in the code but not in the production table. This cascades: if model A
is missing a column, every downstream incremental model (B, C) that selects
from A will also be missing it.

**Diagnosis:**
1. Run `DESCRIBE <table>` to confirm the column is missing from production
2. `grep` the dbt SQL to confirm the column is defined in code
3. Check `config(materialized=...)` — if incremental, this is likely the cause
4. Check git log: was the column added after the last `--full-refresh`?

**Resolution (pick one):**
- **Full refresh** (preferred): `dbt run --full-refresh --select <model>`.
  Must refresh the entire chain from root to leaves in dependency order.
- **Replicate upstream logic** (when you can't refresh): If the missing column
  is computed from raw fields that DO exist, replicate the transformation in
  your model. E.g., if `utm_classification` (computed CASE) is missing but
  `mkt_source` and `mkt_medium` exist, build the CASE logic inline.
- **Preventive config**: `on_schema_change='sync_all_columns'` in the model
  config. But this only applies to future runs — doesn't retroactively add
  columns from before the config was set.

**Key gotcha:** `on_schema_change='sync_all_columns'` may already be a project
default in `dbt_project.yml`, but it only takes effect for runs AFTER the config
was active. Tables materialized before it was set still have the old schema.

### Multiple Source Options
When a field could come from multiple sources (e.g., geo_country from IP vs
shipping_country from orders), document the trade-offs in the plan and recommend
a COALESCE fallback pattern. Let the human decide priority order.

### Grain Mismatches
If the target grain is finer than the source (e.g., event-level target but
session-level source), you need to join back to the finer-grained table.
If coarser, decide on the aggregation strategy (SUM, COUNT DISTINCT, FIRST, etc.)
and document it in the plan.

### URL-Extracted Join Keys (Critical for Event Data)
Some fields are extracted from URLs via regex (e.g., Snowplow's `event_order_id`
from `/menu/{order_id}`). These appear on ALL events in the URL flow, not just the
target event. If you LEFT JOIN a dimension table on such a field, the dimension
attributes leak onto every event in the session. Fix: create a gating CTE that
identifies the specific event (e.g., last event in session with conversion flag),
then join through that CTE. Gate ALL dimension fields behind `CASE WHEN gate.id
IS NOT NULL`.
See also: `snowplow-event-order-id-leak` skill.

### COALESCE Across Multiple Event Name Fields
Event tracking systems often have multiple name fields with different coverage.
For example, Snowplow has `se_action` (structured events), `event_action` (app
events), and `event_name` (native type like page_view). Use COALESCE to unify,
but audit each field separately first — they may contain different types of events
(user interactions vs telemetry). Consider filtering out high-volume telemetry
events that add noise without attribution value.

### Event Noise Filtering (for Event-Level Models)

When building event-level models (MTA, funnel analysis, behavioural analytics),
the raw event table often contains a mix of genuine user interactions and
system telemetry. Telemetry events (performance metrics, A/B test bucketing,
app lifecycle, heartbeats) can dominate volume without providing attribution
signal. Follow this process to filter them:

**Step 1: Audit all event name sources separately**

If the event table has multiple name fields (e.g., se_action, event_action,
event_name), query each one independently to understand what each contributes.
Use a single UNION ALL query with a `field_source` discriminator column so the
user only has to run and download one result:

```sql
SELECT 'se_action' AS field_source, se_action AS event_value,
       COUNT(*) AS event_count, COUNT(DISTINCT session_id) AS session_count
FROM events WHERE se_action IS NOT NULL GROUP BY se_action
UNION ALL
SELECT 'event_action', event_action, COUNT(*), COUNT(DISTINCT session_id)
FROM events WHERE se_action IS NULL AND event_action IS NOT NULL GROUP BY event_action
-- etc.
```

**Step 2: Classify events and propose blacklist vs keep**

Categorise each event as user engagement or telemetry. Common telemetry patterns:
- Performance timing: `*_load_time`, `*_time_to_usable`, `*FetchTime`
- System events: `environment_data_captured`, `*_experiment_*`, `basket_limit`
- Heartbeats: `page_ping`, `application_foreground/background`
- Auto-UI: `carousel_slide_change`, `sticky_cta_state_change`

**Blacklist (not whitelist)** is usually more practical — event taxonomies have
hundreds of values, making whitelisting impractical to maintain. A blacklist of
~20 high-volume noise events typically removes 15-20% of total volume.

**Step 3: Create a review document for stakeholder sign-off**

Produce an Excel/spreadsheet with:
- **Blacklist tab**: Event name, source field, volumes, category, rationale
- **Keep tab**: ALL remaining events (not just examples) with auto-classified
  funnel stage and a blank Review Feedback column for the team to annotate
- **Implementation tab**: Options (CTE / seed table / macro) with trade-offs
- **Legend tab**: Funnel stage classification guide

Auto-classify funnel stages by keyword matching (Conversion, Consideration,
Awareness, Marketing, Retention, Account, Support, etc.) and colour-code them.
Enable autofilter so reviewers can filter by source field or funnel stage.

The keep tab must list ALL events — not just examples — so the team can do a
thorough review and flag anything they disagree with. This is a human review
checkpoint, same principle as Phase 4.

**Step 4: Implement as dbt seed table (recommended)**

After sign-off, create a CSV seed file (e.g., `data/mta_event_blacklist.csv`)
and anti-join in the model. This is more maintainable than a CTE blacklist
because non-engineers can edit the CSV, it's version controlled, and it's
reusable across models.
