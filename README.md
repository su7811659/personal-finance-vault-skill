# Personal Finance Vault - agent skill

An agent skill that guides you (and your coding agent) through building a
**private, machine-readable personal-finance repository**: original
documents archived once, parsed deterministically into normalized CSV,
validated arithmetically, and analyzed with honest handling of unknowns.

Born from a real vault built this way - brokerage statements, payslips, bank
and credit card exports, and a loan record, all reconciling against each
other to the unit - then rewritten from scratch with fictional examples so
the methodology could be shared.

## What the skill makes your agent do

- **Archive-first layout**: `Source/<type>/<institution>/` for every
  document type, named by coverage period, extensible without restructuring.
- **Deterministic parsers**: one converter per source type, full rebuild
  every run, traceability columns on every row.
- **Mandatory reconciliation**: payslip items must sum to their subtotals,
  bank balances must chain row-by-row, card statements must add up - and
  independent sources must agree (payslip net == salary deposit).
- **Honest unknowns**: facts the documents cannot prove are null and listed,
  never guessed; user-stated estimates are superseded when documents arrive.
- **Liability-aware analysis**: cash-flow-aware returns (deposits are not
  gains), empirical income formulas, spending baselines vs one-offs,
  financial-independence scenarios solved for the *required* return.
- **Freshness checks**: every new session opens by telling you which
  statement, payslip, or export is missing or due.

## Install

Copy `skills/personal-finance-vault/` into your agent's skill directory:

- Claude Code (project): `.claude/skills/personal-finance-vault/`
- Claude Code (user-wide): `~/.claude/skills/personal-finance-vault/`
- Other agent frameworks: wherever your framework loads skills from; the
  skill is a plain directory of markdown with YAML frontmatter.

Then ask your agent something like *"help me organize my personal finances
into a repo"* - or invoke it directly with `/personal-finance-vault`.

Shortcut: just clone this repo and start your agent inside it - the root
`AGENTS.md`/`CLAUDE.md` (read by Claude Code, Codex, and other agent CLIs)
tells the agent to walk you through installation, or to simply follow the
skill directly in a fresh private directory. No framework machinery needed.

## Layout

```
skills/personal-finance-vault/
├── SKILL.md                     # the guided workflow (5 phases + red lines)
└── references/
    ├── repo-layout.md           # target directory structure and conventions
    ├── AGENTS-template.md       # agent rules for the generated vault repo
    ├── financial_context.template.json
    ├── parser-playbook.md       # extraction, staged imports, validation
    └── analysis-playbook.md     # returns math, income/spending, FI scenarios
examples/
└── demo-walkthrough.md          # fictional three-session example
```

## Privacy stance (read this)

The vault this skill builds contains unredacted financial documents. The
skill hard-codes the guardrails: the vault repo must stay private,
credentials never enter version control, and nothing from the vault is ever
copied into public places. This public repository contains **no real
financial data** - all examples are fictional.

## Disclaimer

This is a data-organization methodology, not investment, tax, or legal
advice.

## License

MIT
