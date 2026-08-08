# Derived financial model

A mature vault should expose a small, deterministic financial-model layer between reconciled facts and consumers such as agents, CLI tools, and HTML reports. The purpose is not to create another source of truth. It is to give every consumer the same reproducible interpretation of the canonical facts.

## Recommended outputs

When the required source domains exist and their quality gates pass, generate these under `data/derived/`:

- `cashflow_monthly.csv` — conservative monthly cash-flow aggregates whose names match what is actually classified.
- `net_worth_monthly.csv` — historical balance-sheet series with dated components and accessibility tiers.
- `portfolio_risk.json` — current concentration, drawdown/volatility context, and other reproducible portfolio-risk measures.
- `financial_snapshot.json` — compact latest-known financial state assembled from the other derived datasets plus user-owned context.

These files are reproducible outputs, never hand-edited facts.

## Snapshot-first agent behavior

For broad questions such as “How are my finances?”, “What is my net worth?”, or “What changed?”, an agent should:

1. Read `financial_snapshot.json` first when it exists and is fresh.
2. Inspect its component dates, quality state, and known gaps before quoting headline totals.
3. Drill into other derived datasets, the query layer, normalized facts, and finally source documents only when the question needs more detail or the snapshot needs to be challenged.
4. Never recompute a competing headline ad hoc when the vault already has a valid deterministic model. If the model is wrong, fix the model once and rebuild it.

This is an interface rule, not an authority rule: `Source/` and user-owned facts remain canonical.

## Mixed-date honesty

Financial domains rarely arrive on the same day. A bank balance may be dated the 13th, a brokerage statement month-end, a loan balance the 1st, and a pension snapshot later still.

A latest-known snapshot may combine those components, but it must not pretend they are a same-day balance sheet. Include at least:

```json
{
  "as_of": "2031-08-06",
  "as_of_policy": "latest-known-components",
  "component_date_range": {
    "oldest": "2031-07-13",
    "newest": "2031-08-06"
  },
  "known_gaps": []
}
```

Each material component should carry its own `as_of` date. Reports and agents must describe a mixed-date total as “known assets/liabilities using latest-known components”, not as a precise balance sheet for the newest date.

If the use case requires a true month-end series, gate each month on the required month-end inputs rather than forward-filling silently.

## Cash-flow semantics

**Bank inflow is not automatically income. Bank outflow is not automatically spending.**

Loan proceeds, transfers between owned accounts, brokerage funding, refunds, reimbursements, and card settlements can all make naive cash-flow labels false. Until classification is sufficiently complete, prefer narrow factual names such as:

- `salary_in`
- `other_bank_inflows`
- `cash_outflow_ex_brokerage`
- `brokerage_contribution`

Only publish stronger fields such as `income`, `living_expense`, or `savings_rate` after the underlying transfer/dedup classification makes those semantics defensible. Include an unclassified amount/share when relevant.

## Consumer contract

Derived financial-model outputs are stable interfaces for multiple consumers:

```text
reconciled facts
      ↓
derived financial model
      ├── agent Q&A
      ├── CLI
      ├── automated checks
      └── HTML report
```

Consumers must not each implement their own hidden accounting rules. In particular, an HTML report is a renderer of the financial model, not a second calculation engine. If a displayed number is wrong, correct the deterministic model and regenerate every consumer.

## Refresh policy

Default to a deterministic full rebuild of the query layer and derived financial model after a successful transactional import. For normal personal-finance datasets this is usually cheap and avoids stale-cache bugs.

Only introduce dependency-aware incremental refresh when measured rebuild cost justifies the added invalidation complexity. Whether full or incremental, a failed import or failed quality gate must leave the previously valid model untouched.

Recommended maintenance flow:

```text
new source
  ↓
transactional import
  ↓
quality + reconciliation
  ↓
rebuild query layer
  ↓
refresh derived financial model
  ↓
refresh reports
```

Historical values may legitimately change when an old missing document is imported or a parser/accounting rule is corrected. Such changes should be reproducible and explainable from changed inputs or code, never from manual edits.

## Minimal generic snapshot shape

Keep the public skill generic and use fictional examples only:

```json
{
  "as_of": "2031-08-06",
  "as_of_policy": "latest-known-components",
  "net_worth": {
    "known_assets": 0,
    "known_liabilities": 0,
    "known_net_worth": 0,
    "components": {
      "liquid_assets": {"value": 0, "as_of": "2031-07-31"},
      "brokerage": {"value": 0, "as_of": "2031-07-31"},
      "age_gated_assets": {"value": 0, "as_of": "2031-08-06"}
    }
  },
  "known_gaps": []
}
```

The exact schema may evolve with the vault. Preserve the principles: reproducible, conservative semantics, component-level dates, explicit gaps, and one shared model for every consumer.
