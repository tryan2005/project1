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

All SEC data is stored under:

```text
C:\smilefund_project\data\sec\

