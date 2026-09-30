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
      order_id:      {type: int, nulls: 0%, distinct: 4812330}
      customer_id:   {type: int, nulls: 0%, distinct: 212044}
      order_status:  {type: char(1), nulls: 0%, values: {C: closed, O: open, X: cancelled, H: hold, R: returned}}
      order_date:    {type: datetime, nulls: 0.2%, range: [2009-01-04, 2026-09-29]}
      total_amt:     {type: decimal(12,2), nulls: 0%, range: [-15000.00, 982000.00]}
      modified_date: {type: datetime, nulls: 0%}
    sample:
      columns: [order_id, customer_id, order_status, order_date, total_amt]
      rows:
        - [1001, 88, C, 2024-03-02, 1250.00]
        - [1002, 91, X, 2024-03-02, 0.00]
        - [1003, 88, R, 2024-03-05, -340.50]
    notes: |
      Returns are separate orders with status R and negative totals.
      order_date is local time (CST), not UTC.

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
ORDER BY table_name, c.column_id;

import yaml, pandas as pd
from concurrent.futures import ThreadPoolExecutor
from pyspark.sql import functions as F

meta = pd.read_csv("/lakehouse/default/Files/source_metadata.csv")
SKIP = {"created_by", "modified_by", "rowversion"}          # audit cols
RANGE_TYPES = ("int", "bigint", "decimal", "double", "date", "timestamp")

def to_bronze(t):  # dbo.sales_order_header -> bronze.erp_dbo_sales_order_header
    return "bronze.erp_" + t.replace(".", "_")

def profile(table):
    rows = int(meta.loc[meta.table_name == table, "row_count"].iloc[0] or 0)
    df = spark.table(to_bronze(table))
    if rows > 20_000_000:
        df = df.sample(0.05, seed=1)                          # stats on a sample for big tables
    cols = [c for c in df.columns if c.lower() not in SKIP]
    dtypes = dict(df.dtypes)

    # pass 1: nulls, distinct, min/max for every column in ONE job
    aggs = [F.count(F.lit(1)).alias("__n")]
    for c in cols:
        aggs += [F.sum(F.col(c).isNull().cast("int")).alias(f"{c}|nulls"),
                 F.approx_count_distinct(c).alias(f"{c}|distinct")]
        if dtypes[c].startswith(RANGE_TYPES):
            aggs += [F.min(c).alias(f"{c}|min"), F.max(c).alias(f"{c}|max")]
    r = df.agg(*aggs).first().asDict()
    n = r["__n"] or 1

    # pass 2: code values for low-cardinality columns, also ONE job
    low_card = [c for c in cols if r[f"{c}|distinct"] <= 20]
    codes = df.agg(*[F.collect_set(c).alias(c) for c in low_card]).first().asDict() if low_card else {}

    columns = {}
    for c in cols:
        col = {"nulls": f"{100 * r[f'{c}|nulls'] / n:.1f}%", "distinct": r[f"{c}|distinct"]}
        if c in codes:
            col["values"] = sorted(str(v) for v in codes[c] if v is not None)
        elif f"{c}|min" in r:
            col["range"] = [str(r[f"{c}|min"]), str(r[f"{c}|max"])]
        columns[c] = col

    sample = df.limit(3).toPandas().astype(str)
    return table, {"columns": columns,
                   "sample": {"columns": list(sample.columns), "rows": sample.values.tolist()}}

with ThreadPoolExecutor(max_workers=8) as pool:
    results = dict(pool.map(profile, meta.table_name.unique()))