# petroleum-supply-analysis
U.S. petroleum production, imports, exports and consumption analysis using Databricks.
# U.S. Petroleum Supply Analysis

## Project Overview
This project studies U.S. petroleum production, imports, exports,
and product supplied as an indicator of consumption.

We will use Databricks and Apache Spark to process the data through
Bronze, Silver, and Gold layers, then build a Power BI dashboard.

## Data Source
U.S. Energy Information Administration (EIA), API v2.

Dataset: Petroleum — Supply and Disposition
https://api.eia.gov/v2/petroleum/sum/snd/

## Data Collected
- Full load: January 2015 to December 2025
- Full-load records: 49,124
- Full-load JSON size: approximately 16.66 MB
- Incremental sample: January 2026
- Incremental records: 372
- Incremental JSON size: approximately 0.126 MB

The incremental sample demonstrates loading a later month after
the historical baseline. Future loads will collect newly published
months and recheck recent periods for revisions.

## Repository Contents
- full_load/: Historical raw JSON response data
- incremental_load/: Subsequent-month raw JSON response data
- file_summary.csv: File names, record counts, and file sizes

## Data Scope
The current dataset contains U.S. national-level monthly records
for petroleum products and the following activities:
- Field production
- Refinery and blender net production
- Imports
- Exports
- Product supplied

Trading-partner country details and regional breakdowns are
outside the current scope.

## Planned Processing
- Bronze: Preserve raw records and add ingestion metadata.
- Silver: Cast dates and numbers, check missing values, handle
  duplicates, and standardize units.
- Gold: Create monthly and annual product-level summaries for
  dashboard analysis.

Raw data includes both monthly quantities (MBBL) and daily averages
(MBBL/D). These will not be added together. Monthly quantity
analysis will use MBBL.

Production measures will remain distinct. Product totals and their
subcategories will not be added together.

## Planned Dashboard
- Production and product-supplied trends over time
- Imports versus exports by selected petroleum product
- Seasonal patterns in product supplied

## Security
The reviewed samples contain aggregate petroleum statistics,
not personal information such as names, emails, or account details.
API keys will not be committed to this repository.

## Compute and Storage Plan
Development will use small samples and saved files to avoid
repeated API downloads and unnecessary compute runs.
The current dataset is modest in size; Spark is used to demonstrate
the required data-engineering workflow.

## Current Status
Phase 1: Historical and incremental raw data collected.
Pipeline transformations and the dashboard are planned.
