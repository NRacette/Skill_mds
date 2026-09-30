Create a first-draft Claude skill at `.claude/skills/rdl-to-semantic-model/` that
migrates SSRS paginated reports (.rdl) from SQL Server base tables (equivalent to our
Fabric bronze layer) to Power BI paginated reports that query a Fabric semantic model
with DAX.

Architecture (follow exactly):
- My existing parser (<path/to/parser>) converts .rdl to YAML. Treat the YAML as a
  READ-ONLY inventory for reasoning. Per the audit in `audit/report.md`, read these
  elements directly from the RDL instead: <list from audit, or "none">.
- Claude never regenerates RDL. It produces a change plan (`<report>.plan.yaml`) of
  targeted edits keyed by XPath. A deterministic patcher applies the plan to a COPY
  of the original .rdl, so all layout and styling passes through untouched.
- The user supplies a mapping file per semantic model (schema below).

Core rule: preserve every `<Field Name>` exactly and only rewrite `<DataField>` to
the DAX output column, so `Fields!X.Value` expressions in the report body keep
working. Changing a Field Name is forbidden unless the plan also rewrites every
expression that references it, and that case must be flagged for review.

Files to create:
1. SKILL.md: a concise trigger description plus the workflow
   inventory → classify → map → plan → patch → validate → migration log.
   Keep it under ~300 lines and push detail into references/.
2. references/rdl-elements.md: which RDL elements the plan may touch (DataSources,
   DataSets/Query/CommandText/QueryParameters, Field DataField, ReportParameters
   defaults and valid values, rd:DesignerState removal) vs. must never touch
   (layout, styles, page setup, body item names).
3. references/dax-patterns.md: worked examples for converting T-SQL datasets to
   SUMMARIZECOLUMNS; single-value filters; multi-value parameter filters
   (TREATAS-based pattern); date ranges; "All"/blank handling; cascading parameter
   datasets (DISTINCT/VALUES queries for available values); and how DAX output
   column names map to DataField. Mark anything you are unsure of with
   `TODO: verify in Power BI Report Builder`. Do not invent syntax.
4. references/classification.md: rules for sorting each dataset into:
   - DIRECT (select/join/filter → DAX query)
   - LOGIC (CASE, window functions, derived columns → needs a model measure,
     calculated column, or gold-layer change; flag it, don't cram it into DAX)
   - STORED_PROCEDURE (always manual review)
   - UNMAPPED (a source column is missing from the mapping file)
   Reports with too much LOGIC or STORED_PROCEDURE content should get a
   recommendation to consider pointing at the SQL analytics endpoint instead.
5. references/golden-datasource.xml: a placeholder, with a comment telling me to
   paste the <DataSources> block from a report built in Power BI Report Builder
   against our semantic model. The skill copies this verbatim and never
   constructs connect strings itself.
6. references/unsupported-features.md: a checklist of SSRS features that need
   manual handling in Power BI (custom assemblies/code, shared data sources and
   datasets, subreport/drillthrough paths, external images, subscriptions), with
   TODO markers where it should be checked against Microsoft docs.
7. mappings/_template.yaml: the mapping schema, with entries holding
   source (schema.table.column or an aggregate expression), target
   ('Table'[Column] or [Measure]), kind (column|measure|derived|unmapped),
   filterable, and notes. Also a `parameters:` section mapping each @Param to a
   model column, with multi_value and an available_values DAX query.
8. scripts/build_plan_context.py: loads the YAML inventory + mapping file,
   resolves every source column referenced in each dataset against the mapping,
   and emits a compact JSON context listing resolved, unmapped, and
   logic-bearing items. This is what Claude reads before writing a plan.
9. scripts/apply_plan.py: lxml patcher. Namespace-aware, copies to an output
   folder (never edits the original), and fails loudly if an XPath matches zero
   or multiple nodes.
10. scripts/validate_rdl.py: checks well-formed XML, that the namespace is RDL
    2016, that every Fields!X reference in expressions exists in its dataset,
    that no DataField is still a SQL column name, and that every mapped model
    object exists (read from an exported model metadata file:
    <path/to/model TMDL or metadata>).
11. A per-report `migration_log.md` template: classification per dataset, edits
    applied, flags for manual review, and a test checklist (compare row counts
    and a rendered export for a fixed parameter set, old vs. new).

Constraints:
- Python 3.11, lxml, pyyaml only.
- Scripts need CLI args, docstrings, and clear errors.
- Include one end-to-end example in `examples/` using a small sample report from
  <path/to/samples>: YAML → context → plan → patched .rdl → validation output.
- Before writing files, show me the planned folder tree and the plan.yaml schema,
  and wait for my confirmation.