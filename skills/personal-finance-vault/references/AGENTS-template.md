# AGENTS.md template

Copy into the vault repository as `AGENTS.md` and adapt the bracketed parts.
These rules are what make the vault safe to hand to *any* agent session.

```markdown
# Repository workflow

- At the start of a session, run `python scripts/check_data_freshness.py`
  and tell the user which source documents are missing or due soon. Offer to
  import anything they provide.
- Treat `Source/` as the canonical records. Name files by coverage:
  statements `YYYY-MM-DD.<ext>` (period end), monthly documents
  `YYYY-MM.<ext>`, range exports `<start>_<end>.<ext>`.
- Each document type lives in its own `Source/<type>/<institution>/`
  directory with its own converter script and its own normalized outputs.
  Do not repurpose another type's directory or schema.
- Treat `data/normalized/`, `data/quality/`, and `data/derived/` as
  reproducible outputs. Never hand-edit them; rerun the converter:
  [list your converters here, e.g. `python scripts/convert_payslips.py`].
- Review the quality checks after every rebuild. Do not describe the dataset
  as clean while any check is in `mismatch` status or an unexplained warning
  remains.
- Import new documents through the transactional importer
  [name it here, e.g. `python scripts/import_statement.py`]: conflicts are
  rejected, validation runs against a staged rebuild, and a failed import
  leaves the canonical dataset untouched.
- Treat external transfers into investment accounts as cash flows, never as
  investment gains. Treat planned events (scheduled repayments, expected
  wires) as plans, never as completed transactions, until a statement
  confirms them.
- Facts that cannot be derived from documents live in
  `data/personal/financial_context.json`. Unknown values are null and listed
  under `unknown_values_must_not_be_inferred` - never guessed. When new
  documents confirm a value, update it and note what superseded it.
- Document passwords live only in [untracked file / env var - name it here];
  never write them into tracked files. Never commit `data/raw_text/` - it
  duplicates sensitive document content and is regenerated locally.
- This repository is private and contains unredacted personal financial
  data. Never copy its contents into public repositories or services. Commit
  or push only when the user explicitly asks.
```
