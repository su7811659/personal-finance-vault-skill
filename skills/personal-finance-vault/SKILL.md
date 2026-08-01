---
name: personal-finance-vault
description: Guide the user through building and maintaining a private, machine-readable personal-finance repository - archiving source documents (brokerage statements, payslips, bank and credit card exports, loan records), building deterministic parsers with arithmetic reconciliation, maintaining a personal financial context file, and running liability-aware performance, income, spending, and financial-independence analysis. Use when a user wants to organize their personal finances with an agent, import financial documents (對帳單, 薪資單, statements, payslips), or set up a personal finance repo.
---

# Personal Finance Vault

You are helping the user build a **private data vault** for their personal
finances that any agent session can work with: original documents archived
once, parsed deterministically into normalized CSV, validated arithmetically,
and analyzed with honest handling of unknowns.

## Non-negotiable principles

Apply these in every phase. They are the difference between a data vault and
a pile of files.

1. **Source documents are canonical.** Everything else must be reproducible
   from them by rerunning a script. Never hand-edit generated files.
2. **Reconcile before you trust.** Every parser must prove its own output:
   line items sum to stated subtotals, running balances chain correctly,
   totals match across independent sources. A parse without a passing check
   is a draft, not data.
3. **Never infer unknowns.** Missing facts are recorded as null plus an entry
   in an explicit unknowns list - never guessed. When the user later provides
   the fact, replace the estimate and note what superseded it.
4. **Deposits are not gains.** External cash flows (wires into a brokerage,
   loan disbursements) must never be counted as investment performance. Use
   cash-flow-aware return math (Modified Dietz, XIRR).
5. **Credentials never enter version control.** Document passwords live in an
   untracked file or environment variable, resolved at runtime. Verify with
   `git check-ignore` before the first commit.
6. **The repository must be private.** Confirm this before the first push.
   Never copy its contents into public repositories, issues, or pastebins.
7. **Leave room for new sources.** Every document type gets its own
   `Source/<type>/<institution>/` directory, its own parser, and its own
   normalized output files. Never overload an existing schema because a new
   source "almost fits".

## Phase 1 - Bootstrap the repository

1. Create a git repository (confirm it is private if remote) with the layout
   in [references/repo-layout.md](references/repo-layout.md).
2. Write the repository's agent instructions from
   [references/AGENTS-template.md](references/AGENTS-template.md), filling in
   the user's actual source types.
3. Create `.gitignore` covering: password files, raw extracted text dumps,
   OS/editor debris.
4. Create `data/personal/financial_context.json` from
   [references/financial_context.template.json](references/financial_context.template.json).
   Interview the user briefly (goals, liabilities, income shape) and record
   what they say with `"source": "user_reported_in_conversation"` - estimates
   are fine, invention is not.

## Phase 2 - Source intake

Ask which sources exist: brokerage statements, payslips, bank account
exports, credit card statements, loan contracts/screenshots, pension
records, e-invoice exports. For each one the user can provide:

1. Create `Source/<type>/<institution>/`.
2. Normalize file names to their coverage period: `YYYY-MM-DD.pdf` for
   point-in-time statements, `YYYY-MM.<ext>` for monthly documents,
   `<start>_<end>.<ext>` for range exports. Convert local calendars (e.g.
   ROC years: add 1911) in file names; keep originals' content untouched.
3. Detect and delete only **hash-identical** duplicates (`sha256`), with the
   user's confirmation.
4. Check continuity: list missing months and ask the user whether they are
   real gaps or expected (paper-only era, account opened later). Record the
   answer in the source directory's README so no future session re-asks.
5. If files are password-protected, set up the untracked password file now
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
4. Run the quality checks. Investigate every mismatch before declaring the
   dataset clean. Unrecognized labels go to the quality file as warnings -
   never silently dropped, never silently guessed.
5. Personal knowledge (which account is rent, which transfer is the side-gig
   payout) belongs in a user-maintained rules file under `data/personal/`, applied by
   the parser before generic rules - so extending it requires no code change.

## Phase 4 - Analysis

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
- **Financial-independence projection.** Scenario table (return rates x
  spending levels), the *required* return to hit each target, and a stress
  test replaying the user's worst historical drawdown. Solve for what must
  be true, not just what might happen.

Label every number as observed, derived, or assumed. List the assumptions
that most change the conclusion.

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
