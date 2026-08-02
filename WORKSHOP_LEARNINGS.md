# Cortex Code Workshop — Full Learnings & Reference

## What We Built

This workshop walked through an end-to-end data pipeline using Cortex Code:

```
Bronze Source Tables → Dynamic Table (Silver) → Semantic View → Cortex Agent
```

| Layer | Object | Purpose |
|-------|--------|---------|
| Bronze | `SOURCE_DATA.BRONZE_SAP_AP_INVOICES` | Raw SAP AP invoices (15 rows) |
| Bronze | `SOURCE_DATA.BRONZE_ORACLE_AP_INVOICES` | Raw Oracle AP invoices (15 rows) |
| Bronze | `SOURCE_DATA.BRONZE_BAAN_AP_INVOICES` | Raw Baan AP invoices (10 rows) |
| Bronze | `SOURCE_DATA.BRONZE_WORKDAY_AP_INVOICES` | Raw Workday AP invoices (10 rows) |
| Silver | `PIPELINE_LAB.SILVER_AP_INVOICES` | Unified Dynamic Table (50 rows) |
| Semantic | `PIPELINE_LAB.SV_AP_ANALYTICS` | Semantic view for natural-language queries |
| Agent | `PIPELINE_LAB.AP_ANALYTICS_ASSISTANT` | Cortex Agent for AP analytics |
| Eval | `SOURCE_DATA.AGENT_EVAL_SET` | Golden question set (15 questions) |

---

## Connection Configuration

### Option A: Password-based (simplest for workshops)

Add to `~/.snowflake/connections.toml`:

```toml
[connections.coco-workshop]
account   = "<ACCOUNT_IDENTIFIER>"
user      = "<YOUR_USERNAME>"
password  = "<YOUR_PASSWORD>"
role      = "SYSADMIN"
warehouse = "COCO_WORKSHOP_WH"
database  = "COCO_WORKSHOP"
schema    = "PIPELINE_LAB"
```

Launch: `cortex -c coco-workshop`

### Option B: PAT-based (recommended for production)

```toml
[connections.coco-workshop]
account        = "<ACCOUNT_IDENTIFIER>"
user           = "<YOUR_USERNAME>"
token          = "<YOUR_PAT>"
authenticator  = "snowflake_jwt"
role           = "SYSADMIN"
warehouse      = "COCO_WORKSHOP_WH"
database       = "COCO_WORKSHOP"
schema         = "PIPELINE_LAB"
```

Generate a PAT in Snowsight: User Menu → Preferences → Programmatic Access Tokens.

### Option C: Multi-user lab setup (shared account)

```toml
[connections.COCO_LAB]
account   = "<ACCOUNT_IDENTIFIER>"
user      = "<YOUR_USERNAME>"
password  = "<YOUR_PASSWORD>"
role      = "COCO_WORKSHOP_ROLE"
warehouse = "COCO_WORKSHOP_WH"
database  = "COCO_WORKSHOP"
schema    = "<YOUR_SCHEMA>"           # e.g. "USER_07"
```

### Password vs PAT — When to Use Each

| Method | Use When | Security |
|--------|----------|----------|
| **Password** | Workshops, demos, quick tests | Least secure. Stored in plaintext in TOML. |
| **PAT (token)** | CI/CD, automation, daily use | Scoped, rotatable, revocable. Preferred. |
| **Browser SSO** | Interactive sessions only | Most secure but requires browser. Use `authenticator = "externalbrowser"`. |

---

## How to Use the Agent

### 1. Chat via Cortex Code CLI

```
cortex -c coco-workshop
> $cortex-agent
> Chat with COCO_WORKSHOP.PIPELINE_LAB.AP_ANALYTICS_ASSISTANT
```

Or use the lite/objectless mode to test without the deployed object.

### 2. Chat via Snowsight (Snowflake Intelligence)

Navigate to: AI & ML → Cortex Agents → AP_ANALYTICS_ASSISTANT

Ask questions like:
- "What is our total spend by vendor?"
- "How many invoices are pending?"
- "Top 10 vendors by overdue amount"

### 3. Query via SQL (programmatic)

```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE_AGENT(
  'COCO_WORKSHOP.PIPELINE_LAB.AP_ANALYTICS_ASSISTANT',
  'What is total spend by vendor?'
);
```

### 4. Query via REST API

```bash
curl -X POST \
  "https://<account>.snowflakecomputing.com/api/v2/cortex/agents/COCO_WORKSHOP.PIPELINE_LAB.AP_ANALYTICS_ASSISTANT:run" \
  -H "Authorization: Bearer <PAT>" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Total spend by vendor"}]}'
```

### 5. Evaluate the Agent

Use the `AGENT_EVAL_SET` table with 15 golden questions:

```sql
SELECT * FROM COCO_WORKSHOP.SOURCE_DATA.AGENT_EVAL_SET;
```

Run formal evaluations via `$cortex-agent` → EVALUATE intent.

---

## Key Skills Used

| Skill | Purpose | Trigger |
|-------|---------|---------|
| `$dynamic-tables` | Create/monitor/troubleshoot Dynamic Tables | "create DT", "check DT status" |
| `$semantic-view` | Create/audit/debug semantic views | "create semantic view", "audit SV" |
| `$cortex-agent` | Create/test/evaluate/optimize agents | "create agent", "test agent" |
| `$prd-to-pipeline-plan` | Analyze PRDs for pipeline impact (custom) | PRD analysis |

---

## Project Skill: PRD-to-Pipeline-Plan

We created a reusable project skill at `.cortex/skills/prd-to-pipeline-plan/SKILL.md`.

**Invoke with:** `$prd-to-pipeline-plan`

**Inputs:**
- `prd_path` — path to requirements file (XLSX, CSV, markdown)
- `target_dynamic_table` — fully qualified DT name

**Returns 5 sections:**
- A: New source systems
- B: Rules affecting the DT
- C: Rules out of scope
- D: Open questions (never guessed)
- E: Implementation steps

---

## Files in This Repo

| File | Purpose |
|------|---------|
| `00_snowday_setup.sql` | Creates database, schemas, warehouse, role, tags |
| `00_sample_data.sql` | Populates all bronze tables and eval set |
| `01_admin_lab_reset.sql` | Resets lab for re-run |
| `01_demo_reset.sql` | Resets demo objects |
| `02_admin_lab_teardown.sql` | Full cleanup |
| `03_participant_connection_template.toml` | Connection template for participants |
| `sample_business_requirements_*.csv` | PRD requirements (3 files) |
| `AP_ANALYTICS_ASSISTANT_spec.json` | Agent specification (recreatable) |
| `CHANGE_LOG.md` | Change log for Silver DT update |
| `SEMANTIC_VIEW.md` | Semantic view documentation |
| `.cortex/skills/prd-to-pipeline-plan/SKILL.md` | Custom project skill |

---

## Recreating Everything from Scratch

```bash
# 1. Connect
cortex -c coco-workshop

# 2. Run setup (as SYSADMIN)
# Execute 00_snowday_setup.sql
# Execute 00_sample_data.sql

# 3. Create the Silver Dynamic Table
# (CREATE OR ALTER with all 4 sources + status normalization + Baan dedup)

# 4. Create the Semantic View
# CALL SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML(...)

# 5. Create the Agent
# CREATE OR REPLACE AGENT ... FROM SPECIFICATION $$...$$

# 6. Test
# Chat with the agent or run eval set
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `Object does not exist` on DT | Run `ALTER DYNAMIC TABLE ... REFRESH` — DOWNSTREAM lag means no auto-refresh without a consumer |
| Agent returns wrong currency totals | Check semantic view description includes currency grouping warning |
| VQR validation fails | Run `SYSTEM$CORTEX_ANALYST_SVA_TOOL` with `validate_verified_queries` |
| Connection fails | Verify `~/.snowflake/connections.toml` has correct account identifier (format: `ORGNAME-ACCOUNTNAME`) |
| Password auth rejected | Some accounts require MFA. Switch to PAT or `authenticator = "externalbrowser"` |
