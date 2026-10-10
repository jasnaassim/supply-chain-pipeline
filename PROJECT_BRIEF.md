# Project: Global Order & Fulfillment Data Pipeline

## The scenario

You are the data engineer for **ShopNorth**, an online retailer that ships from 6 warehouses
in the US, Europe and Asia. Every day the order system drops a CSV file and the logistics
partner drops a JSON-lines file of shipments. The files are messy. Leadership wants a
trustworthy daily report on revenue (in USD), delivery performance, and data quality.

Your job: build a Python pipeline that ingests the raw files, validates and cleans them,
loads them into an analytical database, and produces a KPI report — re-runnable every day
without double-counting anything.

## The data (in `data/`)

| File | Format | What it is |
|---|---|---|
| `raw/orders/orders_YYYYMMDD.csv` | CSV, 14 daily files (Sep 1–14, 2026) | One row per order event |
| `raw/shipments/shipments_YYYYMMDD.jsonl` | JSON lines, nested | One shipment per order |
| `reference/products.csv` | CSV | 24 products: category, cost, list price, supplier |
| `reference/warehouses.csv` | CSV | 6 warehouses with local currency |
| `reference/fx_rates.csv` | CSV | Daily currency → USD rates |

**Orders columns:** `order_id, order_ts, customer_id, product_id, warehouse_id, quantity,
unit_price, currency, ship_country, status`
`unit_price` is in the order's local `currency`.

**Shipments (nested JSON):**
```json
{"shipment_id": "SH500004", "order_id": "O500004", "carrier": "DHL",
 "shipped_at": "2026-09-02T15:00:00Z",
 "delivery": {"delivered_at": "2026-09-06T19:30:00Z", "promised_date": "2026-09-06"},
 "package": {"weight_kg": 5.34, "items": 3}}
```

### The data is dirty on purpose

Finding the problems is part of the project. Hints of what to look for:

- Exact duplicate rows
- Missing values in required fields
- Values that are impossible (think about quantity, and about dates)
- More than one date format
- Numbers stored as text with extra characters
- The same country written several different ways
- Products that don't exist in the product catalog
- **Late-arriving updates:** an order placed on one day can show up again in a *later* file
  with a new status (`CANCELLED`, `RETURNED`). The latest status must win.
- Shipments with missing or illogical delivery timestamps
- FX rates are missing on weekends

Don't look at `answer_key/` until you've written your own data-quality report.

---

## Milestones

Commit to GitHub after each one. Each milestone is something you can talk about.

### 1. Project setup
- Create a repo with this layout (rename as you like):
  ```
  supply-chain-pipeline/
  ├── data/                 # raw input (given)
  ├── src/pipeline/         # your package
  │   ├── extract.py
  │   ├── validate.py
  │   ├── transform.py
  │   ├── load.py
  │   └── report.py
  ├── tests/
  ├── config.yaml
  ├── requirements.txt
  └── README.md
  ```
- Use a virtual environment. Suggested libraries: `pandas`, `duckdb` (or `sqlite3`, built in),
  `pyyaml`, `pytest`, `matplotlib`.
- Put file paths and settings in `config.yaml`, not hard-coded in the code.

### 2. Explore (notebook is fine here)
- Load one orders file and one shipments file. Profile them: row counts, nulls per column,
  distinct values, min/max, date formats.
- Write down every problem you find. This becomes your validation rule list.

### 3. Extract
- Read all orders CSVs and all shipments JSONL files from their folders.
- Add lineage columns: `source_file` and `loaded_at`.
- Flatten the nested shipment JSON into columns (`delivery.delivered_at` → `delivered_at`).

### 4. Validate (data quality)
- Write each rule as a small function that returns the bad rows plus a reason.
- Split the data into **clean** rows and **quarantine** rows (rejected rows are kept with a
  `reject_reason`, not silently dropped).
- Produce a data-quality summary: rule name, rows checked, rows failed, % failed.

### 5. Transform
- Standardize: parse both date formats, strip symbols from prices, normalize country codes
  (`US`, `us`, ` US`, `United States` → `US`).
- De-duplicate exact copies.
- Resolve late updates: keep the **latest** status per `order_id`.
- Convert `unit_price` to USD using the FX rate for the order date; for weekends, use the most
  recent previous business-day rate.
- Derive: `revenue_usd`, `margin_usd` (using `unit_cost` from products), and for shipments
  `days_to_deliver` and `on_time` (delivered on or before `promised_date`).

### 6. Load (dimensional model)
Build a small star schema in DuckDB/SQLite:
- `dim_product`, `dim_warehouse`, `dim_date`, `dim_customer`
- `fact_orders` (one row per order, latest status)
- `fact_shipments`
- `dq_quarantine` and `dq_run_log` (one row per run: files, rows read/loaded/rejected, duration)

### 7. Make it incremental and idempotent
This is the part that separates a script from a pipeline.
- Track which files have already been processed. A re-run must not double-load.
- Use upsert/merge logic so a late status update **changes** the existing order row.
- Test it: run the pipeline twice → the row counts must be identical.

### 8. Report
Generate a report (Markdown/HTML plus a few PNG charts) answering:
1. Daily revenue and margin in USD, and the top 5 products by revenue
2. Revenue by warehouse and by category
3. Cancellation and return rate
4. On-time delivery % by carrier, and average days to deliver
5. Data-quality summary: what % of rows was rejected, and why

### 9. Engineering quality
- `pytest` unit tests for at least: date parsing, price cleaning, country normalization,
  latest-status dedup, FX weekend fallback.
- Logging with the `logging` module (not `print`).
- A command-line entry point: `python -m pipeline run --date 2026-09-05` and `--all`.
- A GitHub Actions workflow that runs the tests on every push.

### 10. README (your LinkedIn landing page)
- One-paragraph problem statement
- An architecture diagram (raw → validate → clean/quarantine → star schema → report)
- How to run it in 3 commands
- Screenshots of 2–3 charts and the data-quality table
- "Design decisions" section: why quarantine instead of drop, how idempotency works, how
  late updates are handled

---

## Stretch goals (pick any)

- Swap pandas for **PySpark** in the transform step and compare
- Orchestrate the steps with **Airflow** or **Prefect**
- Load to **Snowflake** (free trial) instead of DuckDB
- Add data-quality checks with **Great Expectations** or **pandera**
- Containerize with **Docker**
- Build a small **Streamlit** dashboard on top of the star schema

## Self-check: you're done when

- [ ] One command runs the full pipeline from raw files to report
- [ ] Running it twice gives identical row counts
- [ ] Every rejected row is in quarantine with a reason
- [ ] Your rejected-row counts are close to `answer_key/injected_issues.json`
- [ ] Tests pass in GitHub Actions
- [ ] A stranger can understand the project from the README in 2 minutes
