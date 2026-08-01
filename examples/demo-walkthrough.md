# Demo walkthrough (fictional)

Everything below is invented for illustration: "Jo", employed at "ACME
Software", banking at "Nova Bank". No real person's data appears anywhere in
this repository.

## Session 1 - bootstrap and payslips

Jo: *"Help me organize my finances. I have payslip PDFs from ACME."*

The agent bootstraps a private vault repo (Phase 1), then:

1. Creates `Source/payslips/ACME/`, renames `pay_2029_03.pdf` style files to
   `2029-03.pdf`, finds two byte-identical duplicate downloads by SHA-256 and
   deletes them with Jo's confirmation.
2. Notices `2028-11` is missing; Jo confirms ACME only issued paper slips
   before December 2028 - recorded in the directory README as "not a gap".
3. `pdftotext` returns amounts but garbled labels (broken ToUnicode); the
   agent renders one page to PNG, learns the layout visually, then extracts
   cleanly with pdfium + NFKC normalization.
4. Writes `scripts/convert_payslips.py` producing `payroll.csv` +
   `payroll_items.csv` + `payroll_checks.csv`. All checks pass: items sum to
   subtotals, additions minus deductions equal net pay, for every document.
5. Reports: base salary 52,000 -> 55,000 -> 58,500, raised every April;
   year-end bonus consistently ~2 months of base.

## Session 2 - bank export and cross-checks

Jo drops in a Nova Bank transaction export (`.xlsx`, newest-first, with a
footer row "412 records").

1. Archived as `Source/bank/NovaBank/2030-01-01_2030-12-31.xlsx`.
2. `convert_bank.py` stores rows oldest-first; the balance walk chains with
   zero breaks across all 412 rows.
3. Cross-check: every monthly salary deposit equals the payslip net pay to
   the unit. The vault now validates itself.
4. A recurring 14,500 transfer on the 1st is unknown; Jo says it is rent.
   One row is added to `data/personal/bank_counterparty_rules.csv` - no code
   change - and the converter reruns.

## Session 3 - analysis and maintenance

1. Spending analysis separates baseline (~31,000/month) from one-offs
   (annual tax, a laptop). Annual spending stated as a range with the
   uncategorized share disclosed.
2. A financial-independence scenario table: at Jo's savings rate, the
   3.5%-withdrawal target needs a 4.1%/yr portfolio return; a replay of
   Jo's worst historical drawdown pushes the date out by three years.
   Assumptions listed; unknowns (target retirement spending) stay in the
   unknowns list until Jo defines them.
3. `check_data_freshness.py` + a SessionStart hook are added. The next
   session opens with: `[upcoming] ACME payslip 2031-01 expected by
   2031-02-05`.
