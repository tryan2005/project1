# SMILE Fund — SEC Data Warehouse

A local-only SEC data warehouse for the SMILE Fund that:

- Downloads / reads SEC `company_facts`, `submissions`, and `company_tickers.json`
- Normalizes XBRL facts into a **star schema** in DuckDB
- Stores cleaned data as Parquet for fast re-use
- Seeds core dimensions (sectors, companies, listings, metrics, units, calendar, filings)
- Prepares data for **DCF** and **relative valuation** models in Excel

> **Note:** This project is designed to run on a local Windows machine under `C:\smilefund_project` using pre-downloaded SEC data.

---

## Table of Contents

1. [Project Overview](#project-overview)  
2. [Data Sources](#data-sources)  
3. [Folder Structure](#folder-structure)  
4. [Technology Stack](#technology-stack)  
5. [Star Schema](#star-schema)  
6. [Installation & Setup](#installation--setup)  
7. [Configuration](#configuration)  
8. [How to Run](#how-to-run)  
9. [Example Queries](#example-queries)  
10. [Data Quality & Normalization](#data-quality--normalization)  
11. [Planned Extensions](#planned-extensions)  
12. [Conventions](#conventions)

---

## Project Overview

The goal of this project is to build a **deterministic, reproducible SEC data warehouse** for the SMILE Fund. It ingests raw SEC JSON files and organizes them into a clean relational model that:

- Makes it easy to compare companies across time
- Normalizes different XBRL tags for similar concepts (e.g., revenue, net income)
- Supports automated pulls into Excel DCF and relative valuation templates
- Enables sector-level and portfolio-level analytics

This is **not** an API project: it runs entirely on pre-downloaded SEC files stored locally.

---

## Data Sources

```
(https://www.sec.gov/Archives/edgar/daily-index/bulkdata/submissions.zip)
(http://www.sec.gov/Archives/edgar/daily-index/xbrl/companyfacts.zip)
(https://www.sec.gov/files/company_tickers.json)
```

## Folder Structure

```
smilefund_project/
│
├─ data/
│   └─ sec/
│       ├─ company_facts/
│       ├─ submissions/
│       └─ company_tickers.json
│
├─ warehouse/
│   ├─ parquet/
│   │   └─ schema_filled/
│   └─ sec_warehouse.duckdb
│
└─ src/
    └─ sec_warehouse_star.py
```

## Technololgy Stack

- `Python 3.x`
- `DuckDB`
- `Parquet (PyArrow)`
- `Pandas`
- Standard Library: `json`, `zipfile`, `pathlib`, `datetime`, `hashlib`

## Star Schema

```
CREATE TABLE `dim_calendar` (
  `calendar_id` INT,
  `date` DATE,
  `fiscal_year` INT,
  `fiscal_quarter` VARCHAR(2),
  `fiscal_month` INT,
  `period_type` VARCHAR(10),
  PRIMARY KEY (`calendar_id`)
);

CREATE TABLE `dim_sector` (
  `sector_id` INT,
  `sector_name` VARCHAR(50),
  PRIMARY KEY (`sector_id`)
);

CREATE TABLE `dim_subsector` (
  `subsector_id` INT,
  `subsector_name` VARCHAR(255),
  `subsector_code`  VARCHAR(32),
  `sector_id` INT,
  `description` VARCHAR(255),
  `is_active` BOOLEAN,
  PRIMARY KEY (`subsector_id`),
  FOREIGN KEY (`sector_id`)
      REFERENCES `dim_sector`(`sector_id`)
);

CREATE TABLE `dim_industry` (
  `industry_id` INT,
  `sic_code` INT,
  `industry_title` VARCHAR(225),
  `subsector_id` INT,
  `sector_id` INT,
  `notes` VARCHAR(255),
  `is_active` BOOLEAN,
  PRIMARY KEY (`industry_id`),
  FOREIGN KEY (`subsector_id`)
      REFERENCES `dim_subsector`(`subsector_id`),
  FOREIGN KEY (`sector_id`)
      REFERENCES `dim_sector`(`sector_id`)
);

CREATE TABLE `dim_company` (
  `company_id` INT,
  `cik` BIGINT,
  `name` VARCHAR(100),
  `ticker` VARCHAR(10),
  `sector_id` INT,
  `subsector_id` INT,
  `industry_id` INT,
  `industry_name` VARCHAR(100),
  `sic_code` INT,
  `fiscal_year_end_month` INT,
  `fiscal_year_end_day` INT,
  PRIMARY KEY (`company_id`),
  FOREIGN KEY (`sector_id`)
      REFERENCES `dim_sector`(`sector_id`),
  FOREIGN KEY (`subsector_id`)
      REFERENCES `dim_subsector`(`subsector_id`),
  FOREIGN KEY (`industry_id`)
      REFERENCES `dim_industry`(`industry_id`),
  KEY `UK` (`cik`)
);

CREATE TABLE `dim_listing` (
  `listing_id` INT,
  `company_id` INT,
  `ticker` VARCHAR(10),
  `exchange` VARCHAR(10),
  `start_date` DATE,
  `end_date` DATE,
  `is_primary` BOOLEAN,
  PRIMARY KEY (`listing_id`),
  FOREIGN KEY (`company_id`)
      REFERENCES `dim_company`(`company_id`)
);

CREATE TABLE `dim_security` (
  `security_id` SERIAL,
  `company_id` INT,
  `ticker` VARCHAR(10),
  `cusip` VARCHAR(12),
  `isin` VARCHAR(12),
  `exchange` VARCHAR(20),
  `share_class` VARCHAR(20),
  `currency` VARCHAR(3),
  `status` VARCHAR(20),
  `listing_date` DATE,
  `delisting_date` DATE,
  PRIMARY KEY (`security_id`)
);

CREATE TABLE `dim_metric` (
  `metric_id` INT,
  `metric_name` VARCHAR(100),
  `xrbl_tag` VARCHAR(100),
  `normal_balance` VARCHAR(10),
  PRIMARY KEY (`metric_id`)
);

CREATE TABLE `dim_filing` (
  `filing_id` SERIAL,
  `form_type` VARCHAR(10),
  `filing_date` DATE,
  `accepted_date` DATE,
  `period_end_date` DATE,
  `accession_number` VARCHAR(30),
  `filing_url` VARCHAR(255),
  PRIMARY KEY (`filing_id`)
);

CREATE TABLE `dim_unit` (
  `unit_id` SERIAL,
  `unit_code` VARCHAR(20),
  `category` VARCHAR(20),
  `iso_currency` VARCHAR(3),
  `decimals_hint` INT,
  `description` VARCHAR(100),
  PRIMARY KEY (`unit_id`)
);

CREATE TABLE `fact_financials` (
  `financial_id` SERIAL,
  `company_id` INT,
  `metric_id` INT,
  `calendar_id` INT,
  `value` NUMERIC(18,2),
  `unit_id` INT,
  `filing_id` INT,
  `consolidated_flag` VARCHAR(20),
  PRIMARY KEY (`financial_id`),
  FOREIGN KEY (`filing_id`)
      REFERENCES `dim_filing`(`filing_id`),
  FOREIGN KEY (`calendar_id`)
      REFERENCES `dim_calendar`(`calendar_id`),
  FOREIGN KEY (`metric_id`)
      REFERENCES `dim_metric`(`metric_id`),
  FOREIGN KEY (`company_id`)
      REFERENCES `dim_company`(`company_id`),
  FOREIGN KEY (`unit_id`)
      REFERENCES `dim_unit`(`unit_id`)
);

CREATE TABLE `dim_company_profile` (
  `company_id` INT,
  `country` VARCHAR(50),
  `state_incorporation` VARCHAR(50),
  `hq_city` VARCHAR(100),
  `hq_state` VARCHAR(100),
  `website` VARCHAR(225),
  FOREIGN KEY (`company_id`)
      REFERENCES `dim_company`(`company_id`)
);

CREATE TABLE `sic_sector_ranges` (
  `range_id` INT,
  `sic_min` INT,
  `sic_max` INT,
  `sector_id` INT,
  `notes` TEXT,
  PRIMARY KEY (`range_id`),
  FOREIGN KEY (`sector_id`)
      REFERENCES `dim_sector`(`sector_id`)
);

CREATE TABLE `dim_company_ids` (
  `ids_id` INT,
  `company_id` INT,
  `legal_name` VARCHAR(225),
  `lei` VARCHAR(20),
  PRIMARY KEY (`ids_id`),
  FOREIGN KEY (`company_id`)
      REFERENCES `dim_company`(`company_id`)
);

CREATE TABLE `fact_holdings` (
  `holding_id` SERIAL,
  `company_id` INT,
  `asof_date` DATE,
  `weight` NUMERIC(8,5),
  `shares` NUMERIC(18,2),
  `market_value` NUMERIC(18,2),
  `source` VARCHAR(50),
  PRIMARY KEY (`holding_id`),
  FOREIGN KEY (`company_id`)
      REFERENCES `dim_company`(`company_id`)
);
```

## Installation & Setup

## Configuration

```
BASE = C:\smilefund_project
DATA = BASE\data\sec
FACTS_DIR = data\company_facts
COMPANY_FACTS_ZIP = data\company_facts.zip
SUBMISSIONS_DIR = data\submissions
SUBMISSIONS_ZIP = data\submissions.zip

WAREHOUSE = base\warehouse
PARQUET_OUT = warehouse\parquet\schema_filled
DUCKDB_PATH = warehouse\sec_warehouse.duckdb
```

## How to Run 

## Example Queries

**Latest Revenue for AAPL**

```
SELECT
    c.legal_name,
    ids.id_value AS ticker,
    cal.year,
    f.value AS revenue
FROM fact_financials f
JOIN dim_metric m ON f.metric_id = m.metric_id
JOIN dim_company c ON f.company_id = c.company_id
JOIN dim_company_ids ids ON f.company_id = ids.company_id
JOIN dim_calendar cal ON f.calendar_id = cal.calendar_id
WHERE m.normalized_name = 'Revenue'
  AND ids.id_type = 'ticker'
  AND ids.id_value = 'AAPL'
ORDER BY cal.year DESC
LIMIT 5;
```

**Net Income by Sector**

```
SELECT
    c.legal_name,
    ids.id_value AS ticker,
    cal.year,
    f.value AS revenue
FROM fact_financials f
JOIN dim_metric m ON f.metric_id = m.metric_id
JOIN dim_company c ON f.company_id = c.company_id
JOIN dim_company_ids ids ON f.company_id = ids.company_id
JOIN dim_calendar cal ON f.calendar_id = cal.calendar_id
WHERE m.normalized_name = 'Revenue'
  AND ids.id_type = 'ticker'
  AND ids.id_value = 'AAPL'
ORDER BY cal.year DESC
LIMIT 5;
```

## Data Quality & Normalization

- Metric normalization
- Unit normalization
- Calendar normalization
- Deterministic ID generation

## Planned Extensions

- FDIC bank data integration
- Ratio metrics (ROE, ROA, Margins)
- Excel DCF Integration
- FI Risk Dashboard

## Conventions

