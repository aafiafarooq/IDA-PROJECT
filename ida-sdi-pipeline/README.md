# IDA Project – Phase 1: Sales & Demand Intelligence (SDI) pipeline

Part 1 of the pipeline: **data generator → file watcher → Postgres staging**.
Aafia's transformation (DuckDB star schema) and the query layer plug in after the
`staging_ready` task in `dags/sdi_ingest_pipeline_dag.py`.

## What's in here
| Path | What it does |
|---|---|
| `docker-compose.yaml` | Starts Airflow, our Postgres database (`pharma-db`) and the ingestion API |
| `api/main.py` | Data Ingestion API: upload new CSV files into the pipeline |
| `sql/init_staging.sql` | Creates the `staging` tables and `ops` monitoring tables (runs automatically) |
| `dags/sdi_generate_data_dag.py` | DAG 1 – oversampling layer, runs every 30 min (makes data for the next period) |
| `dags/sdi_ingest_pipeline_dag.py` | DAG 2 – file watcher + validation + load into staging (runs every 5 min) |
| `dags/sdi_lib/generate_data.py` | Generator logic |
| `dags/sdi_lib/ingest.py` | Validation + loading logic + monitoring |
| `dags/sdi_lib/storage_metrics.py` | Records table sizes, row counts and date coverage |

## Setup (once)
1. Install and open **Docker Desktop** (WSL 2 option ticked). Wait for "Engine running".
2. Copy these 3 files into `data/source/`:
   `01_master_data.csv`, `05_pharmacy_sales_data.csv`, `06_hospital_demand_data.csv`
3. Open a terminal in this folder and run:
   ```
   docker compose up -d
   ```
   First time takes ~5–10 minutes (downloads Airflow).
4. Open http://localhost:8080 → user `airflow`, password `airflow`.

## Running the pipeline (fully automatic)
Nothing to click. Once Docker is running:
- **sdi_generate_data** runs every 30 minutes. First run sends the original data through,
  then each run generates the next quarter: Jan-Mar 2026, Apr-Jun, Jul-Sep, Oct-Dec.
  It waits if earlier files are not ingested yet, and stops after Oct-Dec 2026 (4 refreshes).
- **sdi_ingest_pipeline** checks `data/incoming` every 5 minutes and loads any new files.

Manual option (for the demo): open **sdi_generate_data** -> Trigger DAG w/ config ->
set `refresh_no` (0 = original data, 1-4 = a quarter of 2026). Leave it at -1 for "next refresh".

## Data Ingestion API
Open **http://localhost:8000/docs** for a web page where you can try every endpoint.

| Endpoint | What it does |
|---|---|
| `GET /health` | Checks the API is running |
| `POST /ingest/{source}` | Upload a CSV (`master`, `sales` or `hospital_demand`). Checks the file type and columns, then puts it in `data/incoming` |
| `GET /ingest/pending` | Files waiting for Airflow to load |
| `GET /ingest/status` | Latest load results (duration, rows, rejected, status) |

Flow: API receives file -> `data/incoming` -> Airflow file watcher -> validation -> Postgres staging.
The automatic generator writes into the same folder, so both routes go through the same checks.

## Data engineering dashboards (Grafana)
Open **http://localhost:3000** (viewing needs no login; `admin` / `admin` to edit).
Both dashboards are set up automatically from `grafana/` and refresh every 10 seconds,
so they update on screen as soon as Airflow loads a new file.

| Dashboard | Shows | Source tables |
|---|---|---|
| **SDI - Batch Ingestion** | Batch success rate %, load duration per source and per file, rows processed per batch (loaded / rejected / duplicate), rejected rows by reason, rows generated per refresh, load log | `ops.batch_load_log`, `ops.rejected_rows`, `ops.generation_log` |
| **SDI - Storage & Staging** | Total staged rows, database size (MB), share of rows and MB per source (donuts), date coverage per table, cumulative row growth after each refresh | `ops.storage_metrics`, `ops.batch_load_log` |

To change a dashboard: edit it in the browser, then **Share → Export → Save to file** and
replace the JSON in `grafana/dashboards/`.

## Looking at the data
Connect any SQL tool (pgAdmin, DBeaver, VS Code, Power BI) to:
host `localhost`, port `5433`, database `pharma_dwh`, user `pharma`, password `pharma`.

| Table | Contents |
|---|---|
| `staging.stg_master` | Medicines (price changes add a new row → SCD Type 2) |
| `staging.stg_sales` | Sales transactions |
| `staging.stg_hospital_demand` | Monthly hospital demand |
| `ops.batch_load_log` | One row per file: duration, rows read/loaded/rejected, SUCCESS/FAILED |
| `ops.rejected_rows` | Rows that failed validation + reason |
| `ops.generation_log` | Rows made by the generator per refresh |
| `ops.storage_metrics` | After every load: row count, size in MB, first/last date per table |

Every staging row has `batch_id` (which run loaded it), `source_file` and `loaded_at`.

## Validation rules
- All files: `MedicineID` not empty
- Sales: valid `SalesDate`, `UnitRetailPriceUSD > 0`, `QuantitySoldPacks > 0`, ID not empty
- Hospital demand: valid `DemandMonth`, `HospitalID` not empty, quantities not negative
- Master: valid `EffectiveFromDate`, all prices > 0
- Duplicate IDs are skipped (loading the same file twice is safe)

## Data generation logic (for the report)
- Each refresh = one new 3-month period after the original data (ends Dec 2025):
  refresh 1 = Jan–Mar 2026, 2 = Apr–Jun, 3 = Jul–Sep, 4 = Oct–Dec 2026.
- Sales and hospital demand copy the same months from one year earlier (keeps
  seasonality, e.g. the January spike), shift dates forward, change quantities
  by ±15% (sales) / ±10% (demand), and recalculate money columns so
  Net = Gross − Discount and Margin = Net − COGS still hold.
- Master: 5 medicines per refresh get a 3–10% price increase from the period
  start (tests SCD Type 2).
- ~0.2% of generated rows are broken on purpose (empty MedicineID, invalid date,
  negative price/quantity) so validation and the rejected-rows metric are tested.

Tested result (original + 2 refreshes):

| Source | Original | Refresh 1 | Refresh 2 |
|---|---|---|---|
| Master rows | 63 | +5 | +5 |
| Sales rows generated | 500,000 | 112,794 (225 rejected) | 18,737 (37 rejected) |
| Hospital demand rows | 27,216 | 2,268 (4 rejected) | 2,268 (4 rejected) |

Sales load (500k rows) ≈ 15–20 s.

## Useful check commands (run in this folder)
Load results per file:
```
docker exec ida-sdi-pipeline-pharma-db-1 psql -U pharma -d pharma_dwh -c "select source_name, rows_read, rows_loaded, rows_rejected, duration_sec, status from ops.batch_load_log"
```
Why rows were rejected:
```
docker exec ida-sdi-pipeline-pharma-db-1 psql -U pharma -d pharma_dwh -c "select source_name, reason, count(*) from ops.rejected_rows group by 1,2 order by 1,2"
```
Add the indexes to an existing database (new setups get them automatically):
```
docker exec -i ida-sdi-pipeline-pharma-db-1 psql -U pharma -d pharma_dwh < sql/indexes.sql
```
(In PowerShell use: `Get-Content sql/indexes.sql | docker exec -i ida-sdi-pipeline-pharma-db-1 psql -U pharma -d pharma_dwh`)

## Report
`docs/Data_Generation_Report.pdf` – data generation logic, statistics, ingestion method, assumptions.

## Stop / reset
- Stop: `docker compose down` (data kept)
- Wipe everything and start fresh: `docker compose down -v`
