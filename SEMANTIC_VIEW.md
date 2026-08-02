# Semantic View: SV_AP_ANALYTICS

## Overview

Created a semantic view `COCO_WORKSHOP.PIPELINE_LAB.SV_AP_ANALYTICS` over the `SILVER_AP_INVOICES` Dynamic Table to enable natural-language querying via Cortex Analyst.

**Deployed via:** `SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML`

## What It Contains

| Component | Count | Examples |
|-----------|-------|---------|
| Dimensions | 14 | source_system, vendor_name, cost_center, currency_code, approval_status |
| Time Dimensions | 1 | invoice_date |
| Facts | 1 | invoice_amount |
| Metrics | 6 | total_spend, invoice_count, average_invoice_amount, vendor_count, overdue_invoice_count, overdue_amount |
| Verified Queries | 5 | Seed questions for grounding |

## Dimensions

| Name | Description | Synonyms |
|------|-------------|----------|
| source_system | ERP system of origin: SAP, ORACLE, BAAN, or WORKDAY | source, system, ERP |
| source_invoice_id | Unique invoice ID from originating system | — |
| invoice_number | Business-facing invoice reference | invoice ref, invoice #, inv number |
| vendor_id | Vendor/supplier code (varies by source) | — |
| vendor_name | Human-readable vendor name | vendor, supplier, supplier name |
| due_date | Date payment is due | payment due date, due by, pay by date |
| currency_code | ISO 4217: USD, EUR, or GBP | currency, ccy |
| payment_terms | NET30 or NET60 | terms, pay terms |
| po_number | Purchase order (may be NULL) | PO, purchase order |
| line_description | Free-text invoice description | description, memo |
| gl_account | General ledger code from source | GL code, account, ledger account |
| cost_center | Business unit charged | cost center, department, business unit, dept |
| approval_status | APPROVED or PENDING | status, approval state |
| created_at | UTC timestamp of record creation | — |

## Time Dimensions

| Name | Description | Synonyms |
|------|-------------|----------|
| invoice_date | Date issued. Primary dimension for trend analysis. | invoice date, date issued |

## Facts

| Name | Description | Synonyms |
|------|-------------|----------|
| invoice_amount | Value in original transaction currency. Group by currency_code when summing. | amount, invoice value, spend |

## Metrics

| Name | Expression | Description | Synonyms |
|------|-----------|-------------|----------|
| total_spend | `SUM(invoice_amount)` | Total invoice amounts | total amount, total AP |
| invoice_count | `COUNT(*)` | Number of invoices | volume, count, how many invoices |
| average_invoice_amount | `AVG(invoice_amount)` | Average per invoice | avg amount, average spend |
| vendor_count | `COUNT(DISTINCT vendor_name)` | Distinct vendors | number of vendors, supplier count |
| overdue_invoice_count | `COUNT_IF(due_date < CURRENT_DATE())` | Past-due invoices | overdue count, past due, late invoices |
| overdue_amount | `SUM(CASE WHEN due_date < CURRENT_DATE() THEN invoice_amount ELSE 0 END)` | Total past-due amount | unpaid amount, overdue spend |

## Verified Queries (Seed Questions)

| Name | Question | SQL Summary |
|------|----------|-------------|
| total_spend_by_vendor | What is the total AP spend by vendor? | GROUP BY vendor_name, currency_code |
| invoice_count_by_month | How many invoices per month? | DATE_TRUNC('MONTH', invoice_date) |
| top_vendors_overdue | Top 10 vendors by unpaid invoice amount | WHERE due_date < CURRENT_DATE(), LIMIT 10 |
| invoice_count_by_source | Which source system has the most invoices? | GROUP BY source_system |
| pending_by_vendor | Show me pending invoices by vendor | WHERE approval_status = 'PENDING' |

## Design Assumptions

1. **Grain = one row per invoice** — no line-item splits exist in the Silver layer.
2. **`invoice_date` is the primary time dimension** — used for "last 12 months", "by month" questions. `due_date` is a regular dimension for overdue filtering.
3. **`vendor_name` is the grouping key for vendors** — `vendor_id` formats vary by source and aren't human-readable.
4. **"Overdue" = `due_date < CURRENT_DATE()`** — no paid/settled flag exists in the data.
5. **Multi-currency amounts stay separate** — the model warns Cortex Analyst to always group by `currency_code` when aggregating spend.

## Retrieval Commands

```sql
-- View the semantic view definition
DESCRIBE SEMANTIC VIEW COCO_WORKSHOP.PIPELINE_LAB.SV_AP_ANALYTICS;

-- Get the full DDL
SELECT GET_DDL('SEMANTIC_VIEW', 'COCO_WORKSHOP.PIPELINE_LAB.SV_AP_ANALYTICS');
```
