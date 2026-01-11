# DCGeoPulse ETL: Space-Weather Lakehouse Pipeline (Azure Databricks + Delta Lake)

An end-to-end, cloud-native **Azure Databricks + Delta Lake** **lakehouse** pipeline for **time-series ingestion** and **daily analytics datasets**, using public **space-weather** signals (GFZ geomagnetic indices + NMDB neutron monitor stations).

It standardises the sources into fixed **30-minute windows** (1800s) and aggregates them **into daily** analytics tables with coverage and integrity diagnostics.

## Motivation

Space weather can disrupt technology-dependent infrastructure (e.g., power systems, positioning/timing, communications). In highly interconnected environments, disruptions can cascade beyond a single site. This repository focuses on the data engineering foundation: ingesting public space-weather signals (GFZ geomagnetic indices and NMDB neutron monitor stations) and producing trustworthy daily datasets with coverage and integrity diagnostics.

**Data selection (Europe):** GFZ was chosen as a European source for geomagnetic indices. The NMDB stations were also selected intentionally: a Swiss high-altitude main station (`JUNG1`) plus two reference stations (`OULU`, `ROME`) for future comparison and baselining.

**Scope:** This project prepares datasets and diagnostics; it does not attempt event forecasting or asset-specific impact estimation.


## At a glance

* **Incremental ingestion** (bootstrap backfill + daily refresh)
* **Job orchestration on Azure Databricks** (scheduled daily; task dependencies Bronze → Silver → Gold)
* **Delta Lake** tables for Silver and Gold
* Canonical **long-form** schema for all sources
* **Idempotent re-runs** (key-based deduplication and merge-by-key semantics)
* **Resilient source ingestion** (retry with exponential backoff + jitter for transient network errors)
* Daily **data health diagnostics** (coverage %, gaps/overlaps, cross-midnight spillover)


## Tech stack

* **Azure Databricks** (Jobs + notebooks)
* **Apache Spark (PySpark)**
* **Delta Lake**
* **Azure Data Lake Storage Gen2 (ADLS)** via ABFSS paths


## Architecture

```
Public APIs (GFZ, NMDB)
        |
        v
Bronze  — raw exports (one file per run)
        |
        v
Silver  — Delta table (canonical long-form time-series)
        |
        v
Gold    — Delta tables (daily aggregates: long + optional wide)
```

### Why Bronze / Silver / Gold

* **Bronze** preserves raw extracts for auditability and replay.
* **Silver** normalises all sources to one schema for consistent validation and joins.
* **Gold** produces daily, analysis-ready datasets (reporting, dashboards, features).


## Notebooks (run order)

All notebooks live in `notebooks/` and are designed to run sequentially as Job tasks:

| Notebook         | Layer  | Purpose                                                                                                     |
| ---------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| `0_bronze.ipynb` | Bronze | Ingests from source endpoints and writes raw extracts; supports backfill + incremental refresh              |
| `1_silver.ipynb` | Silver | Reads new Bronze files, normalises to canonical schema, applies data-quality rules, writes Silver Delta     |
| `2_gold.ipynb`   | Gold   | Aggregates Silver into daily statistics and diagnostics; optionally pivots into region-oriented wide tables |


## Data model

### Silver (canonical long-form)

Each record represents one measurement window (fixed-length windows such as **30 minutes**):

| Column              | Type      | Meaning                                             |
| ------------------- | --------- | --------------------------------------------------- |
| `window_start_utc`  | timestamp | Window start (UTC)                                  |
| `source`            | string    | Source identifier (e.g., `gfz`, `nmdb`)             |
| `metric`            | string    | Metric name or station code (e.g., `Hp30`, `JUNG1`) |
| `value`             | double    | Observed value                                      |
| `window_duration_s` | int       | Window duration in seconds (e.g., `1800`)           |

### Gold (daily aggregates)

Gold Daily Long produces one row per `date_utc × source × metric` with:

* Daily summary stats (mean / min / max / stddev)
* Window counts and **coverage** derived from observed `window_duration_s`
* Intra-day interval diagnostics (**gaps** / **overlaps**)
* Cross-midnight detection (spillover seconds into the next day)

Gold Daily Wide (optional) pivots selected metrics into columns for quick consumption.


## Data quality and reliability

The pipeline implements common production checks for time-series ingestion and lightweight data observability:

* **Key-based deduplication** on `(window_start_utc, source, metric)`
* **Source-specific validity rules** (e.g., range checks; negative-value handling where appropriate)
* **Safe re-runs** by filtering keys already present in Silver
* Daily **health signals** in Gold (coverage %, gap/overlap metrics, cross-day spillover)


## Outputs

After a successful daily run you will have:

* **Bronze**: raw source extracts (traceable and replayable)
* **Silver (Delta)**: a canonical long-form time-series dataset
* **Gold (Delta)**: daily aggregates for analytics (long, and optionally wide)


## Repository structure

```
.
├─ notebooks/
│  ├─ 0_bronze.ipynb
│  ├─ 1_silver.ipynb
│  └─ 2_gold.ipynb
├─ LICENSE
├─ .gitignore
└─ README.md
```


## License

MIT (see `LICENSE`).


## References

* GFZ Potsdam (geomagnetic indices): [https://www.gfz-potsdam.de/](https://www.gfz-potsdam.de/)
* NMDB (Neutron Monitor Database) / NEST: [https://www.nmdb.eu/](https://www.nmdb.eu/)
