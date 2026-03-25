# Client Signal Triage

Batch triage candidate data signals from client emails, CRM field lists, and funnel definitions into GO/DEFER/NO-GO decisions.

When a client sends a list of 15 CRM fields they want evaluated for a churn model, this skill systematically checks each one against your data warehouse — coverage, temporal safety, composite potential — and produces a prioritized signal inventory in minutes instead of hours.

## Quick Start

```
You: The client sent over these fields they want us to evaluate for the
     SaaS churn model: last_login_date, contract_value, support_tickets_count,
     nps_score, feature_adoption_pct, referral_source, industry_segment,
     company_size, payment_method, billing_cycle, onboarding_completed,
     csm_assigned, health_score, renewal_date, expansion_revenue

Claude: [Runs 6-phase triage workflow]

     Signal Inventory:
     | # | Signal              | Coverage | Assessment                              | Phase   |
     |---|---------------------|----------|-----------------------------------------|---------|
     | 1 | Last Login Date     | 94.2%    | GO — strong recency signal, timestamped | Current |
     | 2 | Contract Value      | 88.7%    | GO — clear numeric signal               | Current |
     | 3 | Support Tickets     | 72.1%    | GO — behavioral signal, event-dated     | Current |
     | 4 | NPS Score           | 31.4%    | GO — survey response bias, but usable   | Current |
     | 5 | Referral Source     | 12.3%    | DEFER — no temporal guard, borderline   | Phase 2 |
     | 6 | Expansion Revenue   | 0.8%     | NO-GO — effectively empty               | Skip    |
     ...

     Composites identified:
     - health_score + csm_assigned → managed account risk indicator

     Next: Evaluate last_login_date and support_tickets_count first
     with ml-feature-evaluator (highest coverage x signal potential).
```

## Installation

### Claude Code

```bash
git clone https://github.com/wan-huiyan/client-signal-triage.git ~/.claude/skills/client-signal-triage
```

### Cursor

Copy the contents of `SKILL.md` into a per-project rule (`.cursor/rules/client-signal-triage.mdc`).

## What You Get

| Output | Description |
|--------|-------------|
| **Signal Inventory Table** | Every candidate field with coverage %, top values, GO/DEFER/NO-GO assessment, and implementation phase |
| **Composite Identification** | Pairs of individually weak signals that combine into strong composites (disambiguation, interaction, or coverage composites) |
| **Prioritized Handoff** | GO candidates ordered by expected impact, ready for deep evaluation with `ml-feature-evaluator` |
| **DEFER Revisit Conditions** | What needs to change for each deferred signal to become actionable |
| **Ethical/Legal Flags** | Demographic fields automatically flagged for review before use in scoring models |

## Comparison

| | Ad-Hoc Field Checking | With This Skill |
|---|---|---|
| **Process** | Check fields one at a time, lose track of what was checked | Systematic 6-phase workflow, full inventory |
| **Coverage** | "The column exists" (schema only) | Actual non-null counts, distribution, top values |
| **Temporal Safety** | Often forgotten until leakage is discovered | Assessed for every field upfront |
| **Composites** | Rarely considered | Systematically scanned after individual classification |
| **Handoff** | Unstructured notes | Prioritized GO list ready for `ml-feature-evaluator` |
| **Time** | 2-4 hours for 15 fields | 15-30 minutes |

## How It Works

The skill follows a 6-phase workflow:

| Phase | What Happens |
|-------|--------------|
| **1. Parse Client Input** | Extract candidate signals, normalize names, group by domain |
| **2. Data Warehouse Check** | Column discovery via INFORMATION_SCHEMA, coverage stats, distribution analysis |
| **3. Temporal Guard Assessment** | Classify each field: has timestamp, static attribute, derived from event, or ungated |
| **4. Classification** | Apply GO/DEFER/NO-GO decision matrix with coverage thresholds and nuance rules |
| **5. Composite Identification** | Scan for disambiguation, interaction, and coverage composite opportunities |
| **6. Handoff** | Summarize, prioritize GO candidates, document DEFER revisit conditions |

Supports BigQuery, Snowflake, and Postgres with warehouse-specific SQL templates.

## Related Skills

| Skill | When to Use Instead |
|-------|-------------------|
| `ml-feature-evaluator` | Deep quantitative diagnostic of a single GO candidate |
| `ml-training-window-assessor` | Questions about extending training history or lookforward windows |
| `brainstorming` | Open-ended "what features could we add?" before you have a candidate list |

## Limitations

- **Requires data warehouse access.** The skill runs SQL queries against your warehouse. Without access, it falls back to metadata-only classification (less accurate).
- **Coverage thresholds are guidelines.** The 20%/2% GO/NO-GO thresholds work well in practice but should be adjusted for domain-specific contexts (e.g., rare events may justify lower thresholds).
- **Does not replace domain expertise.** The skill checks data availability and temporal safety, not whether a signal makes domain sense. A GO classification means "data is available and safe to use," not "this will definitely improve the model."
- **Single-warehouse scope.** Each triage run targets one warehouse. Cross-warehouse signal evaluation requires separate runs.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-03-24 | Initial release: 6-phase workflow, GO/DEFER/NO-GO matrix, composite identification, multi-warehouse SQL templates |

## License

MIT
