C:\smilefund_project
│
├── data
│   └── sec
│       ├── company_facts\        # raw SEC JSONs
│       └── ticker_map.csv        # optional override mapping
│
├── warehouse
│   └── parquet
│       └── marts_v2
│           ├── _logs\
│           ├── _duckdb_tmp\
│           ├── _raw_us_gaap_usd.parquet
│           ├── _best.parquet
│           ├── fact_quarters_true.parquet
│           ├── fact_ttm_aligned.parquet
│           ├── dim_fiscal_year_end.parquet
│           ├── dim_security.parquet
│           └── vw_latest_ttm.parquet
