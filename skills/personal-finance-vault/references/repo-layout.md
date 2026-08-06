# Repository layout

```
<vault-repo>/                      # PRIVATE git repository
├── README.md                      # scope, layout, privacy statement
├── AGENTS.md                      # agent workflow rules (see AGENTS-template.md)
├── .gitignore                     # password files, raw text dumps, OS debris
├── .claude/settings.json          # optional: SessionStart freshness hook
├── Source/                        # canonical original documents (never edited)
│   ├── <new raw documents>        # temporary intake: agent classifies and moves
│   ├── brokerage/<Broker>/        # e.g. 2031-01-31.pdf, named by period end
│   ├── payslips/<Company>/        # 2031-01.pdf ... ; YYYY-yearend.pdf
│   ├── bank/<Bank>/               # 2030-08-01_2031-07-31.xlsx (coverage range)
│   ├── creditcard/<Bank>/         # 2031-07.xlsx (statement cycle month)
│   ├── loan/<loan-id>/            # contracts, web screenshots, dated
│   ├── <snapshot-type>/           # point-in-time records (pension printout,
│   │                              #   insurance register): archive + a context
│   │                              #   entry citing each figure's source file -
│   │                              #   no parser; they are not a recurring series
│   └── <future-type>/<inst>/      # every new source gets its own directory
├── scripts/                       # one deterministic converter per source type
│   ├── convert_<type>.py          # full rebuild, idempotent, sorted output
│   └── check_data_freshness.py    # what is missing or due soon
├── data/
│   ├── documents.csv              # manifest: one row per source doc (hash, period, parser version)
│   ├── raw_text/                  # UNTRACKED loss-minimizing extraction layer
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

- The user may drop any new raw document directly into the top level of
  `Source/`. The agent inspects its content and moves it unchanged into the
  matching canonical subdirectory. Files that cannot be classified safely
  stay at the top level for review. Domain converters never scan the top
  level; they scan only their own `Source/<type>/<institution>/` paths.
- UTF-8 everywhere; dates `YYYY-MM-DD`; decimals without thousands
  separators or currency symbols; one currency per file, named in the docs.
- Every normalized row carries `source_file`, a `file_sha256`, and a
  `parser_version`, so any number can be traced to the document it came from
  and the code that produced it.
- Blank means unavailable - never zero.
- Positive amounts credit the account; negative amounts debit it. State the
  convention in the data README and keep it identical across domains.
- Generated files (`data/normalized/`, `data/quality/`, `data/derived/`)
  are disposable: any dispute is settled by rerunning the converter against
  `Source/`. Whether to commit them is the user's call (Phase 1): committing
  is convenient; gitignoring reduces exposure if the repo ever leaks.
- `data/raw_text/` (layout-preserving extracted text) is always gitignored:
  it duplicates sensitive document content and is regenerable, but keeping
  it locally lets improved parsers reprocess without information loss.
- `data/personal/` holds only analysis-relevant facts - no government IDs,
  no full account numbers, no identity documents, no scanned contracts
  (those stay in `Source/`).
- Add a `.gitattributes` pinning text files to one EOL style (e.g. CSV to
  LF) and marking PDFs/databases binary - byte-identical rebuilds across
  platforms depend on it. Corollary: normalize a text source's line endings
  to the pinned style **before** archiving and hashing it, so the recorded
  `file_sha256` reproduces from a fresh clone (see the parser playbook).
- `Source/` is the only irreplaceable layer (institutions purge download
  history). Recommend an encrypted backup (e.g. an encrypted archive or a
  private encrypted remote) beyond the single working copy.

## What is canonical vs reproducible

| Layer | Canonical? | On conflict |
|---|---|---|
| `Source/` | yes | wins |
| `data/personal/` | yes (user-stated facts) | user decides |
| `data/normalized`, `data/quality`, `data/derived` | no | regenerate |
| `reports/` | no | regenerate |
