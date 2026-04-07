# Dialect-Specific Patterns Reference

| Pattern | Spark/Databricks | Snowflake | BigQuery | Postgres/Redshift |
|---------|-----------------|-----------|----------|-------------------|
| Schema inspection | `DESCRIBE <catalog>.<schema>.<table>` | `DESCRIBE TABLE <db>.<schema>.<table>` | `INFORMATION_SCHEMA.COLUMNS` | `\d <table>` or `information_schema` |
| Non-deterministic agg | `first(<col>)` | `ANY_VALUE(<col>)` | `ANY_VALUE(<col>)` | `MIN(<col>)` |
| Surrogate key standalone | `md5(cast(coalesce(cast(col as string), '') as string))` | `MD5(col::varchar)` | `TO_HEX(MD5(CAST(col AS STRING)))` | `MD5(col::text)` |
| 3-part table name | `catalog.schema.table` | `database.schema.table` | `project.dataset.table` | `schema.table` (2-part) |
