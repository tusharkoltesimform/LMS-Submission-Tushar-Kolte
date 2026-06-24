# Customer Personalization in Financial Services
### Power BI Dashboard — End-to-End Business Intelligence Project


---

## Overview

The dashboard is **live on Power BI Service** and accessible to stakeholders through role-based access.

---

## Dashboard Pages

### Page 1 — Customer Overview
Tracks the health of the customer base across segments, age groups, and demographics.

| KPI | Visual |
|---|---|
| Total Customers · Churn Rate · Engagement Score | Monthly Churn Rate over Time |
| CLV · High Income Customers | Customers by Age Group |
| | Customers by Segment · Gender Breakdown |
<img width="1422" height="805" alt="image" src="https://github.com/user-attachments/assets/01ff908b-7357-4f34-b118-b2125495db62" />


### Page 2 — Transaction Analysis
Breaks down revenue, profit, and volume across time, products, regions, and categories.

| KPI | Visual |
|---|---|
| Revenue · Profit · Quantity Sold | MonthWise Sales / Profit / Quantity |
| Avg Sales per Transaction | Product Wise · Region Wise breakdown |
| Product Personalization Rate | Product Category breakdown |
<img width="1418" height="798" alt="image" src="https://github.com/user-attachments/assets/a6546674-f8c5-40ac-ac1b-5e80319f7bd3" />


### Page 3 — Campaign & Feedback Insights
Evaluates campaign ROI, conversion trends, and the relationship between customer feedback and retention.

| KPI | Visual |
|---|---|
| Total Campaigns · Customer Retention Rate | MonthWise Conversion Rate |
| Customer Frequency · CAC | ROI by Marketing Channel |
| Conversion Rate | Feedback Score vs Retention Rate · Campaign Type Distribution |
<img width="1359" height="799" alt="image" src="https://github.com/user-attachments/assets/49162c2a-e096-4252-ba91-a67d099278f4" />


---



## DAX Measures

19 measures built across a dedicated **Measure Table**, organized by dashboard page.

**Customer metrics** — `Total Customers`, `Churn Rate`, `Churned Customers`, `Customer Engagement Score`, `CLV`, `High Income Customers`

**Transaction metrics** — `Revenue`, `Profit`, `Quantity Sold`, `Avg Sales per Transaction`, `Profit Margin %`, `Product Personalization Rate`

**Campaign metrics** — `Total Campaigns`, `Customer Retention Rate`, `Customer Frequency`, `Customer Acquisition Cost`, `Conversion Rate`, `Total ROI`, `Avg Feedback Score`

All percentage measures (Churn Rate, Retention Rate, Conversion Rate, etc.) use `DIVIDE()` to return safe 0–1 decimals, formatted as Percentage in the Modeling tab.

---

## Row-Level Security — Region Based

RLS was implemented to ensure each regional team only sees data relevant to their area.

**How it works:**
- A security role was created per region (e.g. East, West, North, South) in the Power BI Desktop **Modeling → Manage Roles** pane
- Each role applies a DAX table filter on `data_customers[Region]` using `[Region] = "East"` (and equivalent per role)
- Because `data_customers` is the central dimension table, this single filter propagates through all relationships — restricting transactions, feedback, and campaign data automatically
- After publishing to Power BI Service, workspace members are mapped to their respective roles under **Security settings** on the semantic model

---

## Scheduled Data Refresh — Microsoft Fabric Lakehouse

Data refresh is fully automated using **Microsoft Fabric Lakehouse** as the backend source.

**Setup:**
1. Source data was loaded into a Fabric Lakehouse (OneLake) and exposed as a SQL analytics endpoint
2. Power BI Desktop was connected to the Lakehouse via **Get Data → Microsoft Fabric → Lakehouse**
3. After publishing to Power BI Service, the semantic model's **Scheduled Refresh** was configured:
   - Frequency: **Daily**
   - Time: **2:00 AM** (off-peak)
   - Credentials authenticated via the organizational account with access to the Fabric workspace
4. Fabric handles the upstream data pipeline; Power BI imports the refreshed snapshot on schedule

---
