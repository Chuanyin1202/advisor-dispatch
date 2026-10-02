# Ticket template (paste into the Agent prompt; fill in the < >)

Dispatch number: ticket-<N>#a<K> (echo it back unchanged in the report)
Expected base: <full SHA> (check with `git rev-parse HEAD` before starting; stop and report if it differs)

## Position
This is ticket <N> of "<project / feature>", responsible for <one sentence>.

## Goal
<one sentence: what to do>

## Acceptance criteria (each must be verifiable)
1. <e.g. POST /x with a duplicate position returns 409; test test_xxx passes>
2. <e.g. from page A, back returns to page B and keeps the filter (manual, environment: ...)>

## File ownership (you may touch only these)
- <path/or/dir>

## Test requirements
- Run: <full command>
- Add: <tests to add and the behavior they cover>

## Out of scope
- <explicitly excluded items>

## Working rules
- You work in an isolated worktree and may only touch files in the ownership list; if you need
  to touch anything else, stop and report.
- Install dependencies inside the worktree; never symlink the main repo's node_modules / .venv.
  Before reporting run `find . -maxdepth 2 -type l \( -name node_modules -o -name .venv \)`;
  it must print nothing.
- When done, commit to your branch (<project commit convention>); do not push, do not merge.

## Report format
status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
ticket: ticket-<N>#a<K>
worktree: <absolute path>
branch: <branch name>
commits: <hashes>
evidence table: <per references/evidence-table.md, one row per acceptance criterion, including a "Known weaknesses" row>
concerns: <if any>
