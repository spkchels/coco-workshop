# SILVER_AP_INVOICES — PRD-Driven Update (2026-08-02)

## Artifacts Checklist

| # | Artifact | Location | Purpose |
|---|----------|----------|---------|
| 1 | **Source onboarding requests** | `sample_business_requirements_source_onboarding.csv` | Who requested Baan/Workday, priority, go-live targets, contacts |
| 2 | **Column mapping spec** | `sample_business_requirements_column_mapping.csv` | Field-by-field mapping from each source to the Silver schema |
| 3 | **Business rules** | `sample_business_requirements_business_rules.csv` | Status normalization, dedup logic, scope boundaries (Silver vs Gold) |
| 4 | **PRD Evaluator skill** | `.cortex/skills/prd-to-pipeline-plan/SKILL.md` | Reusable skill for analyzing future PRDs against any target Dynamic Table |
| 5 | **Dynamic Table DDL** | Live in Snowflake (see below) | The deployed `CREATE OR ALTER` definition |
| 6 | **This change log** | `CHANGE_LOG.md` | Review context, assumptions, and validation steps |

## Retrieving the Current DDL

```sql
SELECT GET_DDL('DYNAMIC_TABLE', 'COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES');
```

## What Changed

- Added `BAAN` and `WORKDAY` UNION ALL branches (20 new rows)
- Applied status normalization (BR-001) across all 4 branches
- Added Baan dedup via `QUALIFY ROW_NUMBER()` (BR-003)
- Normalized payment terms at Silver for all sources

## Assumptions Pending Review

1. Payment terms normalized at Silver (PRD says "Open — needs decision")
2. Baan dedup scoped to within-Baan only (not cross-source)
3. Oracle `VALIDATED` → `APPROVED` is a breaking change for downstream consumers

## Validation Queries

```sql
-- 1. Row counts (expect SAP=15, ORACLE=15, BAAN=10, WORKDAY=10)
SELECT SOURCE_SYSTEM, COUNT(*) AS ROW_COUNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM
ORDER BY SOURCE_SYSTEM;

-- 2. Status values (expect only APPROVED, PENDING)
SELECT DISTINCT APPROVAL_STATUS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
ORDER BY APPROVAL_STATUS;

-- 3. Payment terms consistency (expect NET30, NET60 for all sources)
SELECT SOURCE_SYSTEM,
       LISTAGG(DISTINCT PAYMENT_TERMS, ', ') WITHIN GROUP (ORDER BY PAYMENT_TERMS) AS TERMS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM
ORDER BY SOURCE_SYSTEM;

-- 4. Baan dedup check (expect zero rows)
SELECT INVOICE_NUMBER, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER
HAVING CNT > 1;

-- 5. No NULL keys (expect all zeros)
SELECT COUNT_IF(SOURCE_INVOICE_ID IS NULL) AS NULL_IDS,
       COUNT_IF(INVOICE_DATE IS NULL) AS NULL_DATES,
       COUNT_IF(VENDOR_ID IS NULL) AS NULL_VENDORS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES;
```

## Reusing the PRD Evaluator Skill

For the next source onboarding or business rule change:

```
$prd-to-pipeline-plan
prd_path: <path to new requirements file>
target_dynamic_table: COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
```

The skill returns five sections: new sources, rules affecting the DT, rules out of scope, open questions, and implementation steps.
