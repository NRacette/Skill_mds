reports/<report_name>/
├── README.md        # context, decisions, validation
├── report.rdl
├── report.yaml      # what the report does (parser)
└── mapping.yaml     # how it maps to the model
---
report: Monthly Sales by Region
status: in_progress          # not_started | in_progress | validating | migrated | retired
legacy_path: /Sales/Monthly Sales by Region
fabric_target: Sales semantic model / Monthly Sales report
usage_90d: 412
business_owner: Jane Doe (Sales Ops)
consolidate_with: [Weekly Sales by Region]
---

# Monthly Sales by Region

## Purpose
What business question it answers, who uses it, and what decisions it drives.
Example: Regional managers review month-end sales vs. prior year to set next
month's targets. Run at month close; sometimes mid-month for forecasting.

## What It Shows
- **Grain:** region x month
- **Key metrics:** Net Sales, Order Count, Avg Order Value, YoY %
- **Filters/parameters users actually use:** date range, region (usually 1-3 regions)

## Business Definitions
How the business understands each metric, in plain language. These are
compared against model rules in `dictionary/`.
- **Net Sales:** invoiced amount after discounts, excluding tax and cancelled orders
- **Region:** the customer's assigned sales territory

## Model Objects Used
- Facts: `fact_sales`
- Dimensions: `dim_customer`, `dim_date`
- Measures: `[Net Sales]`, `[Avg Order Value]`
- Full field mapping: `mapping.yaml`

## Gaps and Conflicts
Headline summary only; detail lives in `mapping.yaml`.
- `needs_measure`: Avg Order Value
- `conflict`: legacy report assigns region by ship-to state; model uses CRM territory

## Decisions
Dated, with who decided.
- 2026-09-30: Use model's CRM-based region assignment (Jane Doe). Numbers will
  differ from legacy for ~3% of customers.

## Legacy Quirks: Do Not Replicate
Bugs, hardcoded values and workarounds in the old report that should not carry over.
- Hardcoded exclusion of customer_id 4411 (closed account, now handled by status)
- YoY % divides by zero when prior year is empty; shows #Error

## Validation
- **Test parameters:** Aug 2026, all regions; Jan 2025, Region = West
- **Reconcile against:** legacy report output for the same parameters
- **Acceptable variance:** exact on Net Sales except known region reassignment
- **Result:** pending

## Open Questions
- Should returns reduce Net Sales in the month of return or original order month?