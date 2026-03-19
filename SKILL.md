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
version: 2.1.0
date: 2026-03-19
---

# dbt Model Planner

Turn schema requirements into production dbt models through a structured, iterative
planning process. This skill ensures nothing gets built before the mapping is validated
— saving rework and catching issues like missing columns, taxonomy mismatches, and
grain misunderstandings early.

## Complexity Gate

Not every model needs the full 7-phase treatment. Before starting, assess complexity:

**Lightweight path** (skip to Phase 6 with a checklist):
- Fewer than 8 target fields
- All source tables are known and obvious
- Grain is clear and confirmed with the consumer
- No taxonomy mappings or complex joins needed

**Full path** (all 7 phases):
- 8+ target fields, or any ambiguity in source mapping
- Multiple source systems with different naming conventions
- Grain needs discussion or could be misunderstood
- Previous work exists that may be reusable

## Why This Process Matters

Building a dbt model directly from a requirements doc usually leads to rework because:
- Target fields don't map 1:1 to source columns (they need transformation logic)
- Source tables may have stale schemas (especially incremental models — see below)
- Channel/category taxonomies differ between source systems
- The grain of the target model may not match available source grains
- Previous work may already solve parts of the problem

This skill front-loads discovery into a planning phase with human review checkpoints,
so by the time you write SQL, every field mapping is validated.

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

### Phase 2: Context Gathering & Codebase Exploration

Before exploring source models, systematically gather project context that
will shape every decision downstream. Skipping this leads to models that
compile but violate project conventions, miss existing patterns, or use
stale schemas.

#### Step 1: Project Configuration Discovery

Auto-read these files:

```
dbt_project.yml     → materialization defaults, schema naming, tags, vars
packages.yml        → installed packages (dbt_utils, elementary, etc.)
.sqlfluff           → linting rules and dialect
.gitignore          → what file types are excluded from version control
```

Also check for `profiles.yml` in the project root, but note it often lives
in `~/.dbt/profiles.yml` instead (and doesn't exist at all in dbt Cloud).
The `profile:` key in `dbt_project.yml` tells you the warehouse type.

Extract and note:
- **Warehouse dialect** (Databricks/Spark, Snowflake, BigQuery, Redshift, Postgres)
  — this affects SQL syntax throughout the skill
- Default materialization strategy and incremental strategy
- Schema naming convention (e.g., `production_served__<layer>`)
- Catalog/database name
- Any project-level vars that affect model behavior (e.g.,
  `surrogate_key_treat_nulls_as_empty_strings`)
- Package versions (especially dbt_utils — API differs across versions)

#### Step 2: Convention Mining

Search for project-specific conventions that override generic best practices.
These are the rules that get PRs rejected if violated:

```
1. Read project convention docs (CLAUDE.md, CONTRIBUTING.md, DEVELOPMENT.md)
2. Read project memory files (feedback type — these are past PR corrections)
3. Check .github/pull_request_template.md for review checklist items
4. Check recent merged PRs (gh pr list --state merged, or git log) for
   naming/style patterns
```

Compile a **Convention Brief** noting:
- Branch naming convention (e.g., `{TICKET-ID}/{description}`)
- SQL style rules beyond sqlfluff (CTE naming, alias conventions, etc.)
- Model naming patterns (fact_*, dim_*, stg_*, bi_*)
- Testing conventions (what tests are expected on which columns)
- PR process (base branch, review requirements, CI checks)
- Any "never do this" rules from feedback memories

#### Step 3: Source Model Exploration

Use subagents for parallel exploration (or explore sequentially if subagents
are not available). Investigate:

1. **Existing models in the target domain** — check if similar models already exist
   (e.g., `models/bi/campaign/` for campaign-related work)
2. **Core/staging models** that contain the source fields — read the SQL to understand
   available columns, joins, and transformation logic
3. **Previous work** — the user may point to a separate directory or branch with
   prior implementations. Read these thoroughly — they often contain reusable
   transformation patterns (channel mappings, discount bucketing, tenure groupings, etc.)
4. **Schema files (.yml)** — for documented column descriptions and data types

#### Step 4: Source Lineage & Risk Flags

For each source model identified in Step 3, check:

- **Materialization type**: Is it `incremental` or `microbatch`? If yes, flag it
  as a schema drift risk — columns in the SQL may not exist in production.
  Add to the validation checklist for Phase 5.
- **Grain**: What does one row represent? Document explicitly (e.g., "one row per
  event per user" vs "one row per session").
- **Join keys**: What fields link this model to others?
- **Freshness**: When was the model last updated? Check metadata timestamp columns
  (common names: `_row_updated_at_`, `_loaded_at`, `updated_at`, `dbt_updated_at`).

#### Step 5: Stakeholder Alignment Check

Before proceeding to the planning doc, ask the user to confirm with the end
consumer (this requires human action — prompt the user to do it):

1. **What is the grain?** (one row per what?)
2. **What are the goal/conversion events?** (if applicable)
3. **How will they use the output?** A screenshot or example of their
   downstream input format is worth more than a 20-column spec.
4. **What does the end consumer's workflow look like?** Will they transform the
   output further, or consume it directly in a dashboard/model?

This 5-minute conversation prevents the most expensive rework — building at
the wrong grain or including fields the consumer doesn't need. If the grain
assumption turns out to be wrong later (e.g., event-level vs session-level),
loop back to Phase 1 — don't try to patch Phase 3.

Compile all findings into a **Context Brief** that informs the planning document.

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
| Fundamental grain/scope change | **Loop back to Phase 1** — don't patch, replant |

After incorporating feedback, update the planning document and re-present any
fields that changed status. Iterate until all fields are CONFIRMED or explicitly
out of scope.

### Phase 5: Schema Validation & Investigation Queries

Before writing the model SQL, validate that source fields actually exist in
production. This catches a common issue with incremental dbt models where columns
are defined in SQL but not materialised.

**Step 1: Validate source schemas (3-part check)**

For each source table, run three checks — not just column existence:

```sql
-- 1. Column existence (dialect-specific):
--    Databricks/Spark: DESCRIBE <catalog>.<schema>.<table>;
--    Snowflake:        DESCRIBE TABLE <database>.<schema>.<table>;
--    BigQuery:         SELECT column_name, data_type
--                      FROM <project>.<dataset>.INFORMATION_SCHEMA.COLUMNS
--                      WHERE table_name = '<table>';

-- 2. Column population (is it actually filled, not just present?):
SELECT
    COUNT(*) AS total_rows,
    COUNT(<field>) AS non_null_count,
    ROUND(COUNT(<field>) * 100.0 / COUNT(*), 1) AS pct_populated
FROM <table>
WHERE <date_col> >= '<recent_date>';

-- 3. Data freshness (is the table up to date?):
SELECT MAX(<timestamp_col>) AS last_updated, COUNT(*) AS total_rows
FROM <table>;
```

A column can exist but be 100% NULL (partial backfill) or contain stale data
(last refresh weeks ago). All three checks must pass.

Cross-reference the results against every source field in the plan.
Flag any field that exists in the dbt SQL code but NOT in the production table.

**Step 2: Write investigation queries for open questions**

For each ACTION NEEDED item, write a targeted SQL query. Common patterns:

- **Coverage check**: `SELECT COUNT(*), COUNT(field), COUNT(DISTINCT field) FROM table`
- **Value distribution**: `SELECT field, COUNT(*) GROUP BY field ORDER BY 2 DESC`
- **Join fan-out validation**: Compare `COUNT(*)` before and after a LEFT JOIN —
  if the count increases, you have a 1:N fan-out
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

-- Source CTEs (refs at top for lineage visibility)
with source_a as (
    select * from {{ ref('source_model_a') }}
),

-- Transformation CTEs (one per logical step)
transformed as (
    select
        source_a.field_1,
        case
            when source_a.category in ('a', 'b') then 'Group 1'
            else 'Unmapped'  -- always include ELSE for taxonomy mappings
        end as category_group,
        source_b.revenue
    from source_a
    left join source_b
        on source_a.key = source_b.key
),

final as (
    select * from transformed
)

select * from final
```

**Recommended conventions (apply to every dbt project):**

1. **Refs at top:** List all `{{ ref('...') }}` source CTEs at the top of the file
   with `select * from {{ ref('...') }}` so data lineage is immediately visible.
   Keep all transformation logic in later CTEs — never mix source selection with
   business logic. (Note: some teams prefer explicit column lists in source CTEs
   for stricter contracts — follow the project's existing pattern.)

2. **Surrogate primary key:** Add a primary key using
   `{{ dbt_utils.generate_surrogate_key([...]) }}` (or `{{ dbt.generate_surrogate_key() }}`
   for dbt Core 1.6+) composed of all grain-defining dimension columns. Name it
   `[model_name]_sk` (dbt-labs convention) or `[model_name]_key` (also common).
   In the companion YAML, add two separate tests:
   - Column-level `unique` and `not_null` tests on the key column
   - `dbt_utils.unique_combination_of_columns` on the **component dimension columns**
     (not on the key itself — that would be redundant with the `unique` test)

3. **Group only by grain dimensions:** In GROUP BY, include only columns that define
   the output grain. For dimensions that are 1:1 with the grain, use a non-deterministic
   aggregate in SELECT instead of adding to GROUP BY:
   - Spark/Databricks: `first(<col>)`
   - Snowflake: `ANY_VALUE(<col>)`
   - BigQuery: `ANY_VALUE(<col>)`
   - Postgres/Redshift: `MIN(<col>)` (no ANY_VALUE)

   This makes the intended grain explicit and prevents accidental grain changes.

4. **No unnecessary deduplication:** Don't add `qualify row_number() over (...)` or
   `distinct` on CTEs sourcing from upstream models that already guarantee uniqueness.
   Check the upstream model first — if it's already deduplicated, the extra logic adds
   cost and obscures intent.

5. **Linting compliance:** Follow the project's sqlfluff rules (check `.sqlfluff` config
   for overrides). Common rules: explicit `as` for aliases, one select target per line,
   consistent indentation, no SQL reserved keywords as column identifiers.

6. **CASE statements need ELSE:** For all taxonomy/classification CASE statements,
   always include an ELSE clause — either `ELSE 'Unmapped'` (with a monitoring query to
   detect new unmapped values) or `ELSE original_field` to preserve the raw value.
   An incomplete CASE without ELSE silently produces NULL for unmatched values.

7. **Deterministic tie-breakers:** For all `ROW_NUMBER()` / `QUALIFY` patterns, always
   use at least two ORDER BY columns, with the second being a unique ID. A single
   timestamp column is nondeterministic when ties exist.

**Dialect-specific patterns (check your warehouse):**

| Pattern | Spark/Databricks | Snowflake | BigQuery | Postgres/Redshift |
|---------|-----------------|-----------|----------|-------------------|
| Schema inspection | `DESCRIBE <catalog>.<schema>.<table>` | `DESCRIBE TABLE <db>.<schema>.<table>` | `INFORMATION_SCHEMA.COLUMNS` | `\d <table>` or `information_schema` |
| Non-deterministic agg | `first(<col>)` | `ANY_VALUE(<col>)` | `ANY_VALUE(<col>)` | `MIN(<col>)` |
| Surrogate key standalone | `md5(cast(coalesce(cast(col as string), '') as string))` | `MD5(col::varchar)` | `TO_HEX(MD5(CAST(col AS STRING)))` | `MD5(col::text)` |
| 3-part table name | `catalog.schema.table` | `database.schema.table` | `project.dataset.table` | `schema.table` (2-part) |

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

**Step 1: Create a standalone verification query**
- Replace `{{ ref('...') }}` with fully-qualified table names (see dialect table above)
- Replace `{{ dbt_utils.generate_surrogate_key(...) }}` with the dialect-specific
  hash equivalent (see dialect table above — and check if the project sets
  `surrogate_key_treat_nulls_as_empty_strings` which changes the null sentinel)
- Add a date filter to avoid full table scans
- Add `LIMIT 100` for quick syntax validation

**Step 2: Inspect the output for data quality issues**
Common bugs that only surface with real data:
- **NULL fields you expected populated** — check if the source field uses a
  different name, or if COALESCE fallbacks are needed
- **Fields populated on rows they shouldn't be** — especially when joining on
  URL-extracted IDs (see "URL-Extracted Join Keys" below)
- **Unexpected values** — enum fields with values not in your CASE statement
  (this is why ELSE clauses matter)
- **Row count inflation** — compare `COUNT(*)` before and after each LEFT JOIN
  to catch fan-out from 1:N joins

**Step 3: Write standalone QA queries**
Each QA query should be self-contained (no dependency on materialised output).
Key checks:
- NULL rates per column
- Value distribution for computed fields (channel groupings, discount segments)
- Join fan-out validation (count before vs after join)
- Data freshness (`MAX(timestamp_col)` within expected window)
- Single-entity journey trace (visual sanity check of one user's event sequence)

## Tips for Common Challenges

### Taxonomy Mismatches
When source systems use different category names (e.g., UTM tags vs promo channel
categories), build the mapping CASE statement explicitly in the plan. Don't defer
this to implementation — taxonomy decisions need human review. Always include an
ELSE clause in the CASE to catch unmapped values.

### Incremental Model Column Drift
dbt incremental models (`materialized='incremental'` or `microbatch`) silently
omit new columns added to the SQL after initial materialization. The column
exists in the code but not in the production table. This cascades: if model A
is missing a column, every downstream incremental model (B, C) that selects
from A will also be missing it.

**Diagnosis:**
1. Run schema inspection (dialect-specific) to confirm the column is missing
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
Even with this config enabled, the issue can still occur.

### Multiple Source Options
When a field could come from multiple sources (e.g., geo_country from IP vs
shipping_country from orders), document the trade-offs in the plan and recommend
a COALESCE fallback pattern. Let the human decide priority order.

### Grain Mismatches
If the target grain is finer than the source (e.g., event-level target but
session-level source), you need to join back to the finer-grained table.
If coarser, decide on the aggregation strategy (SUM, COUNT DISTINCT, etc.)
and document it in the plan.

When requirements decompose into multiple models (e.g., a detail fact + an
aggregate), note this during planning. Criteria for splitting: if the target
has both detail and aggregate grains, if transformation logic is reusable
across models, or if a single model exceeds ~200 lines.

### URL-Extracted Join Keys (Critical for Event Data)

Some event tracking systems extract IDs from URLs via regex. For example,
Snowplow computes `event_order_id` from any URL matching `/menu/{8-digit-id}`:

```sql
cast(coalesce(
    event_json.order_id,
    split(regexp_extract(page_urlpath, '\/menu\/[0-9]{8}', 0), '/')[2]
) as bigint) as event_order_id
```

This means **every page view in the order flow** gets the order ID — not just
the completion event. If you LEFT JOIN an order table on this field, order
dimensions (revenue, discount, box_mix) silently fan out to ALL events in the
session. The query runs without errors and row counts look normal, but the data
is wrong.

**Symptoms:** `transaction_id`, `revenue`, or order dimensions appear on
page_view, menu_browse, or other non-conversion events. All events in a session
share the same order field values.

**Fix — 3-step gating pattern:**

1. **Identify the actual conversion event** using a session-level flag plus
   deterministic ordering:

```sql
order_events as (
    select event_id, event_order_id
    from events
    where event_order_id is not null
        and order_placed_in_session_flag = 1
    qualify row_number() over (
        partition by session_id, event_order_id
        order by collector_tstamp desc, event_id desc
    ) = 1
)
```

2. **Join the order table through the gating CTE**, not directly on the
   URL-extracted field:

```sql
left join order_events
    on events.event_id = order_events.event_id
left join committed_orders
    on order_events.event_order_id = committed_orders.order_id
```

3. **Gate ALL order-derived fields** behind the event check:

```sql
case when order_events.event_id is not null
    then committed_orders.gross_revenue
end as purchase_revenue
```

**Verification:** After fixing, `COUNT(transaction_id)` should approximately
equal `COUNT(DISTINCT transaction_id)` (1:1). Before the fix, the former is
orders of magnitude larger.

**This pattern applies to ANY URL-extracted ID** — not just order IDs. If
`product_id`, `category_id`, or similar fields are extracted from URLs, they
will have the same fan-out behaviour. The `order_placed_in_session_flag` is
necessary but not sufficient (it's 1 for ALL events in a converting session)
— you still need the ROW_NUMBER to pick one specific event.

### COALESCE Across Multiple Event Name Fields
Event tracking systems often have multiple name fields with different coverage.
For example, Snowplow has `se_action` (structured events), `event_action` (app
events), and `event_name` (native type like page_view). Use COALESCE to unify,
but audit each field separately first — they may contain different types of events
(user interactions vs telemetry). When using COALESCE, also add an
`event_name_source` companion column (e.g., 'se_action', 'event_action',
'event_name') to preserve which field the value came from. This allows downstream
consumers to disaggregate if the merged values have different semantics.

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
FROM events
WHERE se_action IS NOT NULL
GROUP BY 1, 2

UNION ALL

SELECT 'event_action' AS field_source, event_action AS event_value,
       COUNT(*) AS event_count, COUNT(DISTINCT session_id) AS session_count
FROM events
WHERE se_action IS NULL AND event_action IS NOT NULL
GROUP BY 1, 2

UNION ALL

SELECT 'event_name' AS field_source, event_name AS event_value,
       COUNT(*) AS event_count, COUNT(DISTINCT session_id) AS session_count
FROM events
WHERE se_action IS NULL AND event_action IS NULL AND event_name IS NOT NULL
GROUP BY 1, 2

UNION ALL

SELECT 'ALL_NULL' AS field_source, '(all fields NULL)' AS event_value,
       COUNT(*) AS event_count, COUNT(DISTINCT session_id) AS session_count
FROM events
WHERE se_action IS NULL AND event_action IS NULL AND event_name IS NULL
GROUP BY 1, 2

ORDER BY field_source, session_count DESC
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
reusable across models. Keep seeds small (<1000 rows).

Note: If your CI/CD uses `dbt build`, seed dependencies are handled automatically
(seeds are included in DAG order). If your CI uses `dbt run`, ensure `dbt seed`
runs first — otherwise the blacklist anti-join will be silently ineffective.
