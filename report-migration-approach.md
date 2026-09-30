# Report Migration: LLM-Assisted Data Modeling

## Goal

Migrate legacy SSRS reports onto the Fabric data model by giving Copilot and other LLM tools enough structured context to:

1. map each report's fields and logic onto the existing model,
2. identify where the model needs to change, and
3. conform new report logic to the business rules already built into the model instead of duplicating them.

## Principles

- **Current state is read-only.** Everything extracted from Fabric (`model/`) is truth. The LLM proposes changes; it never edits `model/` directly.
- **Generate what can be generated.** Types, keys, stats, relationships, and measures come from scripts. Hand-written content is limited to what only a human knows: grain, business meaning, code meanings, quirks.
- **LLM drafts, human reviews.** Drafted content is marked `reviewed: false` and is reviewed only when a report pulls that table into scope.
- **Expose, then extend, then build new.** When a report needs something, first check whether it exists but isn't exposed, then whether an existing table can be extended, and only then create something new.
- **Keep it small.** Start with a few reports end to end. Add tooling only when the LLM visibly guesses wrong without it.

## Repo Structure

```
report-migration/
├── AGENTS.md                  # repo rules for LLM tools (or .github/copilot-instructions.md)
├── decisions.md               # grain choices, conformed dims, naming rules
├── model/                     # CURRENT STATE, extracted via Fabric Git integration
│   ├── warehouse/             # int_*.sql, dim_*.sql, fact_*.sql
│   └── semantic/              # TMDL
├── sources/
│   └── <system>/
│       ├── <system>_sources.yaml
│       └── procs/*.sql        # stored procedures used by reports
├── dictionary/
│   ├── _index.yaml
│   ├── int/<table>.yaml
│   ├── gold/<table>.yaml
│   └── semantic/<model>_measures.yaml
├── reports/
│   └── <report_name>/
│       ├── report.rdl
│       ├── report.yaml        # parser output
│       └── mapping.yaml       # field + rule mapping, status
├── skills/                    # data modeling + report migration skills
└── tools/                     # rdl parser, extract/profile scripts
```

---

## 1. `model/`: Current State

Pull this through **Fabric Git integration**: warehouses export as SQL database projects and semantic models export as TMDL. Re-sync after every model change. Nothing in this folder is hand-edited.

---

## 2. `sources/`: Source Tables

Answers "what raw data exists and what does it actually look like?" Used to trace report fields back to source columns and to check that proposed model changes are supported by real data.

### File format

One file per source system, tables keyed by name.

```yaml
system: erp
database: ERP_PROD
extracted: 2026-09-30

tables:
  dbo.sales_order_header:
    description: One row per sales order, including cancelled orders.
    grain: order
    row_count: 4812330
    primary_key: [order_id]
    joins:
      - {to: dbo.customer, on: customer_id, cardinality: many-to-one}
      - {to: dbo.sales_order_detail, on: order_id, cardinality: one-to-many}
    columns:
      order_id:      {type: int, role: pk, nulls: 0.0%, distinct: 4812330}
      customer_id:   {type: int, fk_to: dbo.customer.customer_id, nulls: 0.0%, distinct: 212044}
      order_status:  {type: char(1), nulls: 0.0%, distinct: 5, values: {C: closed, O: open, X: cancelled, H: hold, R: returned}}
      order_date:    {type: datetime, nulls: 0.2%, range: [2009-01-04, 2026-09-29]}
      total_amt:     {type: decimal(12,2), nulls: 0.0%, range: [-15000.00, 982000.00]}
    sample:
      columns: [order_id, customer_id, order_status, order_date, total_amt]
      rows:
        - [1001, 88, C, 2024-03-02, 1250.00]
        - [1002, 91, X, 2024-03-02, 0.00]
        - [1003, 88, R, 2024-03-05, -340.50]
    notes: |
      Returns are separate orders with status R and negative totals.
      order_date is local time (CST), not UTC.
    reviewed: false
```

If a system grows past roughly 30–50 tables, split into one file per table with the same structure.

### Tags

| Tag | Level | Source | Purpose |
|---|---|---|---|
| `description` | table | LLM draft | What one row represents |
| `grain` | table | LLM draft | Most important input for modeling decisions |
| `row_count` | table | generated | Scale; dimension vs. fact |
| `primary_key` | table | generated | Joins, dimension keys |
| `joins` | table | hand / LLM from procs | Undeclared relationships the LLM can't guess |
| `type` | column | generated | Always |
| `role` / `fk_to` | column | generated | Keys |
| `nulls` | column | generated | Optional or deprecated columns |
| `distinct` | column | generated | Codes, flags, candidate keys |
| `values` | column | generated codes, hand/LLM meanings | Code meanings are the highest-value hand content |
| `range` | column | generated | Negatives, placeholder dates, history depth |
| `notes` | table | hand, as needed | Quirks; add when a mapping goes wrong |

Skip audit columns (`created_by`, `rowversion`, etc.) except the CDC watermark column.

### Sample rows

- Columnar format: headers once, rows as arrays (fewer tokens).
- 3–5 rows, chosen for variety: different status codes, a row with nulls, an edge case.
- Wide tables: sample only keys and columns the model or reports use.
- Pick samples that join across tables (order rows whose `customer_id` appears in the customer sample).
- ISO dates, `null` for nulls, truncate long text to ~50 chars, quote values like `Y`/`N`.
- Scrub PII with consistent fakes; keep keys and codes real.
- Bulk samples are just `limit(3)`. Deliberate sampling only for tables in priority reports.

### Extraction

**Step 1: Get the list of tables actually brought into Fabric.** Run on the bronze SQL analytics endpoint (or query the CDC pipeline control table, if one exists):

```sql
SELECT STRING_AGG(
         CAST(REPLACE(TABLE_NAME, 'erp_dbo_', 'dbo.') AS varchar(max)),
         ',') AS table_list
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'dbo'
  AND TABLE_NAME LIKE 'erp[_]%';
```

Adjust the `REPLACE` to match the bronze naming convention.

**Step 2: Pull structure from SQL Server, filtered to those tables.** Catalog views only, no data scanned. SQL Server is used here because Spark and the lakehouse endpoint lose exact types and constraints.

```sql
DECLARE @tables nvarchar(max) = N'dbo.sales_order_header,dbo.customer';  -- paste list from step 1

SELECT
    s.name + '.' + t.name AS table_name,
    c.column_id,
    c.name AS column_name,
    ty.name + CASE
        WHEN ty.name IN ('varchar','char','varbinary')
            THEN '(' + CASE WHEN c.max_length = -1 THEN 'max' ELSE CAST(c.max_length AS varchar) END + ')'
        WHEN ty.name IN ('nvarchar','nchar')
            THEN '(' + CASE WHEN c.max_length = -1 THEN 'max' ELSE CAST(c.max_length/2 AS varchar) END + ')'
        WHEN ty.name IN ('decimal','numeric')
            THEN '(' + CAST(c.precision AS varchar) + ',' + CAST(c.scale AS varchar) + ')'
        ELSE '' END AS data_type,
    c.is_nullable,
    CASE WHEN pk.column_id IS NOT NULL THEN 1 ELSE 0 END AS is_pk,
    fk.ref_table + '.' + fk.ref_column AS fk_to,
    rc.row_count
FROM sys.tables t
JOIN sys.schemas s  ON s.schema_id = t.schema_id
JOIN sys.columns c  ON c.object_id = t.object_id
JOIN sys.types ty   ON ty.user_type_id = c.user_type_id
LEFT JOIN (
    SELECT ic.object_id, ic.column_id
    FROM sys.indexes i
    JOIN sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
    WHERE i.is_primary_key = 1
) pk ON pk.object_id = c.object_id AND pk.column_id = c.column_id
LEFT JOIN (
    SELECT parent_object_id, parent_column_id,
           OBJECT_SCHEMA_NAME(referenced_object_id) + '.' + OBJECT_NAME(referenced_object_id) AS ref_table,
           COL_NAME(referenced_object_id, referenced_column_id) AS ref_column
    FROM sys.foreign_key_columns
) fk ON fk.parent_object_id = c.object_id AND fk.parent_column_id = c.column_id
LEFT JOIN (
    SELECT object_id, SUM(row_count) AS row_count
    FROM sys.dm_db_partition_stats
    WHERE index_id IN (0,1)
    GROUP BY object_id
) rc ON rc.object_id = t.object_id
WHERE t.is_ms_shipped = 0
  AND s.name + '.' + t.name IN (SELECT LTRIM(RTRIM(value)) FROM STRING_SPLIT(@tables, ','))
ORDER BY table_name, c.column_id;
```

Notes: `STRING_SPLIT` needs SQL Server 2016+ with compatibility level 130+ (otherwise load names into a table variable). `sys.dm_db_partition_stats` needs `VIEW DATABASE STATE`; without it, drop that join and count rows in the notebook. Export the result to CSV and upload it to the lakehouse `Files/` folder.

**Step 3: Profile bronze tables in Fabric.** One Spark pass per table, several tables in parallel.

```python
import pandas as pd
from concurrent.futures import ThreadPoolExecutor
from pyspark.sql import functions as F

meta = pd.read_csv("/lakehouse/default/Files/source_metadata.csv")
SKIP = {"created_by", "modified_by", "rowversion"}
RANGE_TYPES = ("int", "bigint", "decimal", "double", "float", "date", "timestamp")

def to_bronze(t):  # dbo.sales_order_header -> bronze.erp_dbo_sales_order_header
    return "bronze.erp_" + t.replace(".", "_")

def profile(table):
    try:
        rows = meta.loc[meta.table_name == table, "row_count"].fillna(0).iloc[0]
        df = spark.table(to_bronze(table))
        if rows > 20_000_000:
            df = df.sample(0.05, seed=1)
        cols = [c for c in df.columns if c.lower() not in SKIP]
        dtypes = dict(df.dtypes)

        # pass 1: nulls, distinct, min/max for all columns in one job
        aggs = [F.count(F.lit(1)).alias("__n")]
        for c in cols:
            aggs += [F.sum(F.col(c).isNull().cast("int")).alias(f"{c}|nulls"),
                     F.approx_count_distinct(c).alias(f"{c}|distinct")]
            if dtypes[c].startswith(RANGE_TYPES):
                aggs += [F.min(c).alias(f"{c}|min"), F.max(c).alias(f"{c}|max")]
        r = df.agg(*aggs).first().asDict()
        n = r["__n"] or 1

        # pass 2: code values for low-cardinality columns in one job
        low_card = [c for c in cols if r[f"{c}|distinct"] <= 20]
        codes = df.agg(*[F.collect_set(c).alias(c) for c in low_card]).first().asDict() if low_card else {}

        columns = {}
        for c in cols:
            col = {"nulls": f"{100 * r[f'{c}|nulls'] / n:.1f}%", "distinct": r[f"{c}|distinct"]}
            if c in codes:
                col["values"] = sorted(str(v) for v in codes[c] if v is not None)
            elif f"{c}|min" in r:
                col["range"] = [str(r[f"{c}|min"]), str(r[f"{c}|max"])]
            columns[c.lower()] = col

        pdf = df.limit(3).toPandas()
        sample = {"columns": list(pdf.columns),
                  "rows": [[None if pd.isna(v) else str(v) for v in row] for row in pdf.values]}
        return table, {"columns": columns, "sample": sample}
    except Exception as e:
        print(f"skipped {table}: {e}")
        return table, None

with ThreadPoolExecutor(max_workers=8) as pool:
    results = {t: p for t, p in pool.map(profile, meta.table_name.unique()) if p}
```

**Step 4: Merge and write YAML.**

```python
import yaml
from datetime import date

out = {"system": "erp", "extracted": str(date.today()), "tables": {}}

for t, g in meta.groupby("table_name"):
    prof = results.get(t)
    if not prof:
        continue
    cols = {}
    for _, row in g.iterrows():
        name = row.column_name.lower()
        if name not in prof["columns"]:      # not landed in bronze, or skipped
            continue
        col = {"type": row.data_type}
        if row.is_pk == 1:
            col["role"] = "pk"
        if isinstance(row.fk_to, str):
            col["fk_to"] = row.fk_to
        col.update(prof["columns"][name])
        cols[row.column_name] = col
    out["tables"][t] = {
        "description": None,
        "grain": None,
        "row_count": int(g.row_count.iloc[0]) if pd.notna(g.row_count.iloc[0]) else None,
        "primary_key": g.loc[g.is_pk == 1, "column_name"].tolist(),
        "joins": [],
        "columns": cols,
        "sample": prof["sample"],
        "notes": None,
        "reviewed": False,
    }

with open("/lakehouse/default/Files/erp_sources.yaml", "w") as f:
    yaml.safe_dump(out, f, sort_keys=False, allow_unicode=True, width=200, default_flow_style=None)
```

`default_flow_style=None` keeps each column and each sample row on one line.

**Step 5: LLM drafts the hand-written tags.** In batches, give the LLM the generated YAML plus any procs in `sources/<system>/procs/` that reference each table (grep by table name). It drafts `description`, `grain`, `joins`, and code meanings (from `CASE` statements, `JOIN` conditions, and lookup tables). Everything stays `reviewed: false` until a priority report touches the table.

### Stored procedures

Many report datasets call procs, and much of the business logic lives there. Dump them into `sources/<system>/procs/*.sql`:

```sql
SELECT OBJECT_SCHEMA_NAME(object_id) + '.' + OBJECT_NAME(object_id) AS proc_name, definition
FROM sys.sql_modules
WHERE OBJECTPROPERTY(object_id, 'IsProcedure') = 1;
```

---

## 3. `reports/`: Reports Being Migrated

### Export and prioritize

Bulk export RDLs plus shared datasets (`.rsd`) and data sources (`.rds`):

```powershell
Install-Module ReportingServicesTools
Out-RsFolderContent -ReportServerUri http://yourserver/ReportServer `
  -RsFolder / -Destination C:\rdl_export -Recurse
```

Query `ExecutionLog3` in the ReportServer database for run counts per report over the last ~90 days. This gives the migration priority order, flags dead reports, and shows which parameter values are actually used.

### `report.yaml`: parser output

The report reduced to what matters for modeling. Visual layout is dropped.

```yaml
report: Monthly Sales by Region
usage_90d: 412
parameters:
  - {name: StartDate, type: date}
  - {name: Region, type: string, multi: true, source_query: "SELECT DISTINCT region FROM ..."}
datasets:
  - name: dsSales
    command_type: StoredProcedure
    command: dbo.rpt_monthly_sales       # see sources/erp/procs/
    fields: [region, order_month, net_sales, order_count]
tablix:
  - name: SalesTable
    row_groups: [region, order_month]    # implies required grain
    columns:
      - {header: Net Sales, expr: "=Sum(Fields!net_sales.Value)"}
      - {header: Avg Order, expr: "=Sum(Fields!net_sales.Value)/Sum(Fields!order_count.Value)"}
filters:
  - "=Fields!order_status.Value <> 'X'"
```

Priority of what the parser must capture:

1. Dataset queries and proc references (joins, filters, business logic)
2. Cell expressions (become DAX measures)
3. Row and column groupings (required grain)
4. Report-level filters and parameters (hidden business rules)

Shared dataset references must be resolved to the `.rsd` query; a report pointing at a shared dataset has no query of its own.

### `mapping.yaml`: the working artifact

Generated by the modeling skill, reviewed by a human. The only hand-edited file in the report folder.

```yaml
report: Monthly Sales by Region
target_semantic_model: Sales

fields:
  - report_field: net_sales
    source: erp.sales_order_detail.extended_amount (via rpt_monthly_sales)
    target: "[Net Sales]"
    status: mapped
  - report_field: region
    target: dim_customer.sales_region
    status: mapped
  - report_field: Avg Order
    target: "[Avg Order Value]"
    status: needs_measure

rules:
  - report_logic: "order_status <> 'X'"
    status: matches
    existing_rule: int_sales_lines.exclude_cancelled
  - report_logic: "region from ERP ship-to state only"
    status: conflicts
    existing_rule: dim_customer.region_assignment
    note: model uses CRM territory first; confirm with BA
  - report_logic: "exclude internal test customers (customer_id < 100)"
    status: new
    proposed_layer: int

parameters:
  - {name: Region, target: dim_customer.sales_region}

grain_check: region x month, supported by fact_sales + dim_date
```

**Field status values**

| Status | Meaning |
|---|---|
| `mapped` | Direct match in the model |
| `needs_measure` | Model has the data, DAX measure missing |
| `needs_column` | Model lacks the attribute, source has it |
| `gap` | Data not in the model, or definition unclear |

**Rule status values**

| Status | Meaning |
|---|---|
| `matches` | Report uses an existing model rule |
| `conflicts` | Report defines it differently; needs a BA decision, never resolved silently |
| `new` | Rule doesn't exist in the model yet |

Grep all mappings for `gap`, `needs_column`, and `new` to get the model backlog. Items appearing across several reports go first. The `conflicts` list is an inventory of every place legacy reports disagreed on business definitions.

---

## 4. `dictionary/`: Data Model Dictionary

A compact index of the int and gold layers, focused on transformation logic and business rules. It exists so the LLM can decide what a new report needs without reading every SQL file.

### `_index.yaml`: generated

```yaml
tables:
  int_sales_lines: {layer: int, grain: order line, purpose: cleaned + conformed order lines}
  fact_sales:      {layer: gold, grain: order line, depends_on: [int_sales_lines]}
  dim_customer:    {layer: gold, grain: customer per version, scd: 2}
bus_matrix:
  fact_sales: [dim_customer, dim_date, dim_product]
```

### Int table file

```yaml
# dictionary/int/int_sales_lines.yaml
table: int_sales_lines
layer: int
grain: one row per order line
purpose: Cleaned order lines with status, returns and currency conformed.
sql_file: model/warehouse/int/int_sales_lines.sql
depends_on: [erp.dbo.sales_order_detail, erp.dbo.sales_order_header, int_fx_rates]
rules:
  - name: exclude_cancelled
    logic: order_status <> 'X'
  - name: returns_as_negative
    logic: status R lines carry negative qty and amount
  - name: usd_conversion
    logic: amounts converted to USD at daily rate on order_date
  - name: net_amount
    logic: extended_amount - discount_amount, excl. tax
columns:
  order_line_id:  {type: bigint, role: pk}
  net_amount_usd: {type: decimal(18,2), rule: net_amount}
  order_status:   {type: char(1)}
used_by: [fact_sales, fact_returns]
reviewed: false
```

### Gold table file

```yaml
# dictionary/gold/dim_customer.yaml
table: dim_customer
layer: gold
type: dimension
grain: one row per customer per version
scd: 2
business_key: [customer_id]
sql_file: model/warehouse/gold/dim_customer.sql
depends_on: [int_customer]
rules:
  - name: region_assignment
    logic: sales_region from CRM territory, falls back to ERP ship-to state
columns:
  customer_key:  {type: int, role: pk}
  customer_id:   {type: int, role: bk}
  sales_region:  {type: varchar(50), rule: region_assignment}
  legacy_region: {type: varchar(20), in_semantic: false}
  is_current:    {type: bit}
used_by: [fact_sales]
reviewed: false
```

### Semantic measures file

```yaml
# dictionary/semantic/sales_measures.yaml
semantic_model: Sales
measures:
  fact_sales:
    Net Sales:       {expr: "SUM(fact_sales[net_amount])", format: currency}
    Avg Order Value: {expr: "DIVIDE([Net Sales], DISTINCTCOUNT(fact_sales[order_id]))"}
```

### Tags

| Tag | Source | Purpose |
|---|---|---|
| `layer`, `type` | generated from schema / name prefix | int, dim, fact |
| `grain` | LLM draft from SQL | Checked first when a report groups at a new level |
| `scd`, `business_key` | LLM draft from SQL (`MERGE ... ON`, SCD2 logic) | Whether a new attribute is a simple add or needs history handling |
| `purpose` / `description` | LLM draft | One line |
| `sql_file` | generated | Pointer to the logic; full SQL is never copied into the dictionary |
| `depends_on` | LLM draft from `FROM` / `JOIN` | Walkable lineage: source → int → gold |
| `rules` | LLM draft from `WHERE`, `CASE`, calculations | The conforming surface |
| `columns` (`type`, `role`) | generated | Mapping report fields |
| `columns.rule` | LLM draft | Links a column to the rule that produces it |
| `in_semantic: false` | generated | Exists in gold but not exposed; cheapest fix for a gap |
| `used_by` | generated from other files' `depends_on` | Impact of a change |
| `reviewed` | human | Draft vs. confirmed |

### How to fill

1. **Columns and types:** from the Fabric warehouse SQL endpoint.

   ```sql
   SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE,
          CHARACTER_MAXIMUM_LENGTH, NUMERIC_PRECISION, NUMERIC_SCALE
   FROM INFORMATION_SCHEMA.COLUMNS
   WHERE TABLE_SCHEMA IN ('int', 'gold')
   ORDER BY TABLE_SCHEMA, TABLE_NAME, ORDINAL_POSITION;
   ```

2. **Measures and `in_semantic` flags:** Semantic Link in a Fabric notebook, or read the TMDL in `model/semantic/`.

   ```python
   import sempy.fabric as fabric
   sem_cols = fabric.list_columns("Sales")
   measures = fabric.list_measures("Sales")
   # print(df.columns) on each to confirm field names
   ```

   Compare `sem_cols` to the gold columns to set `in_semantic: false`.

3. **Grain, rules, depends_on, scd, business_key:** LLM drafts from each `.sql` file in `model/warehouse/`. Everything starts at `reviewed: false`.

4. **`_index.yaml` and `used_by`:** a small script reads `depends_on` across all files. Never hand-maintained.

### Regeneration

When the model changes, re-sync `model/` and regenerate. The script overwrites only generated fields (`columns[].type`, `role`, `in_semantic`, `used_by`, `sql_file`, `_index.yaml`, measures) and leaves hand/LLM fields untouched. New tables get empty hand fields. Changed SQL files should get their `rules` re-drafted and `reviewed` reset to `false`.

---

## 5. Conforming Workflow

For each report, in priority order:

1. **Parse** the RDL into `report.yaml`, resolving shared datasets and procs.
2. **Map fields** against `dictionary/` and the measures file.
3. **Extract rules** from the dataset query or proc: filters, `CASE` logic, calculations.
4. **Compare rules** with existing `rules` in the relevant int and gold files → `matches`, `conflicts`, or `new`.
5. **Resolve** each non-mapped field and new rule in this order:
   - **Expose:** column exists with `in_semantic: false`, so add it to the semantic model.
   - **Extend:** add a column or rule to an existing int or gold table.
   - **Build new:** only when neither of the above works.
6. **Place new rules** at the right layer: int if reusable across reports, gold column if it's an attribute, measure if it's an aggregation.
7. **Implement** in Fabric, re-sync `model/`, regenerate the dictionary, review the touched entries.

`conflicts` always go to a BA decision and are recorded in `decisions.md`.

---

## 6. `AGENTS.md` Essentials

Keep it short. Minimum rules:

- Never edit `model/`; propose changes as SQL diffs or new files for review.
- Read `dictionary/_index.yaml` first, then only the table files a report touches.
- Follow the order: expose, then extend, then build new.
- Never resolve a rule `conflict` silently; flag it.
- Respect grain and conformed dimension rules in `decisions.md`.
- Don't add scripts, folders, or steps that weren't asked for.

---

## 7. Getting Started

1. Sync `model/` via Fabric Git integration.
2. Export all RDLs and pull `ExecutionLog3` usage.
3. Parse everything to `report.yaml`.
4. Pick 2–3 high-usage reports.
5. Generate `sources/` and `dictionary/` structure (scripted parts only).
6. LLM-draft hand fields for tables those reports touch; review them.
7. Run the conforming workflow end to end on those reports.
8. Adjust the skills and file formats based on where the LLM guessed wrong, then scale out.
