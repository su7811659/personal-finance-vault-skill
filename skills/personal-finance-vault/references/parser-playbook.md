# Parser playbook

How to turn one source type's documents into trustworthy CSV. Battle-tested
order of operations - skipping steps is how silent data corruption happens.

## 1. Inspect before you code

Open documents from the **start, middle, and end** of the date range.
Layouts drift: columns appear, sections get added, label wording changes.
Note every variant you must handle; expect more in old documents.

## 2. Extract text, and know the failure modes

- **PDF text extraction can be lossy.** CJK-labeled PDFs often lack a usable
  ToUnicode mapping: `pdftotext` returns numbers but garbled or missing
  labels. Two escapes: try a different extraction engine (pdfium via
  `pypdfium2` frequently succeeds where poppler fails), and normalize the
  result with Unicode NFKC (fonts sometimes map to Kangxi-radical
  codepoints that look identical but do not string-match).
- **When text fails entirely, go visual.** Render one or two pages to PNG,
  read them as images to learn the layout and labels, then return to
  programmatic extraction for the actual data. Vision is for *learning the
  layout*, never the production data path.
- **Excel exports from web banking are not clean tables.** Expect title
  rows, key-value metadata rows, a footer row ("N records, exported at..."),
  ragged short rows where trailing cells are omitted, and newest-first
  ordering you should reverse to store oldest-first.
- **Local calendars.** Convert e.g. ROC years (+1911) at the parsing
  boundary; store ISO dates only.

## 3. One converter script per source type

Requirements for every converter:

- **Full rebuild, deterministic.** Scan the whole source directory, sort by
  (institution, filename), regenerate output files from scratch. Never
  append to or merge into existing CSV. Same input -> byte-identical output.
- **Traceability columns** on every row: `source_file`, `source_row`,
  `file_sha256`, `parser_version`.
- **Password resolution chain** for protected files: CLI flag, then
  environment variable, then an untracked dotfile - never a tracked file,
  never hardcoded.
- **Institution-keyed readers.** `READERS = {"BankA": read_bank_a}`; a
  directory with no reader produces a quality warning, not a crash and not
  silent skipping.
- **Two-tier categorization.** Generic keyword rules live in the script.
  Personal facts (which counterparty is the landlord, which transfer is the
  side-gig payout) live in `data/personal/<domain>_rules.csv` as
  substring -> category rows, applied before generic rules. Users extend the
  CSV; the code never changes for a new counterparty.

## 4. Mandatory quality checks

Write results to `data/quality/<domain>_checks.csv` with columns
`check, expected, actual, status` (status: matched / mismatch / warning /
review). Minimum set by domain:

| Domain | Check |
|---|---|
| Payslips | sum(items per section) == stated section subtotal; additions - deductions == stated net pay |
| Bank accounts | balance walk: `balance[i-1] + amount[i] == balance[i]` for every consecutive row |
| Credit cards | sum(purchase lines) == stated statement total; exclude payment/credit records from spending |
| Any | count of rows whose label matched no category (unmapped -> warning, listed individually) |
| Any | filename period == period stated inside the document |

**Every mismatch gets investigated before the dataset is called clean.**
Legitimate explanations (pending settlement, same-day reordering) get
recorded next to the check, not waved away.

## 5. Cross-dataset reconciliation

The strongest validation is two independent documents agreeing:

- payslip net pay == salary deposit in the bank export (per month, to the
  unit)
- card statement total == card payment debit in the bank export
- loan disbursement in the bank export == principal minus origination fee
- brokerage wire == the bank-side outgoing transfer

Run these once both sides exist, and report agreements as explicitly as
discrepancies - they are what makes the vault trustworthy.

## 6. Reverse-engineering formulas (optional but high value)

With enough history, payroll usually reveals exact employer formulas
(raise month; bonus = clean multiplier x some base). Test candidate formulas
against **every** year; a formula that fits one year is a coincidence
(averages and off-by-one-month bases often coincide for a single year).
Prefer the candidate that yields clean round multipliers across all years
with zero residual. Record the formula and its observed parameters in
`financial_context.json`, superseding the user's verbal estimates.

## 7. Freshness checker

A small script that encodes each source's cadence: statement expected N days
after period end, payslip by payday, range exports stale after ~35 days,
statement cycle closing day for cards - plus continuity scans for missed
months and pending-confirmation reminders read from the context file. Print
`[missing] / [upcoming] / [note]` lines; exit 0 always (it reports, it does
not block). Wire it into session start (Claude Code `SessionStart` hook
returning `systemMessage` + `additionalContext` JSON; a plain instruction in
AGENTS.md for other agents).
