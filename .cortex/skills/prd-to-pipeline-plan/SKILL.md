---
name: prd-to-pipeline-plan
description: "Analyze PRD-style requirements files and produce a structured implementation plan for updating a target Dynamic Table. Use when: onboarding new sources, adding business rules to a pipeline, reviewing PRD/requirements docs for pipeline impact. Triggers: PRD, requirements, source onboarding, business rules, pipeline plan, new source, update pipeline from requirements."
user_invocable: true
---

# PRD to Pipeline Plan

## When to Use

Invoke this skill when you have a requirements document (XLSX, CSV, or markdown) that describes changes to a data pipeline and you need a structured analysis before implementation. Typical scenarios:

- New source systems being onboarded to an existing Silver/Gold layer
- Business rules that change how data is transformed
- Column mapping changes or status normalization updates
- Any PRD that implies changes to a Dynamic Table definition

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prd_path` | Yes | Path to the requirements file (XLSX, CSV, or markdown). For XLSX, specify sheet name if multiple sheets exist. |
| `target_dynamic_table` | Yes | Fully-qualified DT name (e.g., `DB.SCHEMA.TABLE`) that the requirements affect. |

## Workflow

### Step 1: Load and Parse Requirements

1. Read the file at `prd_path`.
   - **XLSX**: Read all sheets. Identify which sheets contain source onboarding, business rules, column mappings, or other pipeline-relevant content.
   - **CSV**: Read the file and infer structure from headers.
   - **Markdown**: Parse sections and tables.
2. If the file is ambiguous or has multiple unrelated sections, ask the user which sections are relevant.

### Step 2: Inspect the Target Dynamic Table

Run read-only queries to understand the current state:

```sql
-- Current DT definition
SHOW DYNAMIC TABLES LIKE '<table_name>' IN SCHEMA <db>.<schema>;

-- Current columns
SELECT COLUMN_NAME, DATA_TYPE
FROM <db>.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = '<schema>' AND TABLE_NAME = '<table_name>'
ORDER BY ORDINAL_POSITION;

-- Current row count by key dimensions (if SOURCE_SYSTEM exists)
SELECT SOURCE_SYSTEM, COUNT(*) AS ROW_COUNT
FROM <db>.<schema>.<table_name>
GROUP BY SOURCE_SYSTEM;
```

### Step 3: Identify Upstream Sources

If new sources are referenced in the PRD:

```sql
-- Check if bronze tables already exist
SHOW TABLES IN SCHEMA <source_schema>;
```

Compare existing source table columns against the target DT's column contract.

### Step 4: Produce the Analysis

Return **exactly** these five sections:

---

#### Section A: New Source Systems

For each new source, report:
- Source name and platform
- Bronze table name (if it exists)
- Column mapping to the target DT's existing columns
- Columns that have no clear mapping (flag as open questions)

#### Section B: Business Rules Affecting the Target DT

For each rule that changes the DT definition:
- Rule ID and description
- Whether it adds logic (CASE, QUALIFY, JOIN) to the DT query
- Which UNION ALL branch(es) it affects
- Whether it changes output for existing rows (breaking change)

#### Section C: Business Rules NOT Affecting the Target DT

Explicitly list rules that are out of scope for this DT (e.g., Gold-layer concerns, DMFs, alerting). This prevents silent scope creep.

#### Section D: Ambiguities and Open Questions

For every assumption that cannot be resolved from the document alone:
- State what the document says (or doesn't say)
- State why it's ambiguous
- Suggest who might own the decision
- **Never guess. Never default silently.** If the PRD is unclear, surface it here.

#### Section E: Recommended Implementation Steps

A numbered list of concrete changes to the DT, ordered by dependency:
1. What to add/change in the SQL
2. Whether to use `CREATE OR ALTER` vs. recreate
3. Verification queries to confirm correctness after deployment

---

## Critical Rules

### Never Guess — Always Surface

The primary value of this skill is catching assumptions before they become bugs. Apply this hierarchy:

| Situation | Action |
|-----------|--------|
| PRD explicitly states the mapping/rule | Implement as stated |
| PRD is silent but a pattern exists in the current DT | Note the pattern, recommend following it, flag as assumption |
| PRD is contradictory or ambiguous | Put in Section D, do NOT pick a side |
| PRD says "TBD" or "open question" | Put in Section D verbatim with the stated owner |

### Scope Discipline

- Only recommend changes to the named `target_dynamic_table`.
- If a rule belongs to a different layer (Gold, DMF, alerting), say so explicitly in Section C.
- If a rule requires a new table/view that doesn't exist yet, flag it as a dependency in Section E.

### Breaking Change Awareness

When a rule changes output for existing rows (e.g., status normalization), explicitly call it out:
- What the current output is
- What it will become
- Whether downstream consumers need to be notified

## Stopping Points

- **After Step 1**: If the file format is unclear or has multiple candidate sheets/sections, ask before proceeding.
- **After Step 4 Section D**: If there are high-severity open questions, pause and ask the user whether to proceed with the plan or wait for resolution.

## Output Format

Return the five sections (A–E) as markdown. Keep each section concise — bullet points and tables preferred over prose. End with a one-line summary of scope: how many new sources, how many rule changes, how many open questions.

## Example Usage

**User prompt:**
```
$prd-to-pipeline-plan
prd_path: ./requirements/Q3_source_onboarding.xlsx
target_dynamic_table: COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
```

**Expected output structure:**
```
## A: New Source Systems
- BAAN (BRONZE_BAAN_AP_INVOICES, 10 rows, 16 columns → 16 Silver columns mapped)
- WORKDAY (BRONZE_WORKDAY_AP_INVOICES, 10 rows, 16 columns → 16 Silver columns mapped)

## B: Rules Affecting SILVER_AP_INVOICES
- BR-001: Status normalization → adds CASE statement to all 4 branches
- BR-003: Baan dedup → adds QUALIFY to Baan branch only
  ⚠️ Breaking change: N/A (new source, no existing rows affected)

## C: Rules NOT Affecting This DT
- BR-002: Currency conversion (Gold layer)
- BR-004: Amount threshold alerts (DMF)
- BR-006: GL cross-reference (Gold, Phase 2)

## D: Open Questions
1. BR-005: Payment terms — normalize at Silver or Gold? (Owner: Sarah/David, due 2025-06-20)
2. Baan cost center format change — pass through or standardize?

## E: Implementation Steps
1. Add Baan UNION ALL branch with QUALIFY dedup
2. Add Workday UNION ALL branch
3. Add CASE statement for APPROVAL_STATUS across all 4 branches
4. Use CREATE OR ALTER to preserve refresh history
5. Verify: SELECT SOURCE_SYSTEM, COUNT(*) ... should show 4 systems, ~50 rows

---
Summary: 2 new sources, 2 DT rule changes, 2 open questions requiring business input.
```
