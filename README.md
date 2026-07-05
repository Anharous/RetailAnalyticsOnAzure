# Enterprise Retail Analytics Platform on Azure

An end-to-end retail data pipeline built on **Azure Databricks, PySpark, Delta Lake, and ADLS Gen2**, following a **Medallion Architecture (Bronze → Silver → Gold)**. The pipeline is **metadata-driven** — table behavior (source paths, primary keys, SCD type, validation rules, column schemas) is controlled through a single JSON config file rather than hardcoded per table.

## Architecture

**Azure Databricks (PySpark) → Delta Lake → ADLS Gen2**

| Layer | Purpose |
|---|---|
| **Bronze** | Raw ingestion from landing files into Delta tables, tagged with ingestion metadata (`_IngestionTimestamp`, pipeline run id) |
| **Silver** | Cleansing and conformance — type casting, schema validation, primary/foreign key validation with reject logging, watermark-based incremental loads, and SCD Type 1 / Type 2 / append-only handling depending on the table |
| **Gold** | Business-ready output — fact/dimension selection, currency-converted revenue figures, and store/customer sales summaries |

Key design elements:
- **Metadata-driven config (`metaData_config.json`)** — each table's landing path, Bronze/Silver/Gold paths, primary key, SCD type, column list, and validation rules are defined here
- **Watermark-based incremental loading** so re-running a notebook only processes new records
- **SCD Type 1, Type 2, and append-only** load strategies, selected per table via config
- **Reject logging & audit trail** — invalid records are written to a `rejects/` path and each run is logged with read/valid/rejected counts

> Note: this repo covers the transformation logic (Bronze/Silver/Gold notebooks) that would normally be orchestrated by a scheduler like Azure Data Factory, and there's no dashboard code in this repo — the notebooks are run manually against ADLS Gen2.

## Repository Structure

```
RetailAnalyticsOnAzure/
├── config.ipynb                  # Storage account/container settings + loads metaData_config.json
├── metaData_config.json          # Metadata driving all pipeline logic (per-table config)
├── data_gen.ipynb                # Generates sample CSV data into the ADLS landing folders
├── retail_bronze_layer.ipynb     # Landing → Bronze ingestion
├── retail_silver_layer.ipynb     # Bronze → Silver: validation, casting, SCD1/SCD2/append-only
├── retail_gold_layer.ipynb       # Silver → Gold: fact/dim selection, revenue calc, sales summaries
└── retail_unit_tests.ipynb       # Unit test suite across Bronze/Silver/Gold functions
```

## Tech Stack

- **Azure Databricks** – notebook/compute environment
- **PySpark** – data processing
- **Delta Lake** – storage format with ACID transactions
- **Azure Data Lake Storage Gen2 (ADLS Gen2)** – underlying storage

## Setup & Prerequisites

Before running anything, you'll need:

1. **An Azure Databricks workspace** to import and run the notebooks
2. **An ADLS Gen2 storage account + container**, with this folder structure created inside it:
   ```
   <container>/
   ├── configs/metaData_config.json     # upload this repo's file here
   ├── landing/orders/
   ├── landing/products/
   ├── landing/customers/
   ├── landing/exchange_rates/
   ```
   (the `bronze/`, `silver/`, `gold/`, `rejects/`, and `metadata/` folders are created automatically by the notebooks on first run)
   
3. **Update `config.ipynb`** with your own values — this is the only place credentials/paths are set:
   ```python
   storage_account_name = "your_storage_account_name"
   storage_account_key  = "your_storage_account_key"
   container_name        = "your_container_name"
   ```

## Running the Pipeline

Run the notebooks in this order:

1. `data_gen.ipynb` *(optional)* — generates sample CSVs into the `landing/` folders if you don't have your own source data yet
2. `retail_bronze_layer.ipynb` — loads each table from `landing/` into `bronze/`
3. `retail_silver_layer.ipynb` — validates, casts, and applies SCD logic into `silver/`
4. `retail_gold_layer.ipynb` — builds the Gold fact/dimension/summary tables
5. `retail_unit_tests.ipynb` — run any time to verify the transformation functions independently

Each notebook starts with `%run ./config`, so `config.ipynb` must be updated first and must sit in the same workspace folder as the other notebooks.

## Adding a New Dataset

Since the pipeline is metadata-driven, most of onboarding a new table is a config change, not a code change:

1. **Add a folder** under `landing/<new_table>/` in your ADLS container
2. **Add an entry to `metaData_config.json`** for the new table, following the existing pattern (see `products` for the simplest example) — specify `landing_path`, `bronze_path`, `silver_path`, `gold_path`, `primary_key`, `scd_type` (`append_only`, `type1`, or `type2`), `columns` (with types, and `date_formats` for date/timestamp columns), `foreign_keys`, and `validations`
3. **Add one call per notebook** for the new table name:
   - In `retail_bronze_layer.ipynb`: `process_landing_to_bronze("new_table")`
   - In `retail_silver_layer.ipynb`: `process_bronze_to_silver("new_table")`
   - In `retail_gold_layer.ipynb`: currently the Gold layer has a couple of table-specific branches (e.g. the `orders` revenue/currency calculation, and the `tracked_columns` list inside `scd2_load` for SCD2 change detection), so a genuinely new table with its own business logic in Gold, or a new SCD2 table, will need a small code addition there — everything upstream of that (Bronze and most of Silver) needs no code changes at all.

## 📌 Project Context

This project was built as a capstone for a structured Azure data engineering training program, applying medallion architecture, SCD handling, a metadata-driven framework, and automated unit testing to a retail analytics use case.

