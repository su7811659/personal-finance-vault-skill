# Analysis playbook

How to turn reconciled data into conclusions that survive scrutiny. Only
analyze data whose quality checks pass; label every number **observed**
(read from a document), **derived** (computed from observed), or **assumed**
(chosen by you/the user), and list the assumptions that most change the
conclusion.

## 1. Portfolio performance (external-cash-flow aware)

The one unbreakable rule: **money moved into the account is not profit.**

- **Classify cash flows first.** Every cash/journal/transfer row needs an
  `is_external` decision (external contribution/withdrawal vs internal
  movement). Rows still marked unknown are a *gate*: no return calculation
  until they are classified.
- **Modified Dietz, monthly.** Return = (EV − BV − F) / (BV + Σ w·f), where
  F is net external flow and each flow's weight w is the fraction of the
  month remaining after it lands. Use month-end statement valuations only;
  do not interpolate intra-month values you do not have. Link months
  geometrically for annual/cumulative figures.
- **XIRR with exact dates** for the money-weighted view. Solve with
  bisection on a bracketed rate rather than Newton alone (Newton diverges on
  sign-flip-heavy histories); if no bracket exists, report that XIRR is
  undefined for the period instead of forcing a number.
- **Benchmarks must eat the same cash flows.** Simulate buying the benchmark
  with the user's actual deposit amounts on the actual deposit dates, using
  adjusted (dividend-reinvested) prices. Comparing against a lump-sum
  benchmark start is flattery, not analysis.
- Report drawdown (peak-to-trough on month-end values), annualized monthly
  volatility, best/worst month - the risk numbers that later feed the
  stress test.

## 2. Income structure

Reverse-engineer the employer's actual mechanics from payslip history:

- Detect the raise cycle (which month the base changes) and the rate path.
  Watch for retroactive pay: a one-month spike equal to exactly one month's
  raise difference is back-pay, not a higher salary tier.
- Test candidate bonus formulas against **every** year. A formula that fits
  one year is a coincidence - averages and adjacent-month bases often
  coincide once. Prefer the candidate producing clean round multipliers
  across all years with zero residual.
- Distinguish the stable components (contractual, recurring) from volatile
  ones (performance multipliers); state each component's observed range.
  Planning should use the floor of the volatile parts, not the mean.

## 3. Spending structure

- Separate **recurring baseline** (rent, utilities, subscriptions - detected
  as same-counterparty monthly series, tolerating small amount drift) from
  **one-offs** (annual tax, memberships, travel spikes).
- Disclose the unclassified share explicitly. "Annual spending 550-700k, of
  which 140k in transfers is unclassified" is honest; a single confident
  number is not.
- Watch for double counting across sources: card purchases appear again as
  the card-payment debit in the bank account. Count the purchases, not the
  repayment (or vice versa), never both.
- Shared-expense apps are candidate context, not extra bank transactions.
  Keep gross payment, the user's share, advances for others, and repayments
  separate. A date/amount match alone is not confirmation: require additional
  independent evidence or explicit user confirmation, retain the matching
  rationale and status, and keep ambiguous candidates out of confirmed totals.
  A repayment settles a receivable; it is not automatically income or another
  purchase. An app record alone does not prove the debt was paid.
- Treat recurring exports as snapshots: use stable IDs and a documented
  snapshot-selection rule rather than summing overlapping exports. Preserve
  source-reported currency labels and units. Any reviewed correction belongs
  in a scoped override with evidence and an effective period; record actual
  versus estimated FX separately and never silently relabel the raw source.

## 4. Multi-currency

- Pick one base currency for net worth and projections; state it.
- Use the user's *actual* conversion rates where documents show them (a wire
  of X foreign units costing Y local units defines the rate that mattered);
  otherwise use a stated, dated market rate and list it as an assumption.
- Never mix currencies in one aggregate silently. FX movement is a return
  component of its own - if it is material, show it separately.

## 5. The query layer: encode the dedup rules once

When a second source arrives that covers the same money as the first, the
vault acquires *seams*: a bank export vs the mailed statement for the same
account, card purchases vs their settlement debits, receipts vs the card
lines they duplicate. The rules for not counting the same money twice can be
written down as advice - and advice gets re-derived by hand in every
analysis session, and eventually re-derived wrongly (a card-settlement sum
misread as a balance held at another bank; a spending total quietly missing
the era one source cannot see). Encode the rules **once, executably**:

- Mirror every normalized, derived, and quality CSV into a single SQLite
  file with one rebuild script (typed columns - SQLite compares TEXT above
  every number, so an amount stored as text silently breaks `amount < 0`;
  indexes on the join keys). The CSVs stay canonical; the mirror is
  disposable and rebuilt after every import.
- Write the seam rules as **views**, each with a comment stating the rule:
  - a *continuous account series* view unioning the primary source with the
    secondary, where a secondary row is admitted only when no primary row
    shares its (date, amount, resulting balance) - self-healing: eras and
    gaps the primary cannot see fill in automatically, with no hardcoded
    date windows to maintain;
  - a *unified purchases* view across card rails, with a flag on the bank
    side marking settlement rows so no query can innocently add both;
  - a *monthly summary* view with the dedup already applied - income in,
    card purchases from the card rails, cash out with settlements excluded,
    and the exact account delta as a self-check column.
- Provide a one-line **read-only** query runner (open the connection in
  query-only mode) so a mistyped statement cannot mutate the mirror.
- When two overlapping sources disagree about which is authoritative,
  decide **per column and per era**, and record the decision in the view's
  comment: one source may carry the only merchant text for the early era
  while the other alone carries categories. Neither source "wins" wholesale.

The payoff is not query convenience - it is that a dedup rule wrong in a
view gets fixed once, while a dedup rule wrong in a session's ad-hoc join
gets fixed every time it is rediscovered.

## 6. Balance sheet (net-worth statement)

The FI projection below consumes "current investable net worth" - this is
where that number is assembled honestly. Build it as a **monthly series** in
`data/derived/`, reproducible from the datasets plus the context file, one
row per (month, line item):

- **Tier assets by accessibility**, because they are not interchangeable:
  liquid cash (bank balances from statements); brokerage at month-end
  statement value; **age-gated or locked assets** (statutory pension
  accounts, lock-in-period holdings) - real net worth, but *zero* for any
  plan that needs the money before the gate lifts; **unvalued assets**
  (a surrender-value policy whose current value no document states) -
  listed by name with `null`, never guessed, never silently omitted.
- **Liabilities at amortized balance**, derived from the loan terms in the
  context file (rate, payment, schedule), not from memory. A planned early
  payoff is a plan, not a schedule change, until a statement shows it.
- **One base currency**, with every conversion rate dated and sourced
  (section 4). No FX series in the vault means no converted aggregate -
  gate the statement on its inputs the same way return math gates on
  cash-flow classification, rather than inventing a rate.
- Label every line **observed / derived / assumed**. The honest headline is
  tiered ("liquid X, invested Y, gated Z, unvalued: policy A"), not a single
  proud total with unstated holes in it.

## 7. Financial-independence projection

Structure it as a table plus an inversion, not a single forecast:

1. **Inputs**: investable net worth from the balance sheet above (liquid +
   invested tiers only - gated and unvalued assets are excluded by
   construction), savings capacity per year (net income minus spending
   scenarios), horizon (target date), spending scenarios (lean / base /
   comfortable).
2. **Scenario table**: rows = flat annual return rates (e.g. 0/4/7/10%),
   columns = spending levels; cells = projected net assets at the target
   date. Model liabilities explicitly (scheduled payments; planned early
   payoff as a dated cash outflow).
3. **Targets**: annual spending / SWR. Use a withdrawal rate honest for the
   horizon (a 40-year-old retiree is 3.25-3.5%, not 4%).
4. **The inversion is the headline**: solve for the *required* return per
   spending level (bisection over the projection). "You need +1.5%/yr" beats
   pages of scenarios.
5. **Stress test**: replay the user's own worst historical drawdown at the
   worst plausible time and report the landing zone. If history is short,
   borrow a major index drawdown and say so.
6. Planned events (a repayment, an expected bonus, a pending wire) enter the
   projection as *plans* - dated, reversible, marked unconfirmed - and are
   promoted to facts only when a statement shows them. **A planned event is
   never a completed cash flow.**
7. **The blocking input is the user's target annual spending.** Without it,
   every FI answer is a four-column scenario table forever; with it, the
   inversion collapses to one number. Ask for it explicitly (a range is
   fine), record it in the context file's goals block, and stop re-running
   the full grid once it exists.
8. **The plan is an artifact, not a conversation.** Once the user picks a
   spending level and target, write the chosen plan into the context file's
   goals (target date, target spending, the required-return headline as of
   that date) and give the freshness checker a review cadence (e.g.
   quarterly), so future sessions track progress against a recorded plan
   instead of re-deriving one. Do not let the machinery tempt you into a
   single "FIRE date" headline - the honest output stays the inversion plus
   its assumptions, refreshed as data arrives.

## 8. Cross-dataset checks worth automating

Whenever two domains overlap, add the comparison to the quality layer
rather than doing it once by hand: payslip net vs bank salary credit; card
statement total vs bank card-payment debit; loan disbursement vs principal
minus fees; brokerage wire vs bank outgoing transfer. Track agreement
month by month; a new mismatch is the earliest smoke detector the vault has.
