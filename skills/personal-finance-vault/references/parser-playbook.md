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
- **Positional PDFs (rebuilt from character boxes) have three geometry
  traps.** (a) Group characters into lines by the **vertical center** of each
  glyph's ink box, never by its top: a short glyph - a full-width hyphen, a
  comma - has its ink at the line's optical middle, so its top sits several
  points below the tall characters beside it; top-based grouping exiles it to
  its own row or, in tight layouts, into the neighbouring line, silently
  inserting punctuation mid-name. Centers are stable: within a line they vary
  by ~1 point, adjacent lines sit several points apart. (b) Do not derive
  column boundaries from the header label's own edges - data rows are often
  not aligned to the header text, and a bound set from the label can cut the
  first cell off every value. Bound a column by the *adjacent* columns' edges
  instead. (c) Records can straddle page breaks (first line of a record at
  the bottom of one page, the rest at the top of the next); stitch pages into
  one continuous coordinate stream before attaching lines to records.
- **Keep a loss-minimizing intermediate layer.** Dump each document's
  layout-preserving extracted text to `data/raw_text/` (untracked - it
  duplicates sensitive content and is regenerable). When the parser later
  learns to recognize more row types, you reprocess from this layer without
  information loss instead of re-solving PDF extraction.
- **Excel exports from web banking are not clean tables.** Expect title
  rows, key-value metadata rows, a footer row ("N records, exported at..."),
  ragged short rows where trailing cells are omitted, and newest-first
  ordering you should reverse to store oldest-first.
- **Local calendars.** Convert e.g. ROC years (+1911) at the parsing
  boundary; store ISO dates only.
- **Hash what git will store.** If `.gitattributes` pins CSV sources to LF,
  normalize a CRLF download's line endings **before** archiving and hashing
  it - otherwise the recorded `file_sha256` can never be reproduced from a
  fresh clone. Same rule for any text source: the bytes you hash must be the
  bytes the repository preserves. Never transcode the character encoding,
  though - archive cp950/Big5 content as-is and decode at parse time.

## 3. One converter script per source type

Requirements for every converter:

- **Full rebuild, deterministic.** Scan the whole source directory, sort by
  (institution, filename), regenerate output files from scratch. Never
  append to or merge into existing CSV. Same input -> byte-identical output.
- **Traceability columns** on every row: `source_file`, `source_row`,
  `file_sha256`, `parser_version`. Additionally maintain a dataset-level
  manifest (`data/documents.csv` or per-domain equivalent): one row per
  source document with its hash, detected period, and parser version.
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
- **Overlap defense for range exports.** Files named `<start>_<end>` will
  eventually overlap (the user re-exports a window that intersects an old
  one). Detect period overlap across files of the same institution and
  report it as a quality **error** before rows are double counted - balance
  walks catch this for bank accounts, but domains without a balance chain
  (cards) fail silently.

## 4. Import new documents transactionally

The converter being deterministic is not enough; **a failed import must
leave the canonical dataset untouched.** For the recurring import flow
(new monthly statement arrives):

1. Reject conflicts up front: a file whose target name exists with a
   different hash is an error to surface, never to overwrite; a file whose
   hash already exists is `already_imported` - do not duplicate or rename.
2. Copy sources plus the candidate into a **staging directory**, run the
   full rebuild there, and run all quality checks against the staged output.
3. Only when validation passes, move the candidate into `Source/` and
   replace the canonical generated files. Keep a backup of the replaced
   outputs for one import cycle.
4. Support a validate-only mode (dry run: full staged rebuild and checks,
   zero writes to the repo).
5. Emit a machine-readable summary (status, detected period, row counts,
   warnings, reconciliation statuses) so the session can review the import
   item by item before describing it to the user as clean.

## 5. Mandatory quality checks

Write results to `data/quality/<domain>_checks.csv` with columns
`check, expected, actual, status` (status: matched / mismatch / warning /
review). Minimum set by domain:

| Domain | Check |
|---|---|
| Payslips | sum(items per section) == stated section subtotal; additions - deductions == stated net pay |
| Bank accounts | balance walk: `balance[i-1] + amount[i] == balance[i]` for every consecutive row |
| Credit cards | sum(purchase lines) == stated statement total; exclude payment/credit records from spending |
| Statements with no printed total | prove against an identity in another source (e.g. each cycle's purchases == the next cycle's direct-debit settlement in the bank account); the newest period has no successor yet - record it `not_applicable`, never as a failure |
| Any | count of rows whose label matched no category (unmapped -> warning, listed individually) |
| Any | filename period == period stated inside the document; overlap across range exports |
| Two sources carrying the same **text** | content agreement on rows matched by (date, amount) - see below |

**Arithmetic checks prove amounts; they say nothing about text columns.** A
merchant/description column can be silently wrong for years while every sum
and balance walk stays green. Whenever a second source renders the same
string (a card statement's merchant vs the deposit account's memo for the
same purchase), add a content-agreement check: match rows on (date, amount),
normalize whitespace and full-width forms, compare. Two traps: an **empty**
comparison value is an absent comparison, not an agreeing one -
`"x".startswith("")` is vacuously true, so memo-less rows silently count as
verified unless filtered; and residual disagreements that are the source
systems disagreeing with *each other* (unmappable glyphs one side omits,
separator characters rendered differently) should be folded into the
normalization so the check reads 100% and any future drop is a real
regression - a permanently-yellow check trains everyone to ignore it.

**Every mismatch gets investigated before the dataset is called clean.**
Legitimate explanations (pending settlement, same-day reordering) get
recorded next to the check, not waved away.

Also document what the parser does **not** yet recognize (a "current
limitations" section in the data README). Honesty about the tool is the
same discipline as honesty about the data.

## 6. Cross-dataset reconciliation

The strongest validation is two independent documents agreeing:

- payslip net pay == salary deposit in the bank export (per month, to the
  unit)
- card statement total == card payment debit in the bank export
- loan disbursement in the bank export == principal minus origination fee
- brokerage wire == the bank-side outgoing transfer

Run these once both sides exist, and report agreements as explicitly as
discrepancies - they are what makes the vault trustworthy.

## 7. Reverse-engineering formulas (optional but high value)

With enough history, payroll usually reveals exact employer formulas
(raise month; bonus = clean multiplier x some base). Test candidate formulas
against **every** year; a formula that fits one year is a coincidence
(averages and off-by-one-month bases often coincide for a single year).
Prefer the candidate that yields clean round multipliers across all years
with zero residual. Record the formula and its observed parameters in
`financial_context.json`, superseding the user's verbal estimates. See
[analysis-playbook.md](analysis-playbook.md) for the analysis side.

## 8. Freshness checker

A small script that reports what is missing or due soon. Cadence facts
(payday, statement lag days, export staleness threshold) are personal data:
keep them in a config file under `data/personal/`, not hardcoded, so
adjusting a payday is a data edit rather than a code change. Check:
continuity of monthly series, due dates for the last completed period,
staleness of range exports, and pending confirmations read from the context
file. Print `[missing] / [upcoming] / [note]` lines; exit 0 always (it
reports, it does not block).

Wire it into session start. For Claude Code, `.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python scripts/check_data_freshness.py --hook",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

and in `--hook` mode the script prints one JSON object to stdout - the
`systemMessage` is shown to the user, the `additionalContext` is injected
into the model's context:

```json
{
  "systemMessage": "[upcoming] ACME payslip 2031-01 expected by 2031-02-05",
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Freshness report... surface these items to the user and offer to import."
  }
}
```

For other agent frameworks, an equivalent instruction in the vault's agent
instructions file ("run the checker at session start and report") covers
the same ground.

**Prove the hook fires before trusting it.** Run the hook's exact command by
hand once after wiring it up. A hook that fails silently is worse than no
hook: the freshness report simply never appears, and nobody notices the
absence. The classic trap on Windows is the Microsoft Store `python` stub -
a zero-byte alias that exits with an error and no output, so the hook dies
every session while the real interpreter (`py`, or a fully-pathed python)
sits unused. If the interpreter situation is uncertain, verify with
`python --version` / `py --version` and write the working one into the hook.
