---
name: advisor-dispatch
description: Advisor/dispatch development mode. The main session only plans, splits work into tickets, reviews, merges and watches deploys; implementation is done by subagents, each in its own git worktree. Use when asked to dispatch work, write tickets, run parallel development, use worktrees, "advisor mode", or delegate to subagents. Not for small single-thread edits.
---

# Advisor Dispatch — dispatch-and-supervise development mode

The main session is the **advisor**: it plans, splits tickets, reviews, merges and watches
deployment. It does **not write implementation code**. All implementation is done by
subagents, each in its own git worktree.

Why this split: the advisor's context is reserved for cross-ticket coordination and quality
judgment. If the advisor starts editing code, its context fills with implementation detail
and it competes with the worktree agents for the same files.

## Prerequisites and fallbacks (read this table first)

The core flow needs only: git, the Agent tool (able to set `isolation: "worktree"` and
`model`), and Bash. Everything else is "use it if present, otherwise use the fallback" —
the concept is never dropped:

| Capability | If present | If absent (fallback) |
|---|---|---|
| `SendMessage` (continue the same subagent) | Send review findings and nudges to the original agent | Only after the original agent has finished may you dispatch a new agent into the **same worktree** (how: see "Recovery / follow-up agent in an existing worktree" in Step 2), with the previous findings and current state in the prompt. If the original agent may still be alive, do not dispatch |
| `ListAgents` (is the agent still alive) | Use it for patrols and before any re-dispatch | Infer from commit / uncommitted-change activity in the worktree plus Agent completion notices. If you cannot confirm it has finished, treat it as alive |
| Scheduled wake-up (background sleep, cron-style tool) | Wake up every ~10 minutes to patrol | Patrol whenever any message arrives, and tell the user "there is no automatic wake-up; I only patrol when you interact" |
| External model CLI as verifier (a coding agent from a different model family) | Strongest independence, see `references/verifier.md` | Use a fresh subagent at the advisor's level as verifier |
| Content gate (mechanically judging whether evidence supports an acceptance item) | One request judges every row | The advisor reads row by row: does the evidence actually speak to that acceptance item? |
| Existing plan / spec | Split tickets from it directly | Write a plan first, then split |

Unverified: whether every fallback works on every Claude Code version has not been checked
by the author. Use whatever tools are actually available.

## Model selection (parameterized, never hard-coded)

- **Advisor** = the current main-session model. No configuration.
- **Implementer** = the Agent tool's `model` parameter. Default to a mid-tier model (for
  example `"sonnet"`). If the user says "give this ticket to opus", change that ticket (or
  all of them). Always set `model` **explicitly** on every dispatch; omitting it inherits the
  main session's model, usually the most expensive one.
- **Verifier** (high-risk tickets only): independent of the implementer, level ≥ the
  implementer. Selection order is in `references/verifier.md`.

### Choosing the implementer's level for each ticket

Decide per ticket, at split time, and write the choice into the ledger's `dispatched` line.
Think in three levels, not model names; map each level to whatever models are available now.

| Ticket | Level |
|---|---|
| The user named a model for it | that model for the implementer, no level judgment (the high-risk row still adds a verifier) |
| Low risk and mechanical: copy changes, config, a single-file edit with an obvious fix | lighter than the default |
| Ordinary implementation | the default (mid-tier) |
| High risk: security / permissions / money, concurrency, data migration, multi-file core logic (same definition as in Step 3) | stronger than the default, **and** an independent verifier |
| The spec is vague, or the ticket needs design decisions rather than execution | stronger than the default, or split / clarify the ticket first |

Rules around the table:
- **Cost is not a reason to drop a level on a high-risk ticket.** A cheaper model that needs two
  send-backs usually costs more than a stronger one that passes first time.
- **Escalate on evidence, not on feeling:** `BLOCKED` because the task is too hard (Step 2), or the
  same finding sent back twice and still not fixed (Step 4; the alternative there is to stop and ask
  the user). Escalating means re-dispatching with a stronger level and a changed prompt, never the
  same prompt again.
- **If the ticket is already on the strongest available level**, escalation is not possible and
  re-dispatching the same thing is not allowed. Take the matching alternative from Steps 2 and 4
  instead: supply the missing context (`BLOCKED` for lack of context), split the ticket (too big),
  or stop and ask the user (repeated findings, or the task is simply too hard).
- **Never go below the level the user asked for**, and never pick the verifier's level lower than
  the implementer's.
- The table is a starting point, not a measured threshold. If the project has its own
  convention (for example "all migrations go to the strongest model"), follow that instead.

## Flow overview

```
0. Multi-step flow -> write a flow contract first (Step 1.5); no seam without an owner may start
1. Split tickets (including file-ownership partitioning)
2. Record base -> dispatch (one subagent + own worktree + own branch per ticket)
3. Review each ticket (evidence gate -> advisor reads the diff personally)
4. Not passed -> send back to the same agent -> re-review
5. All passed -> advisor merges in order
6. Watch the deploy -> done only after remote verification (frozen during verification)
```

## Step 1: Split tickets

With no plan, write one first. With a plan, split it into tickets. Every ticket must contain
(template in `templates/ticket.md`): **goal** (one sentence), **acceptance criteria**
(verifiable, not "done well"), **file-ownership list**, **test requirements**, and
**out of scope**.

### File-ownership partitioning (the precondition for parallelism)

A worktree only stops working trees from stepping on each other; it does **not** prevent
merge conflicts.
- Within one parallel batch, a file may belong to **only one** ticket.
- Two tickets that must touch the same file (shared route registration, schema) run in
  series, or the shared edit is pulled out into a prerequisite ticket done first.
- After splitting, compare the ownership lists ticket by ticket; dispatch in parallel only
  when there is no overlap.

### Parallel vs serial: scheduling only, not two processes

When tickets depend on each other, crowd the same files, or requirements are being decided
as you go, run **serially**: one ticket at a time, DONE -> review -> pass, then dispatch the
next. Each ticket still gets the full contract (worktree, report format, evidence gate,
review).

**Forbidden** reasoning: "can't parallelize -> the advisor does it itself -> skip review".
The only valid reason not to use this skill is that the task is too small (even one ticket is
overkill), not that it "can't be parallelized".

## Step 1.5: Flow contract (required for multi-step flows)

Trigger: a ticket involves two or more steps, or the user wants "a whole flow to work", not
"one thing fixed". **Before** splitting, write a three-column table:

| Step | Entry (who, from where, with what state/params) | Exit (destination of every exit, including back / cancel / failure) |

- **Every seam is assigned to a named ticket** (jump target, return path, context params,
  state carry-over, where errors go back to). A seam with no owner may not start. Typical
  failure: each ticket passes its own acceptance, yet the combined flow is broken.
- A ticket may **not** dismiss a seam with "out of scope for this ticket" unless the contract
  names which ticket takes it.
- Cross-check the contract against the spec / design / requirements once; list gaps
  immediately.

## Step 1.6: Structural obstacles go to the user

If implementation reveals that you must touch an existing shared layer, a live feature, a
hard-coded single-scenario assumption, or someone else's module: **stop and put the
trade-off in front of the user** — work around it (compat layer / transition layer / narrower
ticket) vs. restructure properly, with cost, risk and a recommendation. After the user
decides, record `decision: <topic> -> <choice> (date)` in the ledger and rewrite the
affected tickets before dispatching further. **Never** pick the workaround yourself and keep
dispatching — worry about breaking a live feature is the user's risk decision, not something
the advisor may absorb.

## Step 2: Dispatch

### Pre-dispatch snapshot

Before calling Agent, record the **full** base of the target integration branch:
`git -C <repo> rev-parse HEAD`.

- The ledger records `repo + target branch + full SHA` per ticket (short SHAs are for display only).
- A serial ticket takes a **fresh** base after the previous one is merged and the
  integration tests pass.
- Put the expected base in the ticket prompt; the implementer must check it before starting
  and stop and report if it differs.
- Step 3 always diffs against the full base in the ledger; never back-fill it later.
- **Every dispatch has its own number** `ticket-N#aK` (K increments; a re-dispatch, a model
  change or a recovery each counts as a new one). Put it in the ticket prompt and require
  the report to echo it back unchanged.
- Before calling, append `ticket-N#aK dispatching` to the ledger; append `dispatched` once
  it succeeds. If the call errors or times out, **do not re-dispatch directly**: first
  determine whether it actually started (`git worktree list`, plus `ListAgents` if you have
  it). If it did, keep it; only if it certainly did not, re-dispatch under the next number.
  A direct re-dispatch leaves two agents on one ticket.

```
Agent({
  subagent_type: "general-purpose",
  name: "ticket-1-<short-feature-name>",
  model: "<implementer model>",
  isolation: "worktree",
  prompt: <ticket prompt>
})
```

Dispatch a parallel batch **in the same message**.

**Recovery / follow-up agent in an existing worktree.** `isolation: "worktree"` always creates a
*new* worktree; the Agent tool's schema has no parameter to reuse an existing one. To put a
recovery or follow-up agent into the current worktree, dispatch it **without** `isolation` and
put in its prompt: the absolute worktree path, the branch name, the expected base, and the rule
"work only inside this path (use `git -C <path>` and absolute paths; never edit the main
checkout)". At review, confirm via `git -C <worktree> status --short` and `git diff <base>..HEAD`
that the changes landed in the worktree and nothing leaked into the main checkout. Alternative (uses only supported behavior): once the old agent has stopped and its work is
committed, dispatch the recovery agent with its own fresh `isolation: "worktree"` and tell it to
first `git merge <old-branch>` (or cherry-pick the listed commits); then retire the old worktree.
Untested: the "no isolation, work by path" route relies on the agent obeying the path rule, so
the leak check above is mandatory.

The ticket prompt must contain (this is the implementer's contract):
1. One sentence on where this ticket sits in the overall project.
2. The full ticket (goal, acceptance criteria, file ownership, test requirements, out of scope).
3. "You work in an isolated worktree and may only touch files in your ownership list; if you
   need to touch anything else, stop and report."
4. "When done, commit to your branch. **Do not push, do not merge.**" (commit style follows
   the project's convention)
5. Report format: `status` (DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED) + dispatch
   number + absolute worktree path + branch name + commit hashes + acceptance evidence table
   + concerns. The Agent call does not return the worktree path and its location is chosen by the
   tool, not by you: record `worktree=pending` at dispatch and fill in the path from the report.
6. "Do not symlink the main repo's dependency directories (`node_modules`, `.venv`, ...) into
   the worktree; install inside the worktree. A symlink makes type checks and tests read the
   main tree's code and exit 0 without verifying your change. Before reporting, run
   `find . -maxdepth 2 -type l \( -name node_modules -o -name .venv \)`; it must print nothing."
   At review the advisor reruns the same command; any output means send it back and **all
   green evidence for that ticket is void**.

Do not paste the whole session history into the prompt. A fresh agent needs only its ticket,
the interfaces it will touch, and global constraints.

### Acceptance evidence table

One row per acceptance criterion, never merged or omitted. Format and judging rules, including
the structural-dependency exception, are in `references/evidence-table.md` (fill-in template: `templates/evidence-table.md`). Core rule: if any
row has an implementer verdict of FAIL / UNVERIFIED, or an acceptance-item row whose advisor re-check is not PASS at merge time, the ticket may not pass and may not be merged.

### Handling reports

- **Check the dispatch number first**: the report's `ticket-N#aK` must equal the one currently
  active in the ledger. Mismatch or missing -> do not accept it, do not review it; append
  `ticket-N#aK stale report ignored` to the ledger and confirm the old agent has stopped.
- **DONE / DONE_WITH_CONCERNS** -> go to review (read concerns first; handle any that touch
  correctness).
- **NEEDS_CONTEXT** -> supply it and continue the same agent.
- **BLOCKED** -> missing context: supply it; task too hard: re-dispatch with a stronger model;
  ticket too big: split it; plan is wrong: ask the user. **Never** re-dispatch the same prompt
  unchanged and hope for a different result.
- **No report / null** (agent was stopped or died mid-way) -> **not** DONE, and commits in the
  worktree do not go straight to review.
  (1) First confirm whether the agent has really stopped. If it is alive, wait or stop it —
  **never** let a second agent into the same worktree (uncommitted work gets overwritten and
  cannot be recovered).
  (2) Reconstruct the state from `git worktree list`, the branch name and the ledger's base:
  `git status --short`, `git log <base>..HEAD`, `git diff <base>..HEAD`, uncommitted changes,
  any edits outside the ownership list.
  (3) If there is usable work, after confirming the original agent has stopped, dispatch a
  recovery agent to finish and resubmit a complete report; if not, re-dispatch from the
  original base in a new worktree. A commit only means recoverable work exists, not that the
  ticket is done; DONE -> evidence gate -> review may never be skipped.

### Active patrol (against abandonment)

Never assume an agent will report. If the advisor just waits, the user sees "abandoned".
- **Patrol every ~10 minutes**; when there is no message to wait on, schedule a wake-up
  (see the prerequisites table).
- **Judge state by facts, not self-report.** Each round, per ticket, take
  `git -C <worktree> log --oneline <base>..HEAD | wc -l` and `git -C <worktree> status --short`
  and compare with the previous round: changed -> working, leave it alone; unchanged and the
  agent is idle -> stalled; more than 3 minutes since dispatch with no commit and no
  uncommitted change -> unclaimed (confirm it is alive, then send a nudge that only points at
  the ticket).
- **Idle with no final report**: (1) look at the worktree and result files first; if there is
  usable work, inspect it, then ask the agent for its full report and evidence table — the ticket is not reviewed or passed until both arrive (a commit is not a report); (2) otherwise nudge — first line says "you have been idle N
  minutes, here is what I am missing", **do not repaste the ticket** (it will be treated as a
  new ticket), and record `nudge #k` in the ledger; (3) two nudges with no movement -> record
  `unverifiable` and tell the user. No response is not the same as stopped: if you cannot
  confirm it stopped, keep patrolling and never send a second agent into the same worktree.
- **Truncated report** -> immediately ask for just the missing part.
- **Decision points**: answer the moment an agent asks; log `ticket-N question qM: <summary>`
  when it arrives and `qM answered` after replying.
- A prompt for an external verifier agent must say "report back immediately when done"; on
  patrol, inspect the result file it writes directly.

## Step 3: Review (the advisor, personally, one ticket at a time)

```bash
git -C <worktree-path> log --oneline <base>..HEAD
git -C <worktree-path> diff <base>..HEAD
```
(base = the full SHA in the ledger; do not use `HEAD~1` — it truncates multi-commit tickets.)

Three verdicts, none optional:
1. **Spec conformance**: check acceptance criteria one by one. Doing too little or too much
   (out-of-scope additions) both fail.
2. **Quality**: correctness, edge cases, whether the tests really verify something, whether
   project conventions are violated. Ask yourself: does what this change relies on hold on
   every execution path? Were all readers found? Were producers and consumers of sibling
   state all updated?
3. **End-to-end walk** (for tickets with a flow contract; a precondition for saying "ready to
   test"): the advisor **personally** runs the contract's path start to finish (for UI, click
   every exit and every back action; for CLI / API, run the whole chain in order), and logs the
   path walked in the ledger. The implementer's screenshots or self-report do not count. If
   you cannot finish the walk, it has not passed. If a cross-ticket E2E truly needs a merge
   first (structural dependency), the named exception in the evidence-table reference
   applies, and the full path is rerun **immediately** after the merge.

**Evidence gate**: no evidence table, missing rows, a PASS row with no reproducible evidence,
or any row whose **implementer verdict** is FAIL / UNVERIFIED (the "Advisor re-check" column
starts as PENDING and is filled during review) -> send it back immediately, do not start
reading code, and do not merge. Before merge, the "Advisor re-check" cell of every acceptance-item row must be PASS (the "Known weaknesses" row has no re-check).

**Content gate**: the evidence gate checks form only. The most common false pass is "evidence
was pasted, but it does not support that acceptance item" (`1 passed` pasted under an item that
tests a different behavior, or stdout containing `1 failed` while marked PASS). The advisor
pairs each row's evidence with "acceptance item text + implementer verdict" and judges
`supports / contradicts / says_nothing`: both `contradicts` and `says_nothing` go back; if
unsure, look at that row yourself. If a mechanical judging tool is available, ask for all rows
in one request — but `supports` does not mean pass, it does not judge whether the evidence is
genuine, and if the call fails you fall back to reading row by row yourself (the mechanical stage is skipped; no other gate is loosened).

**UI tickets (when a design exists)**: every deviation from the design needs a decision
source (who, which day). "Recorded" is not "authorized"; a neatly written deviation table
deserves a source check even more.

**Advisor re-verification** (not reading the implementer's pasted output): rerun **all**
high-risk acceptance items (security / permissions / money, concurrency, data migration,
multi-file core logic; judged item by item), and at least one other item with **real risk** —
picking the cheapest one for form's sake does not count. With no command, walk one manual
path. If you cannot reproduce the result, it has not passed; if the environment will not run,
that item stays UNVERIFIED and blocks merge — the implementer's word is no substitute.

### High-risk tickets: add an independent verifier

For tickets touching security / permissions / money, concurrency, data migration or
multi-file core logic, after the advisor passes it, dispatch a **fresh-context verifier** for
one more pass (the advisor just split the ticket and tends to follow the implementer's logic).
Contract, model selection and convergence rules are in `references/verifier.md`. Low-risk
tickets do not need it; an extra verifier is just spend.

## Step 4: Fix loop

Review not passed -> send findings back to the **same agent** (it has full context, cheaper and
more accurate than a fresh one; without SendMessage see the prerequisites table):
- Findings are specific: file, behavior, expected vs actual.
- After fixing, **update the original evidence table** and rerun every acceptance item the
  finding touches.
- After the fix the advisor **re-reviews**; never pass it just because "it says it's fixed".
- The same finding sent back twice and still not fixed -> re-dispatch with a stronger model,
  or stop and ask the user.

The advisor never edits code itself. The urge to "just fix this one thing" is a violation.

## Step 5: Merge

- Merge dependencies first; use the project's usual merge method.
- A conflict (meaning ownership partitioning missed something) -> send it back to that
  ticket's agent to rebase in its worktree; the advisor does not resolve it.
- **After each merge, run the full test / typecheck suite** before merging the next — each
  ticket green alone does not mean the whole is green.
- When everything is merged, clean up: `git worktree remove <path>` + `git branch -d
  <ticket-branch>`, and confirm with `git worktree list`. Worktrees of superseded / canceled
  tickets must be handled too. If removal fails because of untracked files, list them with
  `git -C <path> status --short`; only when they are all regenerable artifacts (`__pycache__`,
  build output) use `git worktree remove --force`. Anything else untracked: look at it first.
- If the verifier ran through an external CLI that left a long-running background process in
  a worktree, reclaim it after the worktree is deleted (see `references/verifier.md`).
- No `git reset --hard`, no force push; confirm remote / branch before pushing.
- **Pushing needs the user's explicit consent**: merge complete != permission to push. If the
  user said "do not push, I'll tell you when", that is an iron rule.

## Step 6: Watch the deploy

Merge is not the end. Run the project's deploy flow and watch it through:
- Before deploying, confirm the target environment (which host / pipeline); look it up, do not guess.
- For CI/CD projects, watch the pipeline until green.
- After deploying, **verify remotely** — hit the real endpoint, view the real page, read the
  real logs; passing local tests does not count.
- Verification fails -> **preserve the scene first** (full SHA, environment, logs, response,
  screen), then find the root cause. Do not re-deploy first and wash the scene away.

**Freeze during verification** (in effect from the moment deploy starts until verification
ends): the full SHA and target environment go in the ledger and may not change; during that
time do not merge other tickets, do not push, do not re-deploy. It applies equally while the
user is verifying this deploy. Reporting language: say only "deployed, remotely verified X and
Y" — report what you observed.

## Progress ledger (against compaction amnesia)

Long sessions get compacted, and memory alone will re-dispatch finished tickets (the most
expensive failure mode). At the start, create a ledger (in a scratchpad, or a path git ignores — if the repo has
no suitable `.gitignore` entry, `.git/dispatch-ledger.md` works and is never tracked) and append one line per event. Format, vocabulary, the three terminal states
and the pre-close check are in `references/ledger.md`. After compaction, read the ledger and
`git log` before deciding the next step; trust the ledger, not memory.

## Red lines

- Dispatching two tickets that touch the same file in parallel
- The advisor writing implementation code or resolving merge conflicts itself
- Dispatching and then only waiting, with no patrol for over 10 minutes ("abandonment"). Without a scheduling tool the cadence becomes "patrol on every interaction", and the user must have been told so (see the prerequisites table)
- Starting review without a complete evidence table, or merging a ticket whose review failed
- Merging a ticket with any FAIL / UNVERIFIED row, or any acceptance-item row whose advisor re-check is not PASS (except the named structural-dependency exception)
- Dispatching without specifying a model
- Pushing without the user's consent
- Declaring done without remote verification after deploy; merging / pushing / re-deploying while verification is running
- A ticket agent touching files outside its ownership list and review not catching it
- A multi-step flow with no flow contract, or a seam with no owner; dismissing a seam with "out of scope for this ticket"
- Choosing a workaround for a structural obstacle without handing the trade-off to the user
- Telling the user "ready to test" without personally walking the end-to-end path
- Two agents in the same worktree at once (uncommitted work gets overwritten and cannot be recovered)
- Accepting a report with a mismatched dispatch number, or re-dispatching when the Agent call's outcome is unknown
- Treating "no response" as "stopped" and sending a recovery agent into the same worktree
- Relying on advisor self-review alone for a high-risk ticket, skipping the independent verifier; or letting the verifier edit code
