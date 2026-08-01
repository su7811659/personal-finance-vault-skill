# Repository layout

```
<vault-repo>/                      # PRIVATE git repository
├── README.md                      # scope, layout, privacy statement
├── AGENTS.md                      # agent workflow rules (see AGENTS-template.md)
├── .gitignore                     # password files, raw text dumps, OS debris
├── .claude/settings.json          # optional: SessionStart freshness hook
├── Source/                        # canonical original documents (never edited)
│   ├── <brokerage statements>     # e.g. 2031-01-31.pdf, named by period end
│   ├── payslips/<Company>/        # 2031-01.pdf ... ; YYYY-yearend.pdf
│   ├── bank/<Bank>/               # 2030-08-01_2031-07-31.xlsx (coverage range)
│   ├── creditcard/<Bank>/         # 2031-07.xlsx (statement cycle month)
│   ├── loan/<loan-id>/            # contracts, web screenshots, dated
│   └── <future-type>/<inst>/      # every new source gets its own directory
├── scripts/                       # one deterministic converter per source type
│   ├── convert_<type>.py          # full rebuild, idempotent, sorted output
│   └── check_data_freshness.py    # what is missing or due soon
├── data/
│   ├── normalized/                # reproducible CSV, one domain per file
│   │   ├── payroll.csv            #   one row per payslip
│   │   ├── payroll_items.csv      #   one row per payslip line item
│   │   ├── bank_transactions.csv  #   one row per bank transaction
│   │   ├── creditcard_transactions.csv
│   │   └── ...
│   ├── quality/                   # reproducible check results per domain
│   │   └── <domain>_checks.csv    #   check, expected, actual, status
│   ├── derived/                   # calculated results (returns, projections)
│   └── personal/                  # manually maintained, USER-owned facts
│       ├── financial_context.json #   goals, liabilities, unknowns list
│       └── bank_counterparty_rules.csv  # substring -> category mappings
└── reports/                       # human-readable write-ups
```

## Conventions

- UTF-8 everywhere; dates `YYYY-MM-DD`; decimals without thousands
  separators or currency symbols; one currency per file, named in the docs.
- Every normalized row carries `source_file`, a `file_sha256`, and a
  `parser_version`, so any number can be traced to the document it came from
  and the code that produced it.
- Blank means unavailable - never zero.
- Positive amounts credit the account; negative amounts debit it. State the
  convention in the data README and keep it identical across domains.
- Generated files (`data/normalized/`, `data/quality/`, `data/derived/`) are
  committed for convenience but treated as disposable: any dispute is
  settled by rerunning the converter against `Source/`.

## What is canonical vs reproducible

| Layer | Canonical? | On conflict |
|---|---|---|
| `Source/` | yes | wins |
| `data/personal/` | yes (user-stated facts) | user decides |
| `data/normalized|quality|derived/` | no | regenerate |
| `reports/` | no | regenerate |
