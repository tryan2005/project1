The resulting dataset can be connected directly to Power BI, DuckDB, or any Parquet-compatible data tool.

---

## Features
- **Fast ingestion** of thousands of SEC `company_facts` JSON files  
- **Normalization** of US-GAAP USD-reported facts into tabular format  
- **Validation** of quarterly (`10-Q`) and annual (`10-K`) periods  
- **Transformation** into clean Parquet layers:
  - `_raw_us_gaap_usd.parquet` – raw flattened data  
  - `_best.parquet` – earliest-filed deduplicated facts  
  - `fact_quarters_true.parquet` – Q1–Q4 normalized data  
  - `fact_ttm_aligned.parquet` – calendar-aligned TTM values  
  - `dim_fiscal_year_end.parquet` – fiscal year-end detection  
  - `dim_security.parquet` – CIK–ticker–company mapping  
  - `vw_latest_ttm.parquet` – latest TTM snapshot per tag  
- **DuckDB integration** for in-memory transformations  
- **Automatic retry-safe cleanup** of temporary and log directories  

---

## Requirements

### Python 3.9+  
Install dependencies via `pip`:

```bash
pip install pandas pyarrow duckdb orjson
