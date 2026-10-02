# Progress ledger

Keep it in a scratchpad or a path git ignores (if the repo has no suitable `.gitignore` entry, `.git/dispatch-ledger.md` works and is never tracked); append one line per event:

```
ticket-1#a1 dispatching
ticket-1#a1 dispatched (model=<model>, worktree=pending, base=<full SHA>)
ticket-1#a1 worktree=<path> (filled in from the report)
ticket-1 question q1: <one-line summary>
ticket-1 q1 answered
ticket-1#a1 nudge #1
ticket-1 review round 1: FAIL (<reason>)
ticket-1 review round 2: PASS
ticket-1 merged (<full SHA>); worktree removed
ticket-2#a1 superseded by ticket-2#a2 (<model>): <reason>; a1 agent stopped, worktree handed to a2
ticket-3 canceled: <reason>; worktree removed, branch deleted
decision: <topic> -> <choice> (<date>, <who decided>)
deploy: verified on <env> at <endpoint>
```

Process events use only these words: `dispatching`, `dispatched`, `worktree=<path>`, `nudge #k`, `unverifiable`,
`stale report ignored`, `question qM: <summary>`, `qM answered`, `decision: <topic> -> <choice>`,
`review round N: PASS|FAIL`, `blocked (on <named party>)`.

**A ticket has exactly three terminal states. Each must carry a reason or destination and say
what happens to its agent and worktree; without that it is not closed:**

- `merged (<full SHA>)`
- `superseded by <new ticket or number>: <reason>` (sent back and re-dispatched, re-dispatched
  with another model, replaced by a split). The old agent must have stopped; say whether the
  worktree is handed to the new attempt or removed.
- `canceled: <reason>` (when the advisor cancels on its own, state the reason and tell the user
  on the spot). Remove the worktree and branch together; if work is kept, say where.

**Pre-close check**: every worktree in `git worktree list` must map to a ticket that is still in
progress, or whose terminal state says "reuse / keep"; anything unmatched is a zombie. The
number of questions without an `answered` line must be zero.

After compaction, scan the ledger: any ticket with no terminal line must be investigated; never
assume it is finished or abandoned.
