# U.S. Petroleum Supply Analysis

## Project Overview
This project studies U.S. petroleum production, imports, exports,
stocks, and product supplied as an indicator of consumption.
It includes national and PADD regional data.

We will use Databricks and Apache Spark to process the data through
Bronze, Silver, and Gold layers, then build a Power BI dashboard.

## Team
- Ayza Ahmed — 24L-2577
- Eman Adil — 24L-2589

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

The historical baseline was expanded by extending the history
and adding regional and stock-related series.

The complete dataset is stored in a Databricks Volume.
This repository contains selected raw full-load pages and the
January 2026 incremental payload, rather than all 136 historical files.

The summary and validation reports describe the complete saved loads.

## Ingestion Pattern and Expected Volume
- Full load: Extract the historical baseline for 2005–2025.
- Incremental load: Fetch newly published monthly observations.
- Revisions: Re-fetch recent periods and update existing observations
  when their values change.

Based on the January 2026 sample, one new month is expected to contain
approximately 2,800 records and around 1 MB of raw JSON.
Actual volume will vary by month and available series.

Weekly checks are planned for new releases and revisions.
A check may find no new month. Re-fetching recent months for revisions
will increase the payload beyond the new-month estimate.

## Repository Contents
- full_load/: Two selected raw pages from the historical baseline:
  page_0001.json and page_0136.json.
- incremental_load/: January 2026 raw payload.
- file_summary.csv: File names, record counts, and sizes for the
  complete saved loads.
- validation_summary.csv: Record counts, date ranges, sizes,
  duplicate checks, scope checks, and validation results.
- SAMPLE_NOTE.txt: Explanation of the sample selection.

The full_load folder is a sample of the baseline, not a complete
historical dataset.

## Data Scope
The dataset contains monthly petroleum observations for the
United States and PADD regions 1–5.

Included activities:
- Field production
- Refinery and blender net production
- Imports
- Exports
- Product supplied
- Ending stocks
- Stock change

Trading-partner country details are outside the current scope.

## Validation Results
Both saved loads passed the implemented checks:
- Record counts match the source totals.
- No duplicate period-and-series observations.
- No observations outside the selected date ranges.
- No observations outside the selected area and process filters.

Further value and data-type checks are planned for the Silver layer.

## Planned Processing

### Bronze
Preserve source records and add ingestion metadata, including
load_timestamp, source file, and batch identifier.

### Silver
Create a cleaned observation table with one row per monthly
series and period.

Planned fields include period, series, area code and name,
product code and name, process code and name, value, units,
and load_timestamp.

Processing will include:
- Explicit schemas and controlled date and numeric casting.
- Missing-value and invalid-value checks.
- Duplicate handling using period and series as the business key.
- Consistent column names and unit labels.
- MERGE-based inserts and updates for new and revised observations.

### Gold
Create an observation fact table linked to date, product, area,
and process dimensions, with product-level monthly and annual
summaries for dashboard analysis.

Aggregation rules:
- MBBL quantities and MBBL/D daily averages will remain separate.
- Monthly quantity analysis will use MBBL.
- Ending stocks will use period-end values, not sums across months.
- Production measures will remain distinct.
- Product totals will not be added to their subcategories.
- National totals will not be added to PADD regional figures.
- Annual flow totals will use compatible, non-overlapping monthly data.

## Planned Dashboard
The Power BI dashboard will include:
- Line charts of production and product supplied to show changes
  in supply and estimated consumption.
- Import-versus-export charts by petroleum product to show
  changes in net imports.
- Regional stock trends and PADD comparisons to show how
  inventories differ across regions and over time.

Filters will include period, product, and region.

## Security
The reviewed samples contain aggregate petroleum statistics,
not personal information such as names, emails, or account details.
Area and product names describe statistical categories, not individuals.

No PII masking is currently required. Any unexpected sensitive
fields will be excluded or quarantined before the Silver layer.

API keys will not be committed to this repository.

## Compute and Storage Plan
Development will use Databricks Free Edition.

We will test transformations on small samples before processing
the complete baseline, reuse saved raw files, and avoid unnecessary
full downloads and repeated compute runs.

Incremental processing will focus on new months and a limited
revision window. Storage use will be monitored, and unnecessary
temporary outputs will be removed.

## Current Status
Phase 1: Expanded historical data and an incremental sample
have been collected and validated. Selected raw samples and
supporting reports are included in this repository.

Bronze, Silver, and Gold transformations and the Power BI
dashboard are planned.

Phase 2 will implement the PySpark pipeline, explicit schemas,
parameterized backfills, schema-drift handling, and audit logging.
It will also demonstrate idempotent MERGE processing and at
least one update to an existing observation with a revised value.
