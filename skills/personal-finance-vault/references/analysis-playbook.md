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

## 4. Multi-currency

- Pick one base currency for net worth and projections; state it.
- Use the user's *actual* conversion rates where documents show them (a wire
  of X foreign units costing Y local units defines the rate that mattered);
  otherwise use a stated, dated market rate and list it as an assumption.
- Never mix currencies in one aggregate silently. FX movement is a return
  component of its own - if it is material, show it separately.

## 5. Financial-independence projection

Structure it as a table plus an inversion, not a single forecast:

1. **Inputs**: current investable net worth (assets minus liabilities),
   savings capacity per year (net income minus spending scenarios), horizon
   (target date), spending scenarios (lean / base / comfortable).
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

## 6. Cross-dataset checks worth automating

Whenever two domains overlap, add the comparison to the quality layer
rather than doing it once by hand: payslip net vs bank salary credit; card
statement total vs bank card-payment debit; loan disbursement vs principal
minus fees; brokerage wire vs bank outgoing transfer. Track agreement
month by month; a new mismatch is the earliest smoke detector the vault has.
