# POC - Card Transactions

A data engineering system built on **Microsoft Fabric** and **Azure** — ingesting batch and real-time transaction data, processing it through a Medallion Architecture, detecting fraud with a 5-rule composable engine, and serving a branch-secured Power BI dashboard.

 **Full documentation, architecture walkthrough →** [poc.tusharkolte.com](https://poc.tusharkolte.com)

---

## Architecture

<img width="1672" height="941" alt="poc architecture" src="https://github.com/user-attachments/assets/4d5dd43b-4863-4ad0-88b1-0bcf2fa38f9b" />

---

## What's Built

| Layer | Technology | Output |
|---|---|---|
| **Source** | Azure SQL · ADLS Gen2 · Azure Event Hubs | 500 accounts · 6K+ transactions · live stream |
| **Ingestion** | Fabric Eventstream · PySpark · Azure Functions | Bronze Delta tables (append-only) |
| **Processing** | PySpark · Delta MERGE · Delta Lake | Silver — clean, PII-masked, enriched |
| **Serving** | SQL Analytics Endpoint · Star Schema · SCD Type 2 | Gold — dim/fact tables |
| **Detection** | PySpark Window Functions · 5 fraud rules | `fact_suspicious_transactions` |
| **Dashboard** | Power BI Desktop · DirectQuery · Branch RLS | Management + Analyst pages |
| **Automation** | Fabric Data Pipeline · Azure Functions · Watermarks | Nightly batch · live every 5 min |

---

## Key Highlights

- **Medallion Architecture** — Bronze (raw audit) → Silver (clean, masked) → Gold (analytics-ready)
- **SCD Type 2** on `dim_account` — preserves historical risk context for accurate fraud attribution
- **Watermark incremental loading** — central `_pipeline_watermarks` Delta table, epoch-zero bootstrap
- **5 composable fraud rules** — HIGH_VALUE · VELOCITY (Window Function) · GEO_MISMATCH · NEW_ACCOUNT · KYC_NON_COMPLIANT
- **PII dropped at Silver** — `pan`, `email`, `phone`, `date_of_birth` structurally cannot reach Gold
- **Autonomous live stream** — Azure Functions timer trigger replaces laptop-dependent producer
- **Branch-level RLS** — 8 static roles on `dim_account[branch]`, cascades to both fact tables

---

## Tech Stack

![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-742774?style=flat&logo=microsoft&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![Apache Spark](https://img.shields.io/badge/PySpark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-FF3621?style=flat&logo=databricks&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)

