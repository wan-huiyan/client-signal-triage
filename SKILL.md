---
name: client-signal-triage
version: 1.0.0
date: 2026-03-24
author: wan-huiyan
description: >
  Triage a batch of candidate data signals (from client emails, meeting notes,
  CRM field lists, funnel definitions) into GO/DEFER/NO-GO decisions based on
  data availability checks. Produces a signal inventory table and hands off GO
  candidates to ml-feature-evaluator for deep diagnostic.
  Triggers when the user asks any of these (or close variations):
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
  Other natural-language variations that should trigger this skill:
  - "The client mentioned these data points in the meeting"
  - "Here are the CRM fields they want us to look at"
  - "Which of these Salesforce/HubSpot/CRM fields exist in our warehouse?"
  - "Bulk check these columns for coverage"
  - "We received a field mapping document — what's usable?"
  - "Sort these candidate signals by feasibility"
  Does NOT trigger for: deep quantitative evaluation of a single feature (use
  ml-feature-evaluator), training window extension questions (use
  ml-training-window-assessor), model selection, hyperparameter tuning,
  deployment, code review, testing, or bug fixes. If the user has already
  narrowed down to ONE specific feature and wants full diagnostic, route to
  ml-feature-evaluator instead.
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
based on data availability, coverage, and temporal feasibility. This skill
handles the broad sweep — checking many signals quickly — so that
`ml-feature-evaluator` can focus deeply on the ones that pass triage.

## When to Use This vs. Related Skills

| Situation | Use |
|-----------|-----|
| Client sent a list of 10+ fields to evaluate | **This skill** |
| One specific feature needs full diagnostic | `ml-feature-evaluator` |
| Question is about extending training history | `ml-training-window-assessor` |
| Open-ended "what features could we add?" | `brainstorming` first, then this skill |

## Workflow (follow in order)

### Phase 1: Parse Client Input

1. Read the client input (email, meeting notes, field list, funnel definition).
2. Extract every mentioned field, signal, metric, or data point into a **candidate list**.
3. Normalize names: strip whitespace, convert to snake_case, note the original client-facing name.
4. Group candidates by likely source domain (CRM, behavioral, financial, demographic, etc.).
5. Flag any "future data points" the client mentions as aspirational — track but do not plan for.

**Output:** A numbered candidate list with columns: `#`, `Client Name`, `Normalized Name`, `Domain`.

### Phase 2: Data Warehouse Availability Check

For each candidate signal, run these checks in order. Batch queries where possible to minimize round trips.

#### Step 2a: Column Discovery

Search for matching columns across all relevant tables. Client field names often do not match warehouse column names — always use fuzzy matching.

```sql
-- BigQuery: Search for a column across all tables in a dataset
SELECT table_name, column_name, data_type
FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE LOWER(column_name) LIKE '%search_term%'
ORDER BY table_name, column_name;
```

```sql
-- Snowflake equivalent
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'SCHEMA_NAME'
  AND LOWER(column_name) LIKE '%search_term%';
```

```sql
-- Postgres equivalent
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND column_name ILIKE '%search_term%';
```

**Key pattern:** Always search MULTIPLE tables. Fields often live in unexpected places. For Salesforce-backed warehouses, check all object tables (Account, Contact, Lead, Opportunity, Application, etc.) — not just the one you expect.

If no INFORMATION_SCHEMA match is found, try:
- Broader LIKE patterns (e.g., `%loyalty%` instead of `%loyalty_program_tier%`)
- Checking related tables (the field might be a join away)
- Asking the user for clarification on the field name mapping

#### Step 2b: Coverage Check

For each discovered column, measure coverage and distribution:

```sql
-- Coverage + cardinality + top values (single query per field)
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(field_name IS NOT NULL) AS non_null_count,
  ROUND(COUNTIF(field_name IS NOT NULL) / COUNT(*) * 100, 1) AS coverage_pct,
  COUNT(DISTINCT field_name) AS distinct_values,
  -- For categorical fields: top values
  APPROX_TOP_COUNT(field_name, 10) AS top_values
FROM `project.dataset.table_name`;
```

```sql
-- For numeric fields: distribution summary
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(field_name IS NOT NULL) AS non_null_count,
  ROUND(COUNTIF(field_name IS NOT NULL) / COUNT(*) * 100, 1) AS coverage_pct,
  MIN(field_name) AS min_val,
  MAX(field_name) AS max_val,
  APPROX_QUANTILES(field_name, 4) AS quartiles
FROM `project.dataset.table_name`;
```

#### Step 2c: Temporal Guard Assessment

For each field, determine whether a temporal guard is feasible:

- **Has independent timestamp?** (e.g., `created_date`, `event_timestamp`) — temporal guard is straightforward: `WHERE field_date <= target_date`
- **Static attribute?** (e.g., `gender`, `country`, `industry_segment`) — no temporal guard needed; value does not change over time
- **Derived from a timestamped event?** (e.g., `last_login_date` derived from event log with dates) — temporal guard via event date
- **No timestamp available?** — field cannot be safely used in temporal models without risking leakage. Classify as DEFER unless proven static.

### Phase 3: Classification

Apply the decision matrix to each candidate signal:

#### Decision Matrix

| Classification | Criteria | Action |
|---------------|----------|--------|
| **GO** | Coverage >20% AND (has temporal guard OR is static attribute) AND clear signal potential | Add to current implementation phase. Hand off to `ml-feature-evaluator` for deep diagnostic. |
| **DEFER** | Coverage 2-20%, OR no temporal guard available for a non-static field, OR sparse but potentially valuable signal | Document for future phase. Revisit when coverage improves or temporal guard becomes available. |
| **NO-GO** | Coverage <2%, OR effectively empty (<10 non-null records), OR fundamental data quality issues (e.g., single value dominates >99%), OR ethical/legal concerns without mitigation | Document reason and skip. |

**Important nuances:**

- **Thresholds are guidelines, not hard rules.** A 5% coverage field with very strong signal (e.g., demo request flag) might still be GO if the population it covers is important.
- **Effectively empty fields are common.** A column may exist in the schema but contain only a handful of values (e.g., `referral_source_detail` with 6 non-null values out of 93K rows). Always check actual counts, not just schema existence.
- **Demographic fields need flagging.** Gender, race, ethnicity, religion, disability status — these require ethical/legal review before adding to propensity models. Flag them with a note: "Requires ethical/legal review before use in scoring models." They can still be GO for descriptive analytics.
- **Composite potential overrides individual NO-GO.** If two individually weak signals combine into a meaningful composite (e.g., `country_code` + `has_filed_tax_return` = domestic financial aid eligibility disambiguation), the composite may be GO even if individual signals are marginal.

### Phase 4: Signal Inventory Table

Produce the final signal inventory in this format:

```
| # | Signal | Source Field | Table | Coverage | Top Values | Assessment | Phase |
|---|--------|-------------|-------|----------|------------|------------|-------|
| 1 | Loyalty Tier | loyalty_program_tier | crm_account | 34.2% | Gold (40%), Silver (30%)... | GO — clear ordinal signal, has created_date guard | Current |
| 2 | Demo Request | demo_requested_flag | crm_lead | 8.1% | TRUE (8.1%), NULL (91.9%) | GO — low coverage but high-value signal for converted pop | Current |
| 3 | Fax Opt-In | fax_opt_in | crm_contact | 0.006% | TRUE (6 rows) | NO-GO — effectively empty | Skip |
| 4 | Household Income | household_income_bracket | crm_financial | 12.3% | NULL handling unclear | DEFER — no temporal guard, coverage borderline | Phase 2 |
```

### Phase 5: Composite Signal Identification

After individual classification, scan for composite opportunities:

1. **Disambiguation composites** — Two fields that together resolve ambiguity in a third (e.g., `country_code` + `has_filed_tax_return` disambiguates financial aid eligibility for international customers).
2. **Interaction composites** — Fields whose combination has different predictive meaning than either alone (e.g., `loyalty_tier` + `purchase_stage` — loyalty only meaningful after first purchase).
3. **Coverage composites** — Two partial-coverage fields that together cover a larger population (e.g., `sat_score` + `act_score` -> standardized test indicator).

List any identified composites with the component signals and rationale.

### Phase 6: Handoff

1. **Summarize GO count** — "X of Y candidate signals classified as GO."
2. **Prioritize GO candidates** — Order by expected impact (coverage x signal strength estimate).
3. **Recommend evaluation order** — Which GO candidates to evaluate first with `ml-feature-evaluator`.
4. **List DEFER candidates with revisit conditions** — What would need to change for each DEFER to become GO.
5. **Confirm NO-GO candidates** — Brief rationale for each, so the client team understands why.

## Common Pitfalls

1. **Trusting schema existence as data existence.** A column in INFORMATION_SCHEMA may be 99.99% NULL. Always run coverage checks.
2. **Assuming client field names match warehouse names.** They rarely do. Use LIKE queries and be creative with search terms.
3. **Checking only one table.** CRM fields migrate between objects. A field called `created_date` might live in Account, Contact, Lead, or a custom object. Search broadly.
4. **Ignoring temporal leakage risk.** A field with great coverage and signal is useless if it cannot be temporally gated. Always assess this.
5. **Over-indexing on coverage percentage.** A 5% coverage field that perfectly predicts conversion for a specific subpopulation may be more valuable than a 90% coverage field with weak signal. Context matters.
6. **Forgetting ethical review for demographic fields.** Protected class attributes require explicit review before use in predictive models, regardless of their predictive power.

## Output Formatting

- Always present the signal inventory as a markdown table.
- Group NO-GO candidates at the bottom with brief rationale.
- Highlight any fields that need ethical/legal review with a clear marker.
- If composites are identified, present them in a separate section after the main inventory.
- End with a clear "Next Steps" section listing which GO candidates to evaluate first.
