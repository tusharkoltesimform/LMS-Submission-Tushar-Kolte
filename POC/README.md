# POC - Documentation

> An end-to-end data engineering system that ingests transactional data from Azure sources, processes it through a Medallion Architecture on Microsoft Fabric, applies a 5-rule fraud detection engine, and serves results to a Power BI dashboard with branch-level Row-Level Security.
<img width="1672" height="941" alt="poc architecture" src="https://github.com/user-attachments/assets/863debba-16cd-473e-a4bf-894b768c5749" />

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Data Sources](#data-sources)
- [Medallion Layers](#medallion-layers)
- [Fraud Detection Rules](#fraud-detection-rules)
- [Pipelines & Orchestration](#pipelines--orchestration)
- [Security & Governance](#security--governance)
- [Power BI Dashboard](#power-bi-dashboard)
- [Resource Naming](#resource-naming)
- [Known Limitations](#known-limitations)
- [Reset Procedure](#reset-procedure)

---

## Overview

This POC validates a real-time and batch transaction monitoring pipeline capable of detecting financial fraud across 500 accounts and ~6,000+ transactions. It demonstrates incremental ingestion, PII masking, SCD Type 2 dimensional modelling, and live streaming — all within the Microsoft Fabric ecosystem.

---

## Architecture

The system follows a **5-layer architecture**:

```
Data Sources → Ingestion Layer → Fabric Lakehouse (Bronze → Silver → Gold) → Serving Layer → Consumption
```

| Layer | Components |
|---|---|
| **1. Data Sources** | Azure SQL Database, ADLS Gen2 (CSV files), Azure Functions (live events) |
| **2. Ingestion** | Azure Event Hubs → Fabric Eventstream; ADLS shortcut for files |
| **3. Lakehouse** | Microsoft Fabric Medallion Architecture (Bronze / Silver / Gold Delta tables) |
| **4. Serving** | SQL Analytics Endpoint (high-performance read layer over the Lakehouse) |
| **5. Consumption** | Power BI Dashboard with Row-Level Security |

---

## Tech Stack

**Microsoft Fabric** — Lakehouse, Eventstream, Data Pipelines, PySpark Notebooks, SQL Analytics Endpoint

**Azure** — SQL Database (Serverless), ADLS Gen2, Event Hubs (2 partitions), Key Vault, Azure Functions (Python 3.11)

**Power BI Desktop** — DirectQuery over SQL Analytics Endpoint, DAX measures, branch-level RLS

---

## Data Sources

### Azure SQL Database — `txn-accounts-db`
- 500 accounts with 13 attributes: account ID, PII fields (name, email, phone, PAN, DOB), branch, balance, risk category, KYC status, account status.
- `updated_at` trigger fires on every UPDATE — used as the watermark for incremental Bronze ingestion.

### ADLS Gen2 — `txnmonitoringpoc`
- Container: `historical-transactions`
- 6 monthly CSV files: `transactions_2026_01.csv` through `transactions_2026_06.csv`, ~1,021 rows each.
- Accessed via an OneLake shortcut (`historical_transactions_shortcut`) — no data copy needed.

### Azure Functions — `func-txn-producer-poc`
- Timer-triggered every 5 minutes, emitting 10–20 synthetic transactions to Azure Event Hubs.
- Periodically injects fraud patterns: HIGH_VALUE bursts (minute % 25 < 5) and VELOCITY bursts (minute % 50 < 5).
- Event payload: `transaction_id`, `account_id`, `amount`, `merchant`, `city`, `transaction_timestamp`.

---

## Medallion Layers

### 🟤 Bronze — Raw Ingestion

Immutable raw data exactly as received from sources. No transformations, just metadata added.

| Table | Source | Write Mode |
|---|---|---|
| `raw_accounts` | Azure SQL (incremental by `updated_at`) | Append |
| `raw_historical_transactions` | ADLS CSV files (incremental by filename) | Append |
| `raw_live_transactions` | Fabric Eventstream from Event Hubs | Streaming (Eventstream) |

Incremental state is tracked via a `_pipeline_watermarks` Delta table — human-readable, auditable, and manually resettable.

---

### ⚪ Silver — Clean & Conformed

Validated, deduplicated, enriched, and **PII-masked** business data. Raw PII columns are physically **dropped** (not just masked) so leakage to Gold is structurally impossible.

**PII handling:**

| Field | Bronze | Silver |
|---|---|---|
| email | `aarav@example.com` | `a***@example.com` |
| phone | `9876543210` | `XXXXXX3210` |
| PAN | `ABCDE1234F` | `XXXXX1234F` |
| Date of Birth | `1987-03-14` | Age bucket: `36-50` |

**Key tables:**

- `silver_accounts` — merged (UPSERT) on `account_id`, adds masked fields and age bucket.
- `silver_transactions` — deduped on `transaction_id`, enriched with account attributes (risk, KYC, home city), derives `is_city_mismatch` flag.
- `quarantine_transactions` — rows failing validation (NULL IDs, negative amounts) land here as an audit log.

---

### 🟡 Gold — Business & Analytics

Star schema optimised for Power BI. Contains aggregated, analytics-ready facts and dimensions.

| Table | Type | Notes |
|---|---|---|
| `dim_account` | SCD Type 2 | Tracks historical risk/KYC changes via MD5 hash comparison |
| `dim_date` | Conformed | Full year 2026, 365 rows, generated programmatically |
| `fact_transactions` | Incremental fact | All transactions linked to current `account_key` and `date_key` |
| `fact_suspicious_transactions` | Incremental fact | Only flagged transactions; includes boolean flags per rule + pipe-separated `rules_triggered` string |

**Why SCD Type 2?** If account A105 was `LOW` risk when a ₹75,000 transaction occurred, that historical context must survive any later upgrade to `HIGH` risk. SCD Type 1 would corrupt the audit trail.

---

## Fraud Detection Rules

All 5 rules run in `nb_gold_facts_incremental`. Only **ACTIVE** accounts are evaluated. Each rule sets a boolean flag; results are composed into a `rules_triggered` pipe-string and a `rule_count` integer for prioritisation.

| # | Rule | Trigger Condition |
|---|---|---|
| 1 | **HIGH_VALUE** | `amount > ₹5,000` |
| 2 | **VELOCITY** | More than 3 transactions from the same account within a 1-hour rolling window |
| 3 | **GEO_MISMATCH** | Transaction city ≠ account home city AND `amount > ₹1,000` |
| 4 | **NEW_ACCOUNT_LARGE_TXN** | Account age < 548 days AND `amount > ₹2,000` |
| 5 | **KYC_NON_COMPLIANT** | KYC status is `EXPIRED` or `PENDING` AND `amount > ₹500` |

Higher `rule_count` = higher investigation priority. The `rules_triggered` string (e.g. `HIGH_VALUE|GEO_MISMATCH`) is used for human-readable display in Power BI; boolean flags drive reliable DAX filtering.

---

## Pipelines & Orchestration

### `pl_master_batch` — Nightly at 00:30 UTC

Orchestrates the full Bronze → Silver → Gold refresh with dependency-aware execution:

Every activity has an `ON FAILURE` path: a Web Activity posts to a Power Automate webhook, then a Fail Activity marks the run as failed.

### `pl_bronze_historical_arrival` — Daily at 00:00 UTC

Runs `nb_bronze_historical_incremental` independently to process any newly arrived CSV files in ADLS.

### Eventstream — `es_live_transactions`

Continuously streams Event Hubs events into `raw_live_transactions`. Pause before any pipeline reset; resume to restart live flow. Writes in micro-batches: minimum 100 rows or 60-second timeout.

---

## Security & Governance

### PII Protection
Raw PII columns (`email`, `phone`, `PAN`, `date_of_birth`) are **dropped** at the Silver layer transformation — not just masked. They do not exist in Silver or Gold tables.

### Key Vault
All secrets (SQL password, Event Hub connection string) are stored in Azure Key Vault and retrieved in notebooks via:
```python
notebookutils.credentials.getSecret("kv-txn-poc", "sql-admin-password")
```

### Row-Level Security (Power BI)
8 branch-based roles (`AHM-001`, `MUM-001`, etc.) restrict dashboard data to the user's assigned branch. Roles are assigned via the Fabric semantic model Security settings.

### Incremental Watermarks
The `_pipeline_watermarks` Delta table tracks the last processed timestamp or file list per layer. It uses `WriteSerializable` isolation to allow safe parallel updates from concurrent pipeline activities.

---

## Power BI Dashboard

**Connection:** DirectQuery over the SQL Analytics Endpoint of `lh_txn_monitoring`.

**Data model:** Star schema — `fact_transactions` and `fact_suspicious_transactions` connected to `dim_account` (filtered to `is_current = TRUE`) and `dim_date`.


---

## Resource Naming

### Azure Resources

| Resource | Name |
|---|---|
| Resource Group | `rg-txn-poc` |
| SQL Server | `txnaccounts` (Serverless) |
| SQL Database | `txn-accounts-db` |
| Storage Account | `txnmonitoringpoc` |
| ADLS Container | `historical-transactions` |
| Event Hubs Namespace | `evhns-txn-poc` |
| Event Hub | `txn-events` (2 partitions) |
| Consumer Group | `cg-fabric-eventstream` |
| Key Vault Scope | `kv-txn-poc` |
| Azure Function App | `func-txn-producer-poc` |

### Fabric Items

| Item | Name |
|---|---|
| Lakehouse | `lh_txn_monitoring` |
| Eventstream | `es_live_transactions` |
| Master Pipeline | `pl_master_batch` |
| File Pipeline | `pl_bronze_historical_arrival` |
| Notebooks | `nb_bronze_accounts_incremental`, `nb_bronze_historical_incremental`, `nb_silver_accounts_incremental`, `nb_silver_transactions_incremental`, `nb_gold_dims_scd2`, `nb_gold_facts_incremental` |
| Watermark Init | `nb_init_watermarks` |

---
