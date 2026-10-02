# advisor-dispatch: dispatch-and-supervise development mode (Claude Code skill)

Version: v1.1.0 · [繁體中文](README.zh-TW.md)

## What it does
The main session acts as the **advisor** (plans, splits tickets, reviews, merges, watches the
deploy only) and hands implementation to subagents, each in its own git worktree. The point is
the quality pipeline, not "run more agents":

- Every ticket has: goal, acceptance criteria, file ownership, test requirements, out of scope
- Tickets run in parallel only when file ownership does not overlap; dependent tickets run serially
- The implementer delivers an **acceptance evidence table**; the advisor passes the evidence
  gate, reads the diff personally and reruns the high-risk items
- High-risk tickets get an extra independent verifier (adversarial, judges only, never edits code)
- Periodic patrols (against "dispatched and forgotten"), a ledger (against compaction amnesia),
  full test run after each merge, remote verification after deploy

Not for: small single-thread edits (even one ticket is overkill).

## Contents
```
advisor-dispatch/
├── SKILL.md                    core flow (this is what gets loaded)
├── references/
│   ├── evidence-table.md       evidence table rules and the "structural dependency" exception
│   ├── verifier.md             independent verifier contract, model selection, convergence rules
│   └── ledger.md               progress ledger format and the three terminal states
└── templates/
    ├── ticket.md               ticket template
    └── evidence-table.md       acceptance evidence table template
```
`references/` and `templates/` are referenced from SKILL.md and read only when needed.

## Install (two options; A is recommended)

**A. Copy into a skills directory (recommended)**
```bash
# personal, all projects
mkdir -p ~/.claude/skills && cp -R advisor-dispatch ~/.claude/skills/
# or for one project only
mkdir -p <project>/.claude/skills && cp -R advisor-dispatch <project>/.claude/skills/
```
Why: a skill is just a folder with a SKILL.md. Copying is enough: no manifest, no versioned
release, no marketplace, and you can edit it directly to match your conventions (commit style,
test commands).

**B. Package as a plugin**
Worth it when you distribute to many people and want versioned updates; it needs a plugin
manifest and a marketplace for distribution. This repo ships **no** manifest and the plugin
install flow was **not tested here**; for a few recipients, A costs less.

After installing, restart the Claude Code session; typing `/` should list `advisor-dispatch`
(or just say "dispatch this work" and let the description trigger it).

## Prerequisites
- Required: git, a working Agent tool (able to set `isolation: "worktree"` and `model`), Bash
- Use-if-present, fallback otherwise (comparison table at the top of SKILL.md): `SendMessage`,
  `ListAgents`, a scheduled wake-up tool, an external model CLI as verifier
- The project needs runnable test / typecheck commands, otherwise most evidence rows end up UNVERIFIED

## Example
```
Use dispatch mode to build: an order-export API (CSV) and a matching front-end button.
Split tickets first and show me the file-ownership partition; I'll approve before you dispatch.
Use sonnet for implementation, and add a verifier to the security-related ticket.
```
Expected behavior: the advisor writes a plan and a ticket table -> you confirm -> it records the
base SHA -> dispatches tickets in one message -> patrols -> collects evidence tables, reads
diffs and re-verifies per ticket -> merges in order and runs the full suite after each -> asks
whether it may push.

## Limitations
- SKILL.md, references and templates are English; a Traditional Chinese README is in README.zh-TW.md
- The process is deliberately heavy; for small tasks just do the work directly
- Whether the fallback paths (no SendMessage / ListAgents / scheduling) work on every Claude
  Code version is **unverified**
- It never pushes without your explicit consent; deployment is not gated by this skill and follows your project's own deploy flow
- No hooks or scripts enforce anything; the rules rely on the advisor's discipline. The red-line
  list is a reminder, not a mechanism
- A fresh-environment install-and-run test has **not** been done yet

## License
MIT. See `LICENSE`.
