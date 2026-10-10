# U.S. Petroleum Supply Analysis

## Project Overview

This project studies U.S. petroleum production, imports, exports,
stocks, and product supplied as an indicator of consumption.
It includes national and PADD regional data.

Phase 2 implements Bronze and Silver processing using PySpark and
Delta tables in Databricks Free Edition. Gold modeling and a Power BI
dashboard are planned for a later phase.

## Team

- Ayza Ahmed — 24L-2577
- Eman Adil — 24L-2589

## Repository

https://github.com/ayzaahmed27/petroleum-supply-analysis

## Data Source

U.S. Energy Information Administration (EIA), API v2.

Dataset: Petroleum — Supply and Disposition

https://api.eia.gov/v2/petroleum/sum/snd/

Data endpoint:

https://api.eia.gov/v2/petroleum/sum/snd/data/

## Data Collected

| Load | Period | Records | Raw JSON Size | Files |
|---|---|---:|---:|---:|
| Full load | January 2005–December 2025 | 675,752 | 236.33 MB | 136 |
| Incremental sample | January 2026 | 2,820 | 0.99 MB | 1 |

The historical baseline was expanded by extending the history and
adding regional and stock-related series.

The complete raw dataset is stored in a Databricks Volume.
The repository contains selected historical pages and the January 2026
incremental payload. The summary reports describe the complete saved loads.

### Ingestion Pattern

- Full load: historical baseline extraction.
- Incremental load: newly published monthly observations.
- Revisions: re-fetch recent periods and apply changed observations.

One new month is estimated to contain approximately 2,800 records and
1 MB of raw JSON, based on the January 2026 sample. Actual volume varies.

Weekly source checks are planned. Re-fetching recent months for revisions
will increase the payload beyond the new-month estimate. Scheduling these
checks is not implemented in the current notebook.

## Repository Contents

- `Petroleum_Phase2_Pipeline`: Databricks notebook containing pipeline
  functions and validation demonstrations.
- `full_load/`: selected historical pages, `page_0001.json` and
  `page_0136.json`.
- `incremental_load/`: January 2026 raw payload.
- `file_summary.csv`: file names, record counts, and sizes.
- `validation_summary.csv`: source extraction validation results.
- `SAMPLE_NOTE.txt`: explanation of sample selection.

The `full_load/` folder is a sample, not the complete historical dataset.
The full-baseline execution requires the complete raw files in the Volume.

## Data Scope

Monthly observations cover the United States and PADD regions 1–5.

Included activities:

- Field production
- Refinery and blender net production
- Imports
- Exports
- Product supplied
- Ending stocks
- Stock change

Trading-partner country analysis is outside the current scope.

## Pipeline Architecture

1. Raw JSON files are retained in a Databricks Volume.
2. Raw observations are validated against an explicit contract.
3. Accepted observations are merged into Bronze with provenance metadata.
4. Bronze observations are cleaned, cast, and checked for conflicting keys.
5. Valid observations are merged into Silver.
6. Rejected observations are saved to quarantine.
7. Each file or layer-processing run writes an execution log.

The raw Volume folders serve as the landing area. Bronze retains accepted
source versions. Silver contains one current observation per business key.

### Main Tables

| Purpose | Table |
|---|---|
| Bronze | `workspace.default.petroleum_bronze` |
| Silver | `workspace.default.petroleum_silver` |
| Quarantine | `workspace.default.petroleum_quarantine` |
| Execution logs | `workspace.default.petroleum_execution_logs` |

Controlled tests use separate tables to keep synthetic data out of the
main Bronze and Silver tables.

## Schema Enforcement and Validation

Raw schemas are defined using `StructType` and `StructField`.
Schema inference is not used.

The raw contract requires all 11 expected source fields. Values must be
strings or null, and `period` and `series` must be nonempty.

The pipeline quarantines records containing:

- Unexpected or missing fields
- Incompatible raw value types
- Missing observation keys
- Invalid monthly periods
- Invalid numeric values, including NaN and infinity
- Conflicting observations for the same Silver key within a batch

Raw source names such as `area-name` are mapped to names such as
`area_name`.

Silver processing trims strings, converts monthly periods to the first
day of the month, and casts numeric values to doubles. Null source values
remain null. Negative values are permitted because stock changes can be
negative.

Identical observations within a batch are deduplicated. Conflicting
observations are quarantined instead of choosing an arbitrary value.

The implemented schema-drift policy is rejection and quarantine;
automatic schema evolution is not enabled.

## Bronze Data Model

**Grain:** one accepted source observation version within its batch,
file, and record position.

**Logical primary key:** `bronze_id`, enforced through the ingestion MERGE.

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

Although Bronze business columns are nullable in the schema, raw
validation rejects missing or empty `period` and `series` values.

`bronze_id` incorporates the source system, batch ID, load type, file path,
record position, and raw record hash. It excludes processing time and
run ID, so an identical file replay with the same parameters does not
create another Bronze row.

Changing the batch ID, path, or record contents creates a different
source-version identifier.

## Silver Data Model

**Grain:** one monthly observation per series.

**Logical primary key:** `(period, series)`, enforced through validation,
deduplication, and MERGE logic.

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
| units | STRING | Yes | Trimmed source unit label |
| bronze_id | STRING | No | Identifier of the accepted Bronze version |
| record_hash | STRING | No | Hash of the cleaned business fields |
| source_system | STRING | No | Source system identifier |
| source_file | STRING | No | Originating raw file path |
| batch_id | STRING | No | Batch supplying the stored observation |
| load_type | STRING | No | Source load type |
| bronze_load_timestamp | TIMESTAMP | No | Original Bronze ingestion time |
| run_id | STRING | No | Silver execution that inserted or updated the row |
| load_timestamp | TIMESTAMP | No | Silver insertion or update time |

The logical keys are pipeline rules, not database primary-key constraints.

## MERGE and Idempotency

Bronze uses an insert-only MERGE on `bronze_id`.

Normal Silver processing uses MERGE on `(period, series)`:

- Missing keys are inserted.
- Existing keys are updated only when the cleaned business hash changes.
- Unchanged records are left untouched, including their timestamps and
  stored run metadata.

Every execution creates a new audit entry. This does not change the
idempotency of the Silver data.

### Backfill Policy

`backfill_mode=True` enables insert-only Silver processing:

- Missing observations are inserted.
- Existing observations and their metadata are preserved.

This prevents historical replays in backfill mode from replacing an
existing revised observation.

Backfill mode is explicit. The pipeline does not automatically determine
which source snapshot is newer. Applying corrections to existing rows
requires a reviewed batch processed in normal mode.

## Quarantine and Execution Logging

Quarantine records retain the original payload, rejection reason,
source file, record index, batch ID, load type, layer, run ID, and
load timestamp.

Execution logs contain:

- Log and run identifiers
- Batch ID and load type
- Processing layer
- Source file or processing parameters
- Target table
- Start and end timestamps
- Success or failure status
- Rows read, inserted, updated, and quarantined
- Error details
- Log creation timestamp

Pipeline timestamps use UTC.

Malformed files produce failure logs and raise an exception.
The raw batch function continues processing other files and reports
failed files at the end. If any file failed, it raises a batch error.

A run can succeed while quarantining invalid records. If parsing fails
before records can be counted, `rows_read` is unknown and recorded as null.

## Execution Guide

### Prerequisites

- Access to the Databricks workspace and raw Volume.
- Saved raw files for the batch being processed.
- Permissions to read and write the project Delta tables.
- Notebook imports, schemas, table setup, and helper functions loaded
  in the current session.

An EIA API key is needed for live revision checks, but not for processing
saved raw files.

After a session restart, rerun setup and function-definition cells before
calling the pipeline. Saved tables and files persist, but Python variables
and function definitions must be restored.

The notebook contains demonstrations and live API checks as well as
pipeline definitions. Do not use Run All as a shortcut for a single backfill.

### Parameters

| Parameter | Purpose |
|---|---|
| raw_root | Root folder containing saved batches |
| batch_id | Identifies the input batch |
| load_type | full_load or incremental_load |
| folder_path / file_path | Selects raw files to process |
| bronze_table | Optional target/source override on individual layer functions |
| silver_table | Optional Silver target override |
| backfill_mode | Preserve existing Silver rows when True |

The raw batch wrapper uses the main Bronze table. The individual raw-file
function supports a Bronze target override.

### Full Load

```python
selected_batch = "20261003T114452938244Z"
selected_root = Path(
    "/Volumes/workspace/default/petroleum_raw/phase1_expanded"
)

ingest_raw_batch_to_bronze(
    selected_root / selected_batch / "full_load",
    batch_id=selected_batch,
    load_type="full_load",
)

process_bronze_to_silver(
    batch_id=selected_batch,
    load_type="full_load",
)
```

### Standard Incremental Load

```python
ingest_raw_batch_to_bronze(
    selected_root / selected_batch / "incremental_load",
    batch_id=selected_batch,
    load_type="incremental_load",
)

process_bronze_to_silver(
    batch_id=selected_batch,
    load_type="incremental_load",
    backfill_mode=False,
)
```

For a new saved batch, set `selected_batch` to its identifier and point
to its corresponding folder.

### Historical Backfill from Raw

```python
historical_batch = "20261003T114452938244Z"

ingest_raw_batch_to_bronze(
    selected_root / historical_batch / "full_load",
    batch_id=historical_batch,
    load_type="full_load",
)

process_bronze_to_silver(
    batch_id=historical_batch,
    load_type="full_load",
    backfill_mode=True,
)
```

### Start Directly from Bronze

If the historical batch is already in Bronze, skip raw ingestion:

```python
process_bronze_to_silver(
    batch_id=historical_batch,
    load_type="full_load",
    backfill_mode=True,
)
```

Use the original batch ID, file path, and load type when retrying the
same raw input.

## Validation Evidence

The notebook includes saved outputs for the following demonstrations:

| Test | Observed result |
|---|---|
| Full raw load | 136 successful files, 675,752 records read |
| Full Silver rerun | 0 inserts, 0 updates, 0 duplicate keys; all values and timestamps unchanged |
| January incremental load | 2,820 Silver inserts |
| Controlled revision | Value changed from 30 to 31; 1 update, 0 inserts |
| Revised batch rerun | 0 inserts and updates; complete Silver record unchanged |
| Mixed schema drift | 3 input records; 1 accepted, 2 quarantined |
| Malformed JSON | Failure logged; 0 inserts and updates |
| Historical replay | Revised value 31 preserved in backfill mode |
| Missing-record backfill | 1 insert into an empty test Silver table |
| Backfill rerun | 0 inserts and updates; complete record unchanged |

The full source extraction also passed record-count, date-range,
scope, and duplicate-key checks.

### Revision Test Disclosure

A fresh December 2025 extraction was compared against 2,828 matching
saved observations. No revisions were found in those comparisons.

The demonstrated 30-to-31 change is therefore a clearly labeled
synthetic revision in isolated test tables. It demonstrates the
pipeline's update behavior and is not presented as an actual EIA revision.

## Planned Gold Layer and Dashboard

Gold will contain an observation fact table linked to date, product,
area, and process dimensions, with suitable monthly and annual summaries.

Aggregation rules:

- Keep MBBL quantities and MBBL/D daily averages separate.
- Use period-end values for ending stocks rather than summing months.
- Keep production measures distinct.
- Do not add product totals to their subcategories.
- Do not add national totals to PADD regional figures.
- Use compatible, non-overlapping monthly data for annual flow totals.

The planned Power BI dashboard will show production and product-supplied
trends, imports versus exports, and regional stock comparisons.

## Security

The reviewed samples contain aggregate petroleum statistics, not
personal names, emails, account details, or other individual-level data.
Area and product names describe statistical categories.

No PII masking is currently required. Unexpected fields are rejected
by the raw contract. Quarantine retains rejected payloads, so any future
sensitive source data would also require appropriate quarantine access
and retention controls.

API keys are entered through a hidden prompt and must not be committed.

## Compute and Storage Plan

Development uses Databricks Free Edition.

Small samples and isolated test tables are used before full processing.
Saved raw files are reused to avoid repeated API downloads.
Full-baseline reruns are avoided when a targeted batch test is sufficient.

Storage and compute usage should be monitored. Temporary test tables
can be removed after required evidence has been retained.

## Current Status

Phase 1 data collection and validation are complete.

Phase 2 Bronze and Silver processing, explicit schema validation,
quarantine handling, audit logging, MERGE demonstrations, and
parameterized backfill tests have been implemented and demonstrated.

Gold processing, dashboard development, and scheduled ingestion
remain planned.
