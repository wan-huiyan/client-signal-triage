---
name: client-signal-triage
version: 1.1.0
date: 2026-03-25
author: wan-huiyan
description: >
  Triage a batch of candidate data signals (from client emails, meeting notes,
  CRM field lists, funnel definitions) into GO/DEFER/NO-GO decisions based on
  data availability checks. Produces a signal inventory table and hands off GO
  candidates to ml-feature-evaluator for deep diagnostic.
triggers:
  positive:
    - "The client sent an email with these fields..."
    - "Here's a list of signals we should evaluate"
    - "Can you check which of these fields are available in BQ?"
    - "Triage these candidate features"
    - "The client's funnel definition mentions these fields..."
    - "Check data availability for these signals"
    - "Which of these fields should we add to the model?"
    - "Parse this client email for potential model signals"
    - "We got a new field list from the client team"
    - "Assess these candidate data sources"
    - "What signals can we actually use from this list?"
    - "The client mentioned these data points in the meeting"
    - "Here are the CRM fields they want us to look at"
    - "Which of these Salesforce/HubSpot/CRM fields exist in our warehouse?"
    - "Bulk check these columns for coverage"
    - "We received a field mapping document — what's usable?"
    - "Sort these candidate signals by feasibility"
    - "Run signal triage on these fields"
    - "Data availability check for client fields"
    - "Classify these candidate features as GO or NO-GO"
  negative:
    - "Evaluate the AUC of this single feature"
    - "Should we extend the training window?"
    - "What hyperparameters should I tune?"
    - "Deploy this model to production"
    - "Review this pull request"
    - "Fix this failing test"
    - "What model architecture should I use?"
    - "Run a deep diagnostic on conversion_flag"
    - "Compare XGBoost vs LightGBM"
    - "Write unit tests for the pipeline"
scope: >
  Batch triage of multiple candidate signals against a data warehouse. Not for
  single-feature deep evaluation, model architecture, deployment, or code review.
input: >
  A batch of candidate signals — from client email text, meeting notes, a CRM
  field list, or a funnel definition document — plus context about the target
  data warehouse (BigQuery, Snowflake, Postgres, etc.) and the model these
  signals might feed into.
output: >
  Signal inventory table (Signal | Source Field | Table | Coverage | Values |
  Assessment | Phase), composite signal candidates, and a prioritized list of
  GO candidates ready for ml-feature-evaluator deep diagnostic.
output_contract:
  format: markdown_table
  required_columns:
    - Signal
    - Source Field
    - Table
    - Coverage
    - Assessment
    - Phase
  sections:
    - signal_inventory_table
    - composite_candidates
    - next_steps
consumes_from:
  - name: client-email-parser
    provides: raw field list from client communications
  - name: brainstorming
    provides: open-ended feature ideas to triage
hands_off_to:
  - name: ml-feature-evaluator
    when: GO candidates need deep quantitative diagnostic
    passes: signal name, source table, coverage stats
  - name: ml-training-window-assessor
    when: temporal guard assessment reveals training window questions
    passes: field name, temporal availability range
version_compat: ">=1.0.0"
dependencies: >
  Access to the data warehouse containing candidate fields. INFORMATION_SCHEMA
  access for column discovery. Optionally, knowledge of the target model's
  existing feature set.
idempotent: true
error_behavior: >
  If data warehouse access fails, report which signals could not be checked and
  provide best-effort classification based on metadata alone. If INFORMATION_SCHEMA
  is unavailable, fall back to direct SELECT queries with LIMIT.
namespace: >
  Outputs are scoped to the current conversation. No files written unless the
  user requests a signal inventory saved to disk.
---

# Client Signal Triage

Triage a batch of candidate data signals into GO / DEFER / NO-GO decisions
based on data availability, coverage, and temporal feasibility. Handles the
broad sweep so `ml-feature-evaluator` can focus deeply on GO candidates.

**Use this skill when** you have a batch of candidate signals to triage against a data warehouse. For single-feature deep evaluation, use `ml-feature-evaluator` instead.

## Routing Table

| Situation | Use |
|-----------|-----|
| Client sent a list of 10+ fields to evaluate | **This skill** |
| One specific feature needs full diagnostic | `ml-feature-evaluator` |
| Question is about extending training history | `ml-training-window-assessor` |
| Open-ended "what features could we add?" | `brainstorming` first, then this skill |

## Workflow (follow in order)

### Phase 1: Parse Client Input

1. Read client input (email, meeting notes, field list, funnel definition).
2. Extract every mentioned field, signal, metric, or data point into a **candidate list**.
3. Normalize names: strip whitespace, convert to snake_case, note original client-facing name.
4. Group by source domain (CRM, behavioral, financial, demographic, etc.).
5. Flag "future data points" as aspirational — track but do not plan for.

**Output:** Numbered candidate list: `#`, `Client Name`, `Normalized Name`, `Domain`.

### Phase 2: Data Warehouse Availability Check

Run these checks for each candidate. Batch queries where possible.

<details>
<summary>Step 2a: Column Discovery — SQL templates per warehouse</summary>

Search for matching columns across all relevant tables. Client field names rarely match warehouse names — always use fuzzy matching.

```sql
-- BigQuery
SELECT table_name, column_name, data_type
FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE LOWER(column_name) LIKE '%search_term%'
ORDER BY table_name, column_name;
```

```sql
-- Snowflake
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'SCHEMA_NAME'
  AND LOWER(column_name) LIKE '%search_term%';
```

```sql
-- Postgres
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND column_name ILIKE '%search_term%';
```

**Key rule:** Always search MULTIPLE tables. For Salesforce-backed warehouses, check Account, Contact, Lead, Opportunity, Application, etc.

If no match found:
- Broader LIKE patterns (e.g., `%loyalty%` instead of `%loyalty_program_tier%`)
- Check related tables (field might be a join away)
- Ask user for field name mapping clarification

</details>

<details>
<summary>Step 2b: Coverage Check — SQL templates</summary>

```sql
-- Coverage + cardinality + top values (single query per field)
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(field_name IS NOT NULL) AS non_null_count,
  ROUND(COUNTIF(field_name IS NOT NULL) / COUNT(*) * 100, 1) AS coverage_pct,
  COUNT(DISTINCT field_name) AS distinct_values,
  APPROX_TOP_COUNT(field_name, 10) AS top_values
FROM `project.dataset.table_name`;
```

```sql
-- For numeric fields: distribution summary
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(field_name IS NOT NULL) AS non_null_count,
  ROUND(COUNTIF(field_name IS NOT NULL) / COUNT(*) * 100, 1) AS coverage_pct,
  MIN(field_name) AS min_val, MAX(field_name) AS max_val,
  APPROX_QUANTILES(field_name, 4) AS quartiles
FROM `project.dataset.table_name`;
```

</details>

#### Step 2c: Temporal Guard Assessment

| Scenario | Temporal Guard | Action |
|----------|---------------|--------|
| Has independent timestamp (`created_date`, `event_timestamp`) | `WHERE field_date <= target_date` | Straightforward |
| Static attribute (`gender`, `country`, `industry_segment`) | Not needed | Safe to use |
| Derived from timestamped event (`last_login_date`) | Via event date | Use event log timestamp |
| No timestamp available | None | DEFER unless proven static |

### Phase 3: Classification

#### Decision Matrix

| Classification | Criteria | Action |
|---------------|----------|--------|
| **GO** | Coverage >20% AND (has temporal guard OR static) AND clear signal potential | Hand off to `ml-feature-evaluator` |
| **DEFER** | Coverage 2-20%, OR no temporal guard for non-static field, OR sparse but valuable | Document for future phase |
| **NO-GO** | Coverage <2%, OR <10 non-null records, OR single value >99%, OR ethical/legal block | Document reason and skip |

**Classification nuances (always apply):**
- Thresholds are guidelines, not hard rules. A 5% field with strong signal (e.g., demo request flag) can be GO.
- Schema existence does not mean data existence. A column with 6 non-null rows out of 93K is effectively empty.
- Demographic fields (gender, race, ethnicity, religion, disability) require ethical/legal review flag before use in propensity models.
- Composite potential overrides individual NO-GO — two weak signals combining into a meaningful composite may be GO.

### Phase 4: Signal Inventory Table

```
| # | Signal | Source Field | Table | Coverage | Top Values | Assessment | Phase |
|---|--------|-------------|-------|----------|------------|------------|-------|
| 1 | Loyalty Tier | loyalty_program_tier | crm_account | 34.2% | Gold (40%), Silver (30%)... | GO — ordinal signal, has created_date guard | Current |
| 2 | Demo Request | demo_requested_flag | crm_lead | 8.1% | TRUE (8.1%), NULL (91.9%) | GO — low coverage but high-value for converted pop | Current |
| 3 | Fax Opt-In | fax_opt_in | crm_contact | 0.006% | TRUE (6 rows) | NO-GO — effectively empty | Skip |
| 4 | Household Income | household_income_bracket | crm_financial | 12.3% | NULL handling unclear | DEFER — no temporal guard, borderline coverage | Phase 2 |
```

### Phase 5: Composite Signal Identification

Scan for composite opportunities across three categories:

| Type | Description | Example |
|------|-------------|---------|
| Disambiguation | Two fields resolve ambiguity in a third | `country_code` + `has_filed_tax_return` = eligibility disambiguation |
| Interaction | Combination has different predictive meaning than either alone | `loyalty_tier` + `purchase_stage` — loyalty only meaningful after first purchase |
| Coverage | Two partial-coverage fields together cover larger population | `sat_score` + `act_score` -> standardized test indicator |

### Phase 6: Handoff

1. **Summarize GO count** — "X of Y candidate signals classified as GO."
2. **Prioritize GO candidates** — Order by expected impact (coverage x signal strength estimate).
3. **Recommend evaluation order** — Which GO candidates to evaluate first with `ml-feature-evaluator`.
4. **List DEFER candidates with revisit conditions** — What must change for each DEFER to become GO.
5. **Confirm NO-GO candidates** — Brief rationale for each.

## Pitfalls Checklist

| Pitfall | Rule |
|---------|------|
| Trusting schema existence as data existence | Always run coverage checks — columns can be 99.99% NULL |
| Assuming client names match warehouse names | Use LIKE queries with creative search terms |
| Checking only one table | CRM fields migrate between objects — search broadly |
| Ignoring temporal leakage risk | A field without temporal guard is unusable in temporal models |
| Over-indexing on coverage % | 5% coverage with strong signal can beat 90% with weak signal |
| Forgetting ethical review for demographics | Protected class attributes require explicit review |

## Output Formatting

- Signal inventory as markdown table; NO-GO candidates grouped at bottom with rationale.
- Fields needing ethical/legal review marked with `[ETHICS REVIEW]`.
- Composites in a separate section after main inventory.
- End with "Next Steps" listing GO candidates for `ml-feature-evaluator`.

## Dependencies and Compatibility

This skill depends on access to a SQL-compatible data warehouse (BigQuery, Snowflake, or Postgres). Requires INFORMATION_SCHEMA access for column discovery; alternatively, if not available, falls back to direct SELECT queries. Compatible with ml-feature-evaluator v1.0 and ml-training-window-assessor v1.0. Minimum version of this skill: 1.0.0.
