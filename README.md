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
- **Drop-and-route intake**: put new raw files directly in `Source/`; the
  agent identifies them from their contents, moves them into the correct
  canonical directory, and leaves ambiguous files untouched for review.
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

Clone this public repository somewhere separate from every private finance
vault. The recommended installation is a directory link from the agent's
skill directory to `skills/personal-finance-vault/` inside the clone. Then a
normal `git pull --ff-only` updates the installed skill without another copy.

For Codex on Windows PowerShell, run from the cloned repository:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills"
New-Item -ItemType Junction `
  -Path "$env:USERPROFILE\.codex\skills\personal-finance-vault" `
  -Target (Resolve-Path ".\skills\personal-finance-vault")
```

For Unix-like systems:

```sh
mkdir -p ~/.codex/skills
ln -s "$(pwd)/skills/personal-finance-vault" \
  ~/.codex/skills/personal-finance-vault
```

If the destination already contains an older copied installation, move it
aside first and create the link only after confirming the target path. Other
supported installation locations include:

- Claude Code (project): `.claude/skills/personal-finance-vault/`
- Claude Code (user-wide): `~/.claude/skills/personal-finance-vault/`
- Other agent frameworks: wherever your framework loads skills from; the
  skill is a plain directory of markdown with YAML frontmatter.

Copying the whole `skills/personal-finance-vault/` directory is still valid
when links are undesirable, but copied installations require another full
copy after every update.

Then ask your agent something like *"help me organize my personal finances
into a repo"* - or invoke it directly with `/personal-finance-vault`.
During setup, the agent offers repository names based on your preferred name
or nickname, such as `jo-personal-finance` (recommended),
`jo-finance-vault`, or a lower-identity initials variant, and confirms the
remote is private before any push.

Shortcut: just clone this repo and start your agent inside it - the root
`AGENTS.md`/`CLAUDE.md` (read by Claude Code, Codex, and other agent CLIs)
tells the agent to walk you through installation, or to simply follow the
skill directly in a fresh private directory. No framework machinery needed.

## Update an existing installation

Update the public skill clone, not the private finance vault:

```powershell
git -C C:\path\to\personal-finance-vault-skill pull --ff-only
```

With the recommended junction/symlink installation, the update is now live;
start a new agent session so skill metadata and instructions reload. With a
copied installation, copy the complete `skills/personal-finance-vault/`
directory over the installed copy again and remove files that no longer
exist upstream. Local changes in the public skill clone can block a
fast-forward pull; review or commit them rather than discarding them.

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
