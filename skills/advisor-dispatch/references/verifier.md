# Independent verifier for high-risk tickets

Applies to tickets touching security / permissions / money, concurrency, data migration, or
multi-file core logic. Dispatch it after the advisor's own review has passed.

## Model selection (decided dynamically, no hard-coded model names)

1. **An external model CLI that actually runs** (optional) -> run the verifier through that
   CLI. A different model family gives the strongest independence (models of one family share
   training blind spots). `which <cli>` only proves the binary exists, not that it is logged
   in or has quota — **run one minimal command** to confirm it responds; on failure, go to
   step 2. This is an optional module; without it, go straight to step 2.
2. External CLI unavailable -> verifier = a fresh subagent **at the advisor's level**.
3. Hard floor: whichever path, the verifier's level is **>= the implementer's**. If the implementer
   was chosen above the advisor's level, set the verifier's `model` explicitly to at least the
   implementer's level. If no such model is available, do not dispatch a weaker verifier: stop and
   tell the user (only the user may waive the floor).

```
Agent({
  subagent_type: "general-purpose",
  name: "verify-ticket-<n>",
  model: "<chosen per the order above>",
  prompt: <verifier prompt>
})
```

Whichever model is the verifier, the contract below is unchanged — you swap the eyes, not the rules.

Before dispatching, the advisor writes the diff to a file (the verifier's **starting point**,
not the only thing it may look at):
`git -C <worktree> diff <base>..HEAD > <scratchpad>/ticket-<n>-diff.txt`

## Verifier contract

- **Isolate the narrative, not the information**: do **not** give it the implementation
  process, the implementer's self-report, or the advisor's review conclusions; but it **may
  read the whole repo read-only** (existing callers, unchanged-but-affected consumers, related
  tests, project conventions). Give it: the ticket's acceptance criteria + the diff file path
  + the repo path (read-only).
- **Adversarial stance**: the job is "find why this diff might be wrong", not "confirm it is
  right". Default to suspicion; pass it only when nothing turns up.
- **Judge, never touch**: return `CONFIRMED` (acceptance met, no counterexample found) or
  `REFUTED` (with a concrete counterexample: which input / path fails). It must never edit
  code itself.
- REFUTED -> treat as a failed review and take the counterexample into the fix loop.
- The prompt for an external-CLI verifier must say "report back immediately when done"; on
  patrol, read the result file it writes.

## Convergence rules (the advisor decides; do not ask the user how many rounds)

A verifier can always find something; without convergence rules tickets bounce forever.
The criterion is **severity**:

1. **The round limit only governs how many times the advisor proactively dispatches**: round
   one looks for product defects; after the fix, round two is narrowed to "round-one findings
   + the claimed fix + its knock-on regressions". After round two, do not proactively dispatch again.
2. **Severe defects are always fixed, regardless of round** (corrupting data, missing
   permission checks, wrong money, duplicate writes under concurrency), and the fix is
   verified again. Quality-level items after round two are the advisor's call; anything left
   unfixed goes into the ticket's "Known weaknesses" and is not sent back.
3. **Still finding a severe defect in round three = the design is wrong**: stop patching and
   tell the user "this ticket needs to be redone or re-split", with reasons.

In reports to the user, list "what was fixed" and "what was decided not to fix, and why" separately.

## Process cleanup for external CLIs (only when one was used)

Some CLIs leave a long-running background process detached from the terminal after running in
a worktree; once the worktree is deleted it stays alive and accumulates. After cleaning up
worktrees, look for orphaned processes whose cwd no longer exists and reclaim them (the method
depends on the CLI; not verified generically). Never touch anything while a job is running.
