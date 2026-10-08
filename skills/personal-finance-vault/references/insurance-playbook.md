# Insurance documents

Read for insurance intake, parsing, or coverage interpretation. These are
source-handling rules, not recommendations to buy, cancel, or replace cover.

## Distinguish the evidence

Keep association registers, broker review reports, policy contracts, premium
notices/payment records, and account/surrender-value statements distinct.
A broker summary is evidence of what the report says, not a substitute for
the contract or proof of current payment, in-force status, or account value.
Record the document date and perspective (insured, policyholder, payer);
do not assume these roles refer to the same person.

Use dedicated source types and normalized tables as needed. A short snapshot
may need only a cited context entry; a detailed report warrants policy/rider,
benefit, and condition records. Keep reusable parsing logic separate from
source-specific reviewed facts. Use pseudonymous policy keys in analytical
outputs where possible; identifiers retained for matching must stay text so
leading zeros survive, and should not appear in dashboard payloads.

## Preserve meaning, not just totals

- Separate original premium, scheduled/due premium, and actual payment.
  Paid-up status, discounts, payment frequency, currency, payer, and blank
  cells can change the interpretation. Never turn a report's premium total
  directly into current annual spending without payment evidence.
- Coverage amounts are not cash assets. Only a dated value statement can
  support surrender/account value; absent evidence stays null. Record policy
  loans separately when documented, avoiding double deductions from a value
  already reported net of that loan.
- Benefits are conditional: retain units (per day, event, year, or lifetime),
  ranges, exclusions, limits, and alternatives. Do not sum mutually exclusive
  benefits or add a summary total to the detail it summarizes.
- The same policy in a register and a broker report is not two policies.
  Link observations with evidence and keep disputed matches unresolved;
  retain each source's dated assertions instead of overwriting disagreement.

## Validate and expose limits

Account for every page/section before claiming complete extraction. Check
policy/rider counts, benefit-to-contract references, printed subtotals where
applicable, and cross-page continuations. Visual-only diagrams use the parser
playbook's hash-bound review records. Count checks supplement, not replace,
checks of values, units, and conditions.

Distinguish extraction completeness from financial verification. A fully
extracted report can still lack current premiums, payer, beneficiaries,
contract terms, or asset values. Keep these in a dated pending-review list
and request the specific supporting document when needed.

Fictional example: a review lists an original annual premium of 18,000 and
leaves the current due field blank. Store 18,000 as the reported original
premium and current due as null; do not record 18,000 as this year's paid
expense. A death benefit of 900,000 is coverage, not 900,000 of net worth.
