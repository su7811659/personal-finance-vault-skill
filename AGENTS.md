# Agent onboarding

You are reading the **source repository of the personal-finance-vault
skill** - this is not a vault, and no financial data belongs here. This
repository is public. If the user cloned it and wants to get started, guide
them as follows.

## Never build the vault in this repository

The vault the skill produces contains unredacted financial documents and
must live in its own **separate, private** repository. First step is always:
have the user pick or create an empty directory outside this repo.

## Fastest start (works with any agent)

The skill is plain markdown - no framework machinery required. Read
[skills/personal-finance-vault/SKILL.md](skills/personal-finance-vault/SKILL.md)
and follow its five phases directly, starting in the user's chosen vault
directory. The files under
[skills/personal-finance-vault/references/](skills/personal-finance-vault/references/)
are the supporting playbooks; SKILL.md tells you when to read each.

## Durable install (skill triggers automatically in future sessions)

Prefer a directory junction or symlink from the framework's skill location
to the **whole** `skills/personal-finance-vault/` directory in this clone
(SKILL.md plus `references/` - relative links must keep working). Then future
updates require only `git pull --ff-only` in this public repository and a new
agent session. Copying the directory also works, but every pull must be
followed by another complete copy or the installed skill stays stale.

Common framework skill locations:

- **Claude Code**, user-wide: `~/.claude/skills/personal-finance-vault/`
  (then `/personal-finance-vault` works in any project)
- **Claude Code**, single project: `<project>/.claude/skills/personal-finance-vault/`
- **Codex, Grok, or other AGENTS.md-style CLIs**: if your framework has a
  skills/plugins directory, copy it there; otherwise add one line to your
  global instructions file (e.g. `~/.codex/AGENTS.md`) pointing at the
  SKILL.md path with a note to read and follow it when the user asks to
  organize personal finances.

Never pull this public skill repository into a user's private vault. Keep the
two repositories separate; update the public clone, then let the installed
link expose the new version.

After installing, the user starts a session in their vault directory and
says something like "help me organize my personal finances into a repo" -
or names the skill directly.

## Red line for this repo

Never copy real financial data, statements, or personal identifiers into
this repository - it is public. Examples and fixtures must stay fictional.
