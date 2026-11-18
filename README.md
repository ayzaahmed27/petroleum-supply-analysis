# U.S. Petroleum Supply Analysis

An end-to-end petroleum analytics project using the EIA API,
Databricks, Apache Spark, Delta Lake, and Power BI.

## Project Overview

This project analyzes U.S. petroleum production, imports, exports,
stocks, and product supplied as an indicator of consumption.
It includes national and PADD regional observations.

The implemented pipeline processes saved raw JSON through Bronze,
Silver, and Gold layers. Gold contains a dimensional model and a
regional aggregate table for analytical queries.

Power BI provides two report pages:

- Supply Overview: petroleum trends and regional comparisons.
- Pipeline Health: execution metrics, failures, and quarantined records.

The project includes schema enforcement, audited MERGE operations,
idempotency tests, parameterized backfills, schema-drift handling,
execution analysis, and a FinOps evaluation.

## Team

| Team Member | Student ID |
|---|---|
| Ayza Ahmed | 24L-2577 |
| Eman Adil | 24L-2589 |

## Repository

https://github.com/ayzaahmed27/petroleum-supply-analysis

## Technology Stack

| Component | Technology |
|---|---|
| Data source | EIA Open Data API v2 |
| Processing environment | Databricks Free Edition |
| Processing framework | Apache Spark / PySpark |
| Table storage | Delta Lake |
| Catalog and raw storage | Unity Catalog tables and Volume |
| Pipeline execution | Databricks notebooks and Job |
| Business intelligence | Power BI Desktop, Import mode |
| Version control | GitHub |

## Data Source

U.S. Energy Information Administration (EIA), API v2.

Dataset: Petroleum — Supply and Disposition.

Dataset endpoint:

https://api.eia.gov/v2/petroleum/sum/snd/

Data endpoint:

https://api.eia.gov/v2/petroleum/sum/snd/data/

### Data Collected

| Load | Period | Records | Raw JSON Size | Files |
|---|---|---:|---:|---:|
| Full load | January 2005–December 2025 | 675,752 | 236.33 MB | 136 |
| Incremental sample | January 2026 | 2,820 | 0.99 MB | 1 |

The historical baseline was expanded following feedback by extending
the history and including regional and stock-related series.

The combined observation dataset contains 678,572 records.

The complete raw dataset is stored in a Databricks Volume.
The repository contains selected historical pages and the incremental
sample, rather than all 136 historical files.

The `full_load/` folder is a sample, not a complete historical baseline.
Reproducing the full-baseline execution requires the complete raw files.

### Ingestion Pattern

- Full load: historical baseline extraction.
- Incremental load: newly published monthly observations.
- Revisions: re-fetch recent periods and apply changed observations.

Based on the January 2026 sample, a new month contains approximately
2,800 records and 1 MB of raw JSON. Actual volume varies.

The implemented end-to-end pipeline processes saved raw files.
Automatic retrieval of newly published EIA data and a bounded revision
window are future enhancements.

## Data Scope and Business Questions

Monthly observations cover the United States and PADD regions 1–5.

PADD means Petroleum Administration for Defense Districts.

| PADD | Region |
|---|---|
| 1 | East Coast |
| 2 | Midwest |
| 3 | Gulf Coast |
| 4 | Rocky Mountain |
| 5 | West Coast |

Included activities:

- Field production
- Refinery and blender net production
- Imports
- Exports
- Product supplied
- Ending stocks
- Stock change

The analytical questions are:

1. How have petroleum production and estimated consumption changed?
2. How do imports and exports compare for a selected product?
3. How do petroleum observations vary across regions and seasons?

Trading-partner country analysis is outside the current scope.

## Repository Contents

The six notebooks are located at the repository root.

| Notebook | Purpose |
|---|---|
| [Petroleum_Phase2_Pipeline.ipynb](Petroleum_Phase2_Pipeline.ipynb) | Bronze/Silver implementation, setup, and validation demonstrations |
| [Petroleum_Phase2_Execution_Evidence.ipynb](Petroleum_Phase2_Execution_Evidence.ipynb) | Executed Phase 2 notebook with saved test outputs |
| [Petroleum_Phase3_Gold.ipynb](Petroleum_Phase3_Gold.ipynb) | Gold development, validation, reconciliation, and optimization |
| [Petroleum_Pipeline_Core.ipynb](Petroleum_Pipeline_Core.ipynb) | Reusable Bronze/Silver processing functions |
| [Petroleum_Gold_Core.ipynb](Petroleum_Gold_Core.ipynb) | Gold schemas, setup, dimension/fact/aggregate loaders, and refresh orchestration |
| [Petroleum_End_to_End_Pipeline.ipynb](Petroleum_End_to_End_Pipeline.ipynb) | Parameterized pipeline entry point, audit verification, and evidence export |

Supporting files:

- `full_load/`: selected historical raw pages.
- `incremental_load/`: January 2026 raw payload.
- `file_summary.csv`: raw file names, record counts, and sizes.
- `validation_summary.csv`: extraction validation results.
- `SAMPLE_NOTE.txt`: explanation of sample selection.
- [phase2_audit_evidence.csv](phase2_audit_evidence.csv): selected Phase 2 audit evidence.
- [docs/gold_data_dictionary.md](docs/gold_data_dictionary.md):
  Gold column definitions, grains, keys, and ER diagram.

The Power BI dashboard and final architectural PDF are separate
submission deliverables.

## Pipeline Architecture

Processing follows this order:

1. Retain original raw JSON files in a Databricks Volume.
2. Validate observations against the raw contract.
3. Merge accepted observations into Bronze with provenance metadata.
4. Clean, cast, and deduplicate Bronze observations.
5. Merge valid observations into Silver.
6. Refresh the five Gold dimensions.
7. Refresh the detailed Gold observation fact.
8. Refresh the monthly regional aggregate.
9. Verify and export execution audit evidence.
10. Refresh the Power BI Import model to display updated source data.

Invalid records are written to quarantine.
Each processing operation writes an execution log.

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| Raw | Original saved JSON payloads for replay and backfill |
| Bronze | Accepted source observation versions and ingestion metadata |
| Silver | One cleaned current observation per monthly series |
| Gold | Dimensions, detailed facts, and regional summaries |
| Operational tables | Execution logs and rejected-record evidence |
| Power BI | Business analysis and pipeline monitoring |

### Main Operational Tables

| Purpose | Table |
|---|---|
| Bronze | `workspace.default.petroleum_bronze` |
| Silver | `workspace.default.petroleum_silver` |
| Quarantine | `workspace.default.petroleum_quarantine` |
| Execution logs | `workspace.default.petroleum_execution_logs` |

Controlled revision and schema-drift tests use isolated tables so that
synthetic records do not enter the main observation dataset.

## Schema Enforcement and Validation

Raw schemas use explicit `StructType` and `StructField` definitions.
Schema inference is not used.

The raw contract requires all 11 expected fields. Values must be strings
or null, and `period` and `series` must be nonempty.

Validation rejects or quarantines records with:

- Unexpected or missing fields
- Incompatible raw types
- Missing observation keys
- Invalid monthly periods
- Invalid numeric values, including NaN and infinity
- Conflicting observations for the same Silver business key within a batch

Source names such as `area-name` are mapped to `area_name`.

Silver processing trims strings, converts monthly periods to the first
day of the month, and casts numeric values to doubles.

Null values remain null. Negative values are permitted because some
activities, such as stock changes, can be negative.

Identical observations within a batch are deduplicated. Conflicting
observations are quarantined instead of selecting an arbitrary value.

The implemented schema-drift policy is rejection and quarantine.
Automatic schema evolution is not enabled.

## Bronze Data Dictionary

**Grain:** one accepted source observation version within its batch,
file, and record position.

**Logical primary key:** `bronze_id`.

| Column | Type | Nullable | Description |
|---|---|---|---|
| period | STRING | Yes | Source month, normally YYYY-MM |
| duoarea | STRING | Yes | Source area code |
| area_name | STRING | Yes | Area description |
| product | STRING | Yes | Product code |
| product_name | STRING | Yes | Product description |
| process | STRING | Yes | Activity code |
| process_name | STRING | Yes | Activity description |
| series | STRING | Yes | Source series identifier |
| series_description | STRING | Yes | Series description |
| value | STRING | Yes | Original source value |
| units | STRING | Yes | Source measurement unit |
| bronze_id | STRING | No | Deterministic source-version identifier |
| record_hash | STRING | No | SHA-256 hash of the raw observation |
| source_system | STRING | No | Source system identifier, EIA |
| source_file | STRING | No | Raw file path |
| record_index | BIGINT | No | Zero-based position in the data array |
| batch_id | STRING | No | Input batch identifier |
| load_type | STRING | No | full_load or incremental_load |
| run_id | STRING | No | Ingestion execution identifier |
| load_timestamp | TIMESTAMP | No | Bronze ingestion time |

Although the Bronze business columns are nullable in the schema,
validation rejects null or empty `period` and `series`.

`bronze_id` incorporates the source system, batch ID, load type, file
path, record position, and raw record hash.

Processing time and run ID are excluded. Replaying an identical file
with the same parameters therefore does not create another Bronze row.

Changing the batch ID, file path, record position, or contents can
create a different source-version identifier.

## Silver Data Dictionary

**Grain:** one monthly observation per series.

**Logical primary key:** `(period, series)`.

| Column | Type | Nullable | Description |
|---|---|---|---|
| period | DATE | No | First day of the observation month |
| duoarea | STRING | Yes | Area code |
| area_name | STRING | Yes | Cleaned area description |
| product | STRING | Yes | Product code |
| product_name | STRING | Yes | Cleaned product description |
| process | STRING | Yes | Activity code |
| process_name | STRING | Yes | Cleaned activity description |
| series | STRING | No | Cleaned series identifier |
| series_description | STRING | Yes | Cleaned series description |
| value | DOUBLE | Yes | Validated numeric observation |
| units | STRING | Yes | Trimmed measurement unit |
| bronze_id | STRING | No | Accepted Bronze source-version identifier |
| record_hash | STRING | No | Hash of cleaned business fields |
| source_system | STRING | No | Source system identifier |
| source_file | STRING | No | Originating raw file path |
| batch_id | STRING | No | Batch supplying the stored observation |
| load_type | STRING | No | Source load type |
| bronze_load_timestamp | TIMESTAMP | No | Original Bronze ingestion time |
| run_id | STRING | No | Operation that inserted or updated the Silver row |
| load_timestamp | TIMESTAMP | No | Silver insertion or update time |

Logical uniqueness is enforced through validation, deduplication,
and MERGE logic rather than enforced database primary-key constraints.

## Gold Dimensional Model

Gold contains five dimensions, one detailed fact table, and one
aggregate fact table.

All table names below use the prefix
`workspace.default.petroleum_gold_`.

Counts describe the documented dataset snapshot, not guaranteed
counts for future loads.

| Table suffix | Type | Grain | Logical key | Recorded rows |
|---|---|---|---|---:|
| dim_date | Dimension | One calendar day | date_key | 8,035 |
| dim_product | Dimension | One product code | product_key | 58 |
| dim_area | Dimension | One source area code | area_key | 12 |
| dim_process | Dimension | One process code | process_key | 7 |
| dim_unit | Dimension | One measurement unit | unit_key | 2 |
| fact_observation | Detailed fact | One month and EIA series | observation_key | 678,572 |
| agg_monthly_product_process | Aggregate fact | One month, product, process, and unit | agg_key | 128,147 |

The date dimension covers complete calendar years from 2005 through
2026. Monthly observations reference the first day of their month.

The area dimension preserves original codes, including codes with
the same geographic label. Twelve source area codes do not represent
twelve distinct geographic regions.

### Keys and Relationships

- Date keys use the integer format `yyyyMMdd`.
- Product, area, process, and unit keys use deterministic SHA-256 hashes.
- `observation_key` identifies the business key `period + series`.
- The fact `record_hash` detects changes in analytical content.
- `agg_key` identifies the aggregate's four grouping keys.

There are five dimension-to-observation-fact relationships.

Date, Product, Process, and Unit also connect to the aggregate fact,
giving nine relationships in the analytical model.

The aggregate has no area key because regional observations have
already been combined.

Dimensions use SCD Type 1 behavior: changed descriptions replace
existing descriptions.

### Aggregate Implementation

The aggregate loader reads the Gold observation fact, selects area
codes beginning with `R`, and groups by:

- `date_key`
- `product_key`
- `process_key`
- `unit_key`

It stores:

- `padd_total_value`: sum of selected regional values
- `series_count`: number of fact rows included
- `load_timestamp`: insertion or update time

National area codes are excluded. The implementation preserves
separate source code families in the detailed fact; interpreting
regional sums requires checking that selected series are compatible
and non-overlapping.

This is a pipeline-maintained Delta aggregate table, not a native
materialized view.

The complete column-level definitions and ER diagram are available in
[Gold Data Dictionary](docs/gold_data_dictionary.md).

## Analytical Rules

- Keep MBBL quantities and MBBL/D daily averages separate.
- Do not add U.S. totals to PADD regional observations.
- Do not add product totals to their component products.
- Keep distinct production and other process measures separate.
- Do not sum ending stocks across months as an annual stock total.
- Preserve null values as missing observations rather than zero.
- Use compatible, non-overlapping observations for aggregated totals.
- Inspect original area codes when multiple codes share a label.

The dashboard is intended to be analyzed with a specific product
and a meaningful process selection. Selecting all products and
processes does not produce a universally meaningful petroleum total.

## MERGE and Idempotency

### Bronze

Bronze uses an insert-only MERGE on `bronze_id`.

### Silver

Normal processing merges on `(period, series)`:

- Insert missing observations.
- Update existing observations when the cleaned business hash changes.
- Leave unchanged records and their stored metadata untouched.

### Gold

- Dimensions update when their descriptive attributes change.
- Observation facts update when their analytical record hash changes.
- Aggregate rows update when the regional total or series count changes.
- Missing keys are inserted.

Stable identifiers exclude processing timestamps and run IDs.

The pipeline uses temporary SQL views as MERGE sources. These views
are session-scoped references to DataFrames, not persisted
materialized views.

Every execution creates new audit entries. Audit growth does not
violate business-data idempotency.

### Backfill Policy

`backfill_mode=True` enables insert-only Silver processing:

- Missing observations are inserted.
- Existing observations and metadata are preserved.

This prevents an old historical file from overwriting a revised
observation during backfill.

Backfill mode must be selected explicitly. The pipeline does not
automatically determine which raw snapshot is newer.

Corrections to existing observations require a reviewed batch
processed in normal mode.

## Quarantine and Execution Logging

Quarantine stores the rejected payload, rejection reason, source
file, record index, batch ID, load type, layer, run ID, and timestamp.

Execution logs contain:

- Log and run identifiers
- Batch ID and load type
- Processing layer
- Source file, table, or parameters
- Target table
- Start and end timestamps
- Success or failure status
- Rows read, inserted, updated, and quarantined
- Gold rows output and rows aggregated
- Error details
- Log creation timestamp

Pipeline timestamps use UTC.

A run may succeed while quarantining invalid records.

Malformed JSON files generate failure logs and raise an exception.
The raw batch wrapper attempts the other files and raises a batch
error if any file failed. Downstream processing does not proceed
after that error.

Completed table writes are not rolled back as a single transaction
across the entire pipeline. Recovery relies on audited, idempotent
reruns.

For one raw incremental file processed through all layers, the
expected audit coverage is:

| Operation | Entries |
|---|---:|
| End-to-End | 1 |
| Raw-to-Bronze | 1 |
| Bronze-to-Silver | 1 |
| Silver-to-Gold: five dimensions, fact, aggregate | 7 |
| Total | 10 |

Other file counts and starting layers change the expected total.

The aggregate loader currently records the full fact input count in
`rows_aggregated`. This is a workload metric; it is not an exact count
of regional contributing rows after the area filter.

## Pipeline Integration and Automation

`Petroleum_End_to_End_Pipeline` loads both shared core notebooks:

```python
%run ./Petroleum_Pipeline_Core
```

```python
%run ./Petroleum_Gold_Core
```

It then calls `run_petroleum_pipeline(...)`.

The processing order is:

1. Raw-to-Bronze, when starting from Raw.
2. Bronze-to-Silver, when starting from Raw or Bronze.
3. Gold dimensions: Date, Product, Area, Process, Unit.
4. Gold observation fact.
5. Gold aggregate.

Gold dimension and observation-fact preparation use one pinned
Silver Delta version. The aggregate reads the refreshed Gold fact.

Gold table writes are separate transactions. A fixed Silver snapshot
provides consistent source input but does not make the complete
multi-table refresh atomic.

The project demonstrates execution through a Databricks Job.
The documented configuration limits concurrent runs to one.

Job execution of saved batches is separate from automatic API
extraction and Power BI refresh.

## Execution Guide

### Prerequisites

- Databricks workspace with access to the raw Volume.
- Complete saved input files for the selected batch.
- Permissions to read and write the project tables.
- Bronze, Silver, quarantine, and execution-log tables initialized.
- Shared core notebooks available alongside the end-to-end notebook.

For a new workspace, complete the Phase 2 setup and restore the
required raw files first. The core notebooks are not a replacement
for every initial workspace setup step.

An EIA API key is needed for live extraction or revision checks,
but not for processing saved raw files.

### Normal Notebook Execution

In an initialized workspace, open
`Petroleum_End_to_End_Pipeline` and execute its cells in order.

It loads the shared definitions, reads parameters, runs processing,
and verifies audit evidence.

Do not rerun every development and controlled-test notebook before
each normal batch.

### Parameters

| Parameter | Purpose |
|---|---|
| raw_root | Root folder containing saved batches |
| batch_id | Saved batch identifier |
| load_type | full_load or incremental_load |
| start_layer | raw, bronze, or silver |
| backfill_mode | Preserve existing Silver observations when true |
| source_folder | Raw input folder passed to the orchestration function |

`source_folder` is required when starting from Raw.
The notebook constructs it from the raw root, batch ID, and load type.

### Full Load

After loading the core notebooks and defining the orchestration
function in the end-to-end notebook:

```python
selected_batch = "20261003T114452938244Z"
selected_root = (
    "/Volumes/workspace/default/petroleum_raw/phase1_expanded"
)

result = run_petroleum_pipeline(
    batch_id=selected_batch,
    load_type="full_load",
    start_layer="raw",
    source_folder=f"{selected_root}/{selected_batch}/full_load",
    backfill_mode=False,
)
```

Use normal mode only for the intended baseline or reviewed source
snapshot. Use backfill mode when replaying old historical data that
must not replace existing revisions.

### Standard Incremental Load

```python
result = run_petroleum_pipeline(
    batch_id=selected_batch,
    load_type="incremental_load",
    start_layer="raw",
    source_folder=(
        f"{selected_root}/{selected_batch}/incremental_load"
    ),
    backfill_mode=False,
)
```

For newly saved data, replace the batch identifier and use its
corresponding folder.

### Historical Backfill from Raw

```python
historical_batch = "20261003T114452938244Z"

result = run_petroleum_pipeline(
    batch_id=historical_batch,
    load_type="full_load",
    start_layer="raw",
    source_folder=(
        f"{selected_root}/{historical_batch}/full_load"
    ),
    backfill_mode=True,
)
```

### Resume from Existing Bronze

```python
result = run_petroleum_pipeline(
    batch_id=historical_batch,
    load_type="full_load",
    start_layer="bronze",
    backfill_mode=True,
)
```

This skips raw-file ingestion and processes the selected batch
already present in Bronze.

### Refresh Gold from Existing Silver

```python
result = run_petroleum_pipeline(
    batch_id=selected_batch,
    load_type="incremental_load",
    start_layer="silver",
    backfill_mode=False,
)
```

This skips Raw-to-Bronze and Bronze-to-Silver.

Gold refresh uses the complete current Silver snapshot. In this
mode, batch ID and load type describe the triggering operation;
they do not restrict Gold to that batch.

Use the original batch ID, file path, and load type when retrying
the same raw input.

## Power BI Dashboard and Semantic Model

Power BI Desktop connects directly to the Databricks SQL warehouse
using the Databricks connector in Import mode.

The model includes:

- Five Gold dimensions
- Detailed observation fact
- Monthly aggregate fact
- Execution logs
- Quarantine records

The star schema uses nine active one-to-many relationships from
dimensions to the two fact tables, with single-direction filtering.

Unneeded technical columns were removed from the Power BI fact
import to reduce model size.

### Supply Overview

- Year, Product, and Process slicers
- Latest-month volume KPI
- Month-over-month change KPI
- Monthly U.S. volume trends by process
- Regional PADD comparison

MBBL and MBBL/D are handled separately. Product totals and components
must not be combined indiscriminately.

### Pipeline Health

- Total logged operations
- Failed operations
- Success rate
- Average duration by layer
- Success/failure breakdown by layer
- Quarantined-record counts by reason
- Failed-operation details and error messages

The Total Runs metric counts execution-log entries, not necessarily
complete end-to-end executions.

Controlled tests and development failures are included in operational
history and should be interpreted accordingly.

### Refresh Behavior

Because Power BI uses Import mode, updates to Databricks tables do
not automatically update the imported model.

Refresh the Power BI model after a successful pipeline run.
Power BI Service publication and scheduled refresh are future work.

## Performance Optimization and Execution Analysis

Implemented or observed optimization includes:

- Liquid Clustering on Gold fact `date_key`, `product_key`, and `area_key`.
- OPTIMIZE executed on the observation fact and dimensions.
- A persisted aggregate table for precomputed summaries.
- Conditional MERGE updates that skip unchanged records.
- Reduced technical-column imports in Power BI.
- Broadcast joins selected by Spark for small dimensions.
- Column pruning in the fact-preparation scan.

The fact table was already approximately 46.6 MB in one file.
The recorded OPTIMIZE operation retained one file with the same size;
this does not demonstrate a substantial immediate performance gain.

Physical-plan and Query Profile evidence show five broadcast hash
joins in the inspected Gold fact-preparation query.

The large Silver side was not shuffled for those joins.
Other operations, such as grouping, duplicate checks, and MERGE,
can involve exchanges and additional scans.

The documented fact-preparation query took approximately 1.37 seconds.
This timing is for that query, not the whole pipeline.

The report uses Databricks Query Profile and the formatted physical
plan to explain execution in the serverless environment.

## Validation Evidence

Phase 2 evidence is available in:

- [Executed Phase 2 notebook](Petroleum_Phase2_Execution_Evidence.ipynb)
- [Selected Phase 2 audit logs](phase2_audit_evidence.csv)

The audit CSV contains 10 selected operations: nine successful
operations and one intentional failure.

| Test | Recorded result |
|---|---|
| Full raw load | 136 successful files; 675,752 records read |
| Full Silver rerun | 0 inserts and updates; no duplicate keys; metadata unchanged |
| January incremental load | 2,820 Silver inserts |
| Controlled revision | Value 30 changed to 31; 1 update, 0 inserts |
| Revised batch rerun | 0 inserts and updates; complete record unchanged |
| Mixed schema drift | 3 input records; 1 accepted, 2 quarantined |
| Malformed JSON | Failure logged; Bronze unchanged |
| Historical replay | Revised value 31 preserved in backfill mode |
| Missing-record backfill | Missing observation inserted |
| Backfill rerun | 0 inserts and updates; record unchanged |
| Gold fact reconciliation | 678,572 observations; keys unique and values matched Silver |
| Gold fact and aggregate rerun | 0 inserts and updates |
| One-file end-to-end audit coverage | 10 expected log entries verified |

### Controlled Revision Disclosure

A fresh December 2025 extraction was compared with 2,828 matching
saved observations. No revisions were found in those comparisons.

The demonstrated 30-to-31 change is therefore a clearly labeled
synthetic revision in isolated test tables.

It demonstrates MERGE update behavior and is not presented as an
actual EIA revision.

## Security

The reviewed source samples contain aggregate petroleum statistics,
not individual names, emails, account details, or other personal records.

No PII masking is currently required for the petroleum dataset.

Unexpected fields are rejected by the raw contract. Quarantine
retains rejected payloads, so future sensitive data would require
appropriate quarantine access and retention controls.

Execution evidence may contain workspace metadata and user identifiers.
Review exported evidence before public sharing.

EIA API keys and Databricks authentication tokens must not be committed.
Credentials should be entered or managed through the appropriate
runtime or connector credential mechanism.

## Cost and FinOps Evaluation

Development and execution used Databricks Free Edition.
The report records no incurred cloud charges.

Execution logs and job history were used to analyze runtime.

| Measurement | Documented result |
|---|---:|
| Mean incremental end-to-end notebook execution | Approximately 93 seconds across six recorded runs |
| Recorded Job wall-clock, including startup/setup | 9 minutes 7 seconds |
| Mean Raw-to-Bronze incremental file operation | Approximately 4.4 seconds |
| Mean Bronze-to-Silver incremental batch operation | Approximately 9.5 seconds |
| Mean Gold table operation | Approximately 8.6 seconds |

These measurements describe the recorded runs, including reruns,
and are not a guaranteed runtime for future new-data batches.

Gold refresh currently prepares data from the full Silver snapshot
and then aggregates the refreshed fact, even when Silver is unchanged.

The report estimates hypothetical annual costs for 8,760 hourly
executions using explicit DBU and price assumptions.

Those estimates are illustrative, not actual billing evidence.
Job wall-clock time is not automatically equivalent to billable
compute consumption.

### Implemented Cost-Awareness Practices

- Test with small samples and isolated tables.
- Reuse saved raw files.
- Avoid unnecessary complete baseline downloads.
- Record operation durations and row metrics.
- Reduce unnecessary columns in the BI import.

### Future Cost Optimizations

- Skip Gold refresh when Silver has no changes, when safe for recovery
  and model-maintenance requirements.
- Process changed Gold observations using Silver Change Data Feed
  or another reliable change-tracking strategy.
- Reduce repeated scans during dimension preparation.
- Match execution frequency to source publication and revision needs.
- Use measured billing usage for production cost estimates.

## Current Status and Limitations

### Implemented and Demonstrated

- Expanded Phase 1 raw data collection and validation
- Bronze and Silver processing with explicit schemas
- Quarantine and failure logging
- Conditional MERGE and idempotency demonstrations
- Controlled revised-value update
- Parameterized historical backfills and layer-based restarts
- Gold dimensions, observation fact, and regional aggregate
- Integrated Bronze-to-Gold orchestration
- Databricks Job execution
- Power BI Supply Overview and Pipeline Health pages
- Gold data dictionary and ER diagram
- Query execution analysis, optimization evidence, and FinOps evaluation

### Limitations and Future Work

- Automate fresh EIA API extraction and bounded revision checks.
- Implement change-driven Gold processing instead of full preparation.
- Add a product hierarchy to prevent overlapping product selections.
- Validate compatible source area-code families for regional summaries.
- Add a dedicated Net Imports measure and seasonal heatmap.
- Publish to Power BI Service and configure scheduled model refresh.
- Improve exact contributing-row metrics for aggregate audit logging.
- Define source-deletion handling if required; current MERGE processing
  focuses on inserts and updates.
- Maintain serialized pipeline execution and account for partial
  multi-table refreshes during recovery.

## Submission Deliverables

1. GitHub repository containing notebooks and documentation.
2. Power BI dashboard file (`.pbix`).
3. Final architectural PDF covering the data model, pipeline,
   engineering decisions, execution analysis, and FinOps evaluation.
