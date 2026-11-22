# Intelligent Data Architectures (IDA) — Fall 2026
## Phase 1: Sales & Demand Intelligence (SDI) Data Mart
**Repository:** [https://github.com/aafiafarooq/IDA-PROJECT](https://github.com/aafiafarooq/IDA-PROJECT)  
**Milestone 1 Deliverable:** Implementation through Transformation, Dimensional Modeling & Analytical Query Layer

---

### 1. Project Overview & Architecture
This repository implements the **Sales & Demand Intelligence (SDI) Data Mart** for an enterprise-level Pharmaceutical Supply Chain using **DuckDB** and **Kimball Dimensional Modeling**.

The platform unifies 3 siloed operational systems:
1. **Medicine Master Data (`01_master_data.csv`):** 63 medicines across 7 therapeutic categories.
2. **Pharmacy POS & E-commerce Sales (`05_pharmacy_sales_data.csv`):** 500,000 retail, hospital, and e-commerce transactions.
3. **Hospital Demand & Inventory Logs (`06_hospital_demand_data.csv`):** 27,216 monthly logs across 12 hospitals.

```
                      +-----------------------------+
                      | Raw Sources (01, 05, 06)   |
                      +--------------+--------------+
                                     |
                                     v
                      +-----------------------------+
                      | Staging Views & Data Quality|
                      | (stg_sales, stg_demand...)  |
                      +--------------+--------------+
                                     |
                    +----------------+----------------+
                    |                                 |
                    v                                 v
       +--------------------------+     +--------------------------+
       |   Dimension Tables (SCD) |     |       Fact Tables        |
       |--------------------------|     |--------------------------|
       | DimDate (2023-2029)      |     | Fact_Sales (500K rows)   |
       | DimMedicine (SCD 1 & 2)  |     | Fact_Hospital_Demand     |
       | DimHospital (SCD 2)      |     |  (27,216 rows)           |
       | DimPharmacy (SCD 2)      |     +-------------+------------+
       | DimLocation (Role-play)  |                   |
       +------------+-------------+                   |
                    |                                 |
                    +----------------+----------------+
                                     |
                                     v
                      +-----------------------------+
                      | Analytical BI Views (<100ms)|
                      | - vw_bi_channel_performance |
                      | - vw_bi_hospital_forecast   |
                      | - vw_bi_fast_slow_moving    |
                      +-----------------------------+
```

---

### 2. Core Deliverables & Repository Structure

| File | Type | Description | Rubric |
| :--- | :--- | :--- | :---: |
| **`Milestone.1.SDI.DWH.Excel.xlsx`** | Excel Workbook | Official multi-sheet planning document (8 tabs: `ReadMe`, `Business.Req.XRay`, `OLTP.XRay`, `Transform.OLTP`, `Dim.Tables`, `Fact.Table`, `Star.Schema`, `Bus.Matrix`). | 2.5% |
| **`phase1_sdi_duckdb.sql`** | SQL Script | Complete DDL, SCD Type 1 & 2 upsert logic, sequences, validation rules, fact loading, and analytical views. | 2.5% |
| **`run_etl.py`** | Python Script | Automated pipeline runner that builds the DuckDB data mart (`dwh.duckdb`). Supports `--rebuild`. | Pipeline |
| **`check_etl.py`** | Python Script | Verification test suite: validates financial reconciliation, foreign key integrity, executes 3 dimensional queries, and logs latency metrics. | 2.0% |
| **`test_queries/`** | Directory | Standalone executables for the 3 dimensional queries (`test_query1.py`, `test_query2.py`, `test_query3.py`). | Query Layer |
| **`screenshots/`** | Directory | Terminal execution screenshots proving live query execution in DuckDB. | Evidence |
| **`.gitignore`** | Config | Excludes the 218 MB raw sales CSV to comply with GitHub's 100 MB file limit. | Protection |

---

### 3. Key Pipeline Achievements & Integrity Checks

* **Data Reconciliation:**
  * Raw Sales: **500,000** $\rightarrow$ Loaded in `Fact_Sales`: **500,000** (0 rejects).
  * Raw Demand: **27,216** $\rightarrow$ Loaded in `Fact_Hospital_Demand`: **27,216** (0 rejects).
* **Financial Reconciliation:**
  * Raw CSV Gross Sales: **\$6,038,281.59**
  * Fact Table Gross Sales: **\$6,038,281.59**
  * Variance: **\$0.00**
* **Foreign Key Integrity:** **0 orphan keys** across all dimension lookups.
* **SCD Consistency:** **0 duplicate active records** (`IsCurrent = TRUE`).
* **Channel Modeling Resolution:** Transaction-level channels (`Retail` [305,048], `Hospital` [109,192], `E-commerce` [85,760]) correctly modeled in `Fact_Sales`.

---

### 4. Analytical Query Layer & Observability (Phase 1 BI Dashboards)

All 3 core queries execute with sub-second latency in DuckDB:

| Query ID | Dashboard Name | Analytical Purpose | Execution Latency | Status |
| :--- | :--- | :--- | :---: | :---: |
| **`Q1_SDI`** | Channel Performance & MoM Growth | Evaluates revenue split, gross margin %, discount leakage, and MoM growth across Retail, Hospital, and E-commerce. | **~92 ms** | Passed |
| **`Q2_SDI`** | Hospital Forecast vs Consumed | Evaluates forecast accuracy (MAPE %, bias) and ranks hospitals by unfulfilled demand packs. | **~53 ms** | Passed |
| **`Q3_SDI`** | Fast vs Slow Moving Cold Chain | Multi-fact drill-across combining pharmacy sales and hospital consumption to rank fast/slow movers. | **~16 ms** | Passed |

#### Live Execution Screenshots:

* **Query 1 (`test_queries/test_query1.py`) — Channel Revenue Split & MoM Growth:**
  ![Query 1](screenshots/query1_channel_performance.jpg)

* **Query 2 (`test_queries/test_query2.py`) — Hospital Forecast vs. Consumption & Shortages:**
  ![Query 2](screenshots/query2_hospital_shortages.jpg)

* **Query 3 (`test_queries/test_query3.py`) — Multi-Fact Drill-Across Fast vs. Slow Movers:**
  ![Query 3](screenshots/query3_cold_chain_fast_movers.png)

---

### 5. How to Run the Pipeline

1. **Install Prerequisites:**
   ```bash
   pip install duckdb openpyxl
   ```

2. **Execute the ETL Pipeline:**
   ```bash
   python run_etl.py --rebuild
   ```

3. **Run the Verification & Query Test Suite:**
   ```bash
   python check_etl.py
   ```
