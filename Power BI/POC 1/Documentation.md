
## Overview

This dashboard provides an interactive analysis of retail sales performance across multiple dimensions including sales, profitability, customer behavior, and discount impact. The solution is built in Power BI using data sourced from Excel and CSV files and is designed to support executive decision-making through KPI monitoring and self-service analytics.

---


### Data Sources
- Sample Superstore dataset (Excel)
- Product Data (Excel/CSV)



# Dashboard Pages

## 1. Sales Overview

The Sales Overview page provides a high-level summary of business performance.

### KPIs
- Total Sales (Current Year vs Previous Year)
- Total Profit (Current Year vs Previous Year)
- Profit Margin % (Current Year vs Previous Year)
- Total Quantity Sold (Current Year vs Previous Year)

### Visualizations
- Sales by City (Map)
- Sales by Region (Bar Chart)
- Category and Sub-Category performance table with:
  - Total Sales
  - Total Profit
  - Profit Margin %

### Filters
- Year
- Segment


---

## 2. Profit / Sales Analysis

This page focuses on sales and profit trends over time.

### KPIs
- Total Customers
- Average Order Value
- Total Orders
- Profit per Order

### Visualizations
- Total Sales by Year with Forecast
- Total Profit by Year with Forecast
- Top 5 Sub-Categories by Sales
- Top 5 Sub-Categories by Profit
- Monthly Sales vs Profit Comparison

---

## 3. Customer Segmentation

Analyzes customer behavior and retention patterns.

### KPIs
- Total Customers
- Repeat Customers
- Customer Retention Rate
- Customer Frequency

### Visualizations
- Customer Age Distribution
- Top Customers by Sales
- Orders by Ship Mode

---

## 4. Retail Impact Insights

Analyzes the effect of discounts on business performance.

### KPIs
- Discount to Sales Ratio
- Sales Volume Change Due to Discounts
- Profit Volume Change Due to Discounts
- Discount Redemption Rate

### Visualizations
- Key Influencers for Sales
- What-If Analysis for Profit Growth
- Profit Projection based on Sales Increase %

---

# Security Implementation

## Row-Level Security (RLS)

Row-Level Security has been implemented using the **Region** column.

This allows users to view only the data belonging to their assigned region while maintaining a single centralized dataset.

Example:
- West Region User → West data only
- East Region User → East data only

This ensures secure and controlled access to business information.

---

# Scheduled Refresh

The dataset refresh is automated using **OneDrive Integration**.

### Refresh Configuration
1. Source files are stored in OneDrive.
2. Power BI is connected using OneDrive/SharePoint URLs.
3. Power BI Service automatically synchronizes changes from OneDrive.
4. Scheduled refresh ensures reports remain up to date without manual intervention.

### Benefits
- No local machine dependency
- Automatic cloud synchronization
- Reduced maintenance effort
- Improved data reliability

---
