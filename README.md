# 📊 Seasonality Coefficients Calculation

A data engineering & analytics pipeline built at **Danone** to calculate standardized **seasonality coefficients** for global sell-out sales data, across multiple business scopes (Dairy, Plant-Based, Waters, and Specialized Nutrition), feeding a downstream Power BI reporting layer.

## 📌 Project Overview

Different business categories sell at different paces throughout the year — some peak in summer (e.g., waters), others have steady demand (e.g., dairy staples). To make volume/value forecasts and targets comparable and realistic across categories and countries, this project computes a **seasonality coefficient per period, per business "grain"** (country × category combination), based on rolling 3-month (L3M) sales weighted against full-year historical totals.

The pipeline pulls raw sell-out data from a cloud data warehouse, applies business-scope-specific data preparation rules, calculates coefficients with year-completeness validation, and exports a consolidated, Power BI-ready output table.

> ⚠️ **Note:** This is a generalized, anonymized version of an internal Danone project. All database connection details, credentials, internal table names, and personal file paths have been removed or replaced with generic placeholders. No real sales figures or proprietary business results are included — this repository showcases the **methodology and code only**.

## 🎯 Objectives

- Consolidate global sell-out data across five business scopes (**Dairy, Plant-Based, Waters, Specialized Nutrition - Early Life, Specialized Nutrition - Pediatrics/Allergy**) into a single analytical base.
- Define a consistent **"grain"** (the unit of analysis: country + category combination) per business scope, handling scope-specific edge cases.
- Calculate a **seasonality coefficient** per grain and period, based on a rolling 3-month sales window (L3M) relative to full-year historical sales.
- Validate **year completeness** per grain before including it in historical totals, to avoid coefficients skewed by partial/incomplete years.
- Produce a clean, standardized output table ready to be consumed by **Power BI** dashboards.

## 🔍 Methodology

### 1. Data Extraction
- Connected to the company's cloud data warehouse and queried global sell-out delivery data.
- Built five parallel `SELECT ... UNION ALL` blocks, one per business scope, each filtering the relevant categories/subcategories and country list for that scope.
- Materialized the consolidated result as a table for reuse across the pipeline.

### 2. Exploratory Data Analysis
- Checked shape, data types, and null percentages across all columns.
- Reviewed unique values per categorical column (business scope, country, period, fact type) to validate data completeness before modeling.
- Split the data into per-scope sub-dataframes to inspect row counts, country coverage, and period coverage independently.

### 3. Business-Scope-Specific Preparation
Each business scope required a tailored **grain definition** and edge-case handling:
- **Dairy / Plant-Based:** grain = `Country | Scope | Category`.
- **Waters:** grain = `Country | WATERS | Subcategory`, with a special rule consolidating China's "Other Drinks" and "Functional Drinks" categories into a unified "Super Drinks" subcategory to align with local business classification.
- **Specialized Nutrition (Early Life / Pediatrics):** grain defined at subcategory level, covering infant formula, allergy, and other pediatric nutrition subcategories across country-specific product structures.

### 4. Coefficient Calculation (`calculate_coefficients`)
Reusable function applied to every business scope:
1. **Year completeness check:** for each grain, determines the set of periods present in its most recent year and only keeps prior years that match that exact same period pattern — preventing distortion from incomplete years.
2. **Rolling L3M (Last 3 Months) sum:** computed per grain and per metric (volume/value).
3. **Historical total:** full-year-validated total sales per grain and metric.
4. **Coefficient:** `L3M_total / historical_total` — representing the relative weight of that 3-month window within the grain's typical annual pattern.

### 5. Consolidation & Export
- Concatenated the per-scope coefficient tables into a single master table.
- Validated the master table (row counts, grain counts, null checks) before export.
- Reformatted columns/types to match the expected **Power BI** data model.
- Exported the final table to Excel for ingestion into the reporting layer.

## 🛠️ Tech Stack

- **Language:** Python (PySpark + pandas)
- **Data warehouse:** Snowflake (accessed via Spark connector on a Databricks-based platform)
- **Data manipulation:** pandas, numpy
- **Output / BI integration:** Excel export feeding a **Power BI** dashboard

## 📁 Repository Structure

```
├── Seasonality_Coefficients_Final.ipynb   # Full pipeline: extraction, EDA, per-scope prep, coefficient calc, export
└── README.md
```

## 🚀 How to Run

This notebook was originally built on a Spark-based data platform (Databricks) connected to Snowflake. To adapt it to your own environment:

1. Install dependencies:
   ```bash
   pip install pyspark pandas numpy openpyxl
   ```

2. Update the connection block with your own warehouse credentials (ideally via a secrets manager, not hardcoded):
   ```python
   sfOptions_prod = {
     "sfURL": "<your_company>.snowflakecomputing.com",
     "sfUser": dbutils.secrets.get(scope="snowflake-secrets", key="snowflake-user"),
     "sfPassword": dbutils.secrets.get(scope="snowflake-secrets", key="snowflake-password"),
     "sfDatabase": "<database_name>",
     "sfSchema": "<schema_name>",
     "sfWarehouse": "<warehouse_name>",
   }
   ```

3. Point `dbtable` to your own sell-out/sales fact table with a similar schema (category, subcategory, period, country, metric type, value).

4. Run the notebook top to bottom. The core reusable function (`calculate_coefficients`) can be applied to any business scope by first defining a `GRAIN_ID` column that fits your own product/category hierarchy.

## 📈 Key Insights (Methodology-level)

- Standardizing the "grain" concept per business scope (rather than a one-size-fits-all category key) made it possible to handle real-world data quirks — like China's water subcategory consolidation — without breaking the shared coefficient logic.
- The **year-completeness validation** step is what makes the coefficients reliable: without it, grains with partial data (e.g., a new market only reporting since P8) would produce misleadingly high or low seasonality weights.
- Building the coefficient function once and reusing it across five different business scopes kept the pipeline maintainable and easy to extend to new categories.

## 📄 License

This repository shares an anonymized methodology developed during a professional engagement. No proprietary data, credentials, or confidential business results are included.
