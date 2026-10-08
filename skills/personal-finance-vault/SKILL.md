---
name: personal-finance-vault
description: Guide the user through building and maintaining a private, machine-readable personal-finance repository - archiving source documents (brokerage statements, payslips, bank and credit card exports, loan records), building deterministic parsers with arithmetic reconciliation, maintaining a personal financial context file, and running liability-aware performance, income, spending, balance-sheet/net-worth, and financial-independence (FIRE) analysis. Use when a user wants to organize their personal finances with an agent, import financial documents (對帳單, 薪資單, statements, payslips), build a net-worth statement (資產負債表), plan financial independence, or set up a personal finance repo.
---

# Personal Finance Vault

You are helping the user build a **private data vault** for their personal
finances that any agent session can work with: original documents archived
once, parsed deterministically into normalized CSV, validated arithmetically,
and analyzed with honest handling of unknowns.

## Non-negotiable principles

Apply these in every phase. They are the difference between a data vault and
a pile of files.

1. **Source documents are canonical.** Generated data must be reproducible
   from them plus explicit user-owned context and reviewed inputs by rerunning
   a script. Never hand-edit generated files.
2. **Reconcile before you trust.** Every parser must prove its own output:
   line items sum to stated subtotals, running balances chain correctly,
   totals match across independent sources. A parse without a passing check
   is a draft, not data.
3. **Never infer unknowns.** Missing facts are recorded as null plus an entry
   in an explicit unknowns list - never guessed. When the user later provides
   the fact, replace the estimate and note what superseded it.
   The same discipline applies to **decisions**: when the user rules
   something out ("I will not surrender this policy"), record it in the
   context file's `user_decisions` with its scope and what remains open -
   otherwise every future session re-proposes the rejected option.
4. **Deposits are not gains, and plans are not events.** External cash flows
   (wires into a brokerage, loan disbursements) must never be counted as
   investment performance - use cash-flow-aware return math (Modified Dietz,
   XIRR). A planned repayment or expected wire is never treated as a
   completed transaction until a statement confirms it.
5. **Credentials never enter version control.** Document passwords live in an
   untracked file or environment variable, resolved at runtime. Verify with
   `git check-ignore` before the first commit.
6. **The repository must be private, and even inside it, minimize.** Confirm
   privacy before the first push; never copy vault contents into public
   repositories, issues, or pastebins. Within the vault, the machine-readable
   layers keep only what analysis needs - no government IDs, no full account
   numbers, no identity documents. And be transparent about the data flow
   this workflow implies: the documents are read by the user's agent/model
   provider - say so in Phase 1 and let the user decide.
7. **Leave room for new sources.** Every document type gets its own
   `Source/<type>/<institution>/` directory, its own parser, and its own
   normalized output files. Never overload an existing schema because a new
   source "almost fits".

## Phase 1 - Bootstrap the repository

1. Ask what preferred name, nickname, or initials the user wants in the
   repository name. Offer 2-4 concrete choices rather than an empty naming
   question: `<name>-personal-finance` (recommended),
   `<name>-finance-vault`, `personal-finance-<name>`, and a lower-identity
   option such as `<initials>-finance-vault`. Never derive a repository name
   from a government ID, email address, or machine username without the
   user's confirmation. Also ask where the separate vault directory should
   live.
2. Create the git repository (confirm it is private if remote) with the
   layout in [references/repo-layout.md](references/repo-layout.md). The
   vault must not be created inside this public skill repository. Two choices
   to put to the user explicitly: (a) whether generated CSVs are committed
   (convenient) or gitignored (less exposure if the repo ever leaks - they
   are reproducible either way); (b) that this workflow means their
   financial documents are read by their agent/model provider - confirm
   they accept that data flow. Also remind them `Source/` is irreplaceable
   (banks purge download history) and deserves an encrypted backup.
3. Write the repository's agent instructions from
   [references/AGENTS-template.md](references/AGENTS-template.md), filling in
   the user's actual source types.
4. Create `.gitignore` covering: password files, raw extracted text dumps,
   OS/editor debris.
5. Create `data/personal/financial_context.json` from
   [references/financial_context.template.json](references/financial_context.template.json).
   Interview the user briefly (goals, liabilities, income shape) and record
   what they say with `"source": "user_reported_in_conversation"` - estimates
   are fine, invention is not.

## Phase 2 - Source intake

Ask which sources exist: brokerage statements, payslips, bank account
exports, credit card statements, loan contracts/screenshots, pension
records, insurance documents, shared-expense exports, and e-invoice exports.
For insurance intake and interpretation, read
[references/insurance-playbook.md](references/insurance-playbook.md).
Tell the user to drop new raw documents directly
into the top level of `Source/`; do not require an inbox directory or require
the user to know the final taxonomy. At session start and whenever the user
says files were added, scan only files directly under `Source/` and route
them as follows:

1. Inspect content, not just the supplied filename, to identify the document
   type, institution, account context, and coverage date. If encrypted, use
   an environment variable or untracked password file; never echo or track
   the password. If classification is ambiguous, leave the file at the
   `Source/` top level and ask the user instead of guessing.
2. Create the corresponding `Source/<type>/<institution>/` directory. Each
   domain parser must scan only its canonical subdirectory, never the
   `Source/` top level, so an unclassified document cannot be imported by
   accident.
3. Normalize file names to their coverage period: `YYYY-MM-DD.pdf` for
   point-in-time statements, `YYYY-MM.<ext>` for monthly documents,
   `<start>_<end>.<ext>` for range exports. Convert local calendars (e.g.
   ROC years: add 1911) in file names; keep originals' content untouched.
   Record an original-name -> new-name mapping in the directory README -
   original filenames often carry traceability hints you may want later.
4. Before moving, reject a destination collision unless the files are
   hash-identical. Detect and delete only **hash-identical** duplicates
   (`sha256`), with the
   user's confirmation.
5. Pass recognized documents to the source type's transactional importer,
   which stages and validates before publishing the unchanged original and
   generated outputs. Review its summary and quality checks. If no parser
   exists yet, archive the original and explicitly mark it unparsed; archival
   alone does not establish analysis readiness.
6. Check continuity: list missing months and ask the user whether they are
   real gaps or expected (paper-only era, account opened later). Record the
   answer in the source directory's README so no future session re-asks.
7. If files are password-protected, set up the untracked password file now
   and record *where the password lives* (not the password) in the README.

## Phase 3 - Build parsers

Follow [references/parser-playbook.md](references/parser-playbook.md) for
each source type. Summary of the loop:

1. Inspect a few documents spanning the date range (layouts drift).
2. Extract programmatically; when text extraction fails or labels are
   garbled, render pages to images and read them visually to learn the
   layout, then find a library that extracts it deterministically.
3. Write one deterministic converter script per source type that rebuilds
   `data/normalized/<domain>.csv` from scratch on every run (idempotent, no
   appending), plus `data/quality/<domain>_checks.csv`.
4. For the recurring "new statement arrived" flow, make imports
   **transactional**: reject date/hash conflicts, rebuild into a staging
   area, validate there, and only replace the canonical dataset when every
   check passes - a failed import must leave the vault untouched. Provide a
   validate-only dry run.
5. Run the quality checks. Investigate every mismatch before declaring the
   dataset clean. Unrecognized labels go to the quality file as warnings -
   never silently dropped, never silently guessed. Document what the parser
   does not yet recognize.
6. Personal knowledge (which account is rent, which transfer is the side-gig
   payout) belongs in a user-maintained rules file under `data/personal/`, applied by
   the parser before generic rules - so extending it requires no code change.
7. Track source coverage separately from arithmetic correctness: identify
   archived-but-unparsed documents and missing pages/sections. When a visual
   fact cannot be extracted reliably, use the source-bound reviewed-record
   workflow in the parser playbook, not an unexplained hardcoded value.

## Phase 4 - Analysis

Follow [references/analysis-playbook.md](references/analysis-playbook.md).
Only analyze data that passed its checks. The high-value analyses, in the
order they usually become possible:

- **Cross-dataset reconciliation.** Salary on payslips vs salary deposits in
  the bank export; card statement total vs the card payment in the bank
  export; loan disbursement vs principal minus fees. Every match increases
  trust; every mismatch is a finding.
- **Portfolio performance**, external-cash-flow aware: Modified Dietz per
  month, XIRR with exact wire dates, drawdown and volatility, benchmark
  comparison using the same deposit dates.
- **Income structure.** Reverse-engineer the employer's actual formulas from
  payslip history (raise cycle and rates, bonus = multiplier x base). Present
  empirical numbers; update the context file, superseding user guesses.
- **Spending structure.** Classify bank/card flows; separate recurring
  baseline from one-offs; state the annual spending range honestly.
- **The query layer.** Once a second source covers the same money as the
  first, encode the don't-count-it-twice rules as SQLite views over a
  mirror of the CSVs, instead of re-deriving them by hand every session -
  hand-derived dedup eventually goes wrong silently.
- **Balance sheet.** A monthly net-worth series with assets tiered by
  accessibility (liquid / invested / age-gated / unvalued-but-listed) and
  liabilities at amortized balance. This is where the FI projection's
  inputs come from; it is gated on its own inputs (dated FX, statement
  valuations), never padded with invented rates.
- **Financial-independence projection.** Scenario table (return rates x
  spending levels), the *required* return to hit each target, and a stress
  test replaying the user's worst historical drawdown. Solve for what must
  be true, not just what might happen. The blocking input is the user's
  target annual spending - ask for it, record it in the context file, and
  once a plan is chosen, write it down with a review cadence instead of
  re-deriving it every session.

Label every number as observed, derived, or assumed. List the assumptions
that most change the conclusion.

### Optional HTML dashboard

When validated data can support a useful overview, offer a local HTML
dashboard and let the user choose whether to generate it. Read
[references/dashboard-playbook.md](references/dashboard-playbook.md) to
assess readiness and, if accepted, build and verify it. Enough data means
enough evidence for the proposed panels, not an arbitrary row count or a
requirement to have every financial domain. Do not generate automatically,
repeat a declined offer every session, or treat approval as permission to
publish, host, or upload private data.

## Phase 5 - Maintenance

1. Add a **freshness checker** script that knows each source's cadence and
   due dates (statement N days after month end, payslip on payday, export
   staleness threshold) and prints what is missing or due soon, plus pending
   confirmations read from the context file.
2. Wire it to run at session start (for Claude Code: a `SessionStart` hook in
   `.claude/settings.json`; for other agents: an instruction in the agent
   instructions file), so any future session greets the user with what to
   fetch - e.g. "July statement due in 4 days; payslip expected the 5th."
3. When new documents arrive: archive, rerun the converter, review checks,
   rerun affected analyses. Commit only when the user asks.
4. As operations become routine, **encapsulate them as vault-local skills**
   (e.g. `.claude/skills/import-<broker>-statement/` or
   `.agents/skills/...` inside the vault repo): a short SOP that names the
   importer script, the checks to review, and the red lines. The vault then
   carries its own operating manual, and any future session follows the
   same procedure instead of improvising.
5. Keep build commands, expected coverage, output inventories, and quality
   policy in one machine-readable configuration once multiple import paths
   need them. Local skills reference that configuration rather than copying
   its rules. Follow the parser playbook's environment and scoped-rebuild
   guidance; optional integrations must not block unrelated local imports.

## Red lines

- Never commit or echo passwords, even "temporarily".
- Never push to a public remote; never paste vault contents into public
  places. If asked to build a public artifact *from* the vault (like this
  skill), rewrite everything from scratch with fictional data and sweep for
  real names, employers, account numbers, and amounts before publishing.
- Never present an unreconciled parse, an inferred unknown, or a
  deposit-inflated return as fact.
- This workflow organizes the user's own data. It is not investment, tax, or
  legal advice; say so when conclusions border on those domains.
