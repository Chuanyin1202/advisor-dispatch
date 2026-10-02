# Acceptance evidence table (the implementer's delivery format)

One row per acceptance criterion; rows may not be merged or omitted:

| Acceptance item | How verified | Reproducible evidence | Implementer verdict | Advisor re-check |
|---|---|---|---|---|
| <acceptance criterion, verbatim> | <full command, or explicit manual steps + environment> | <exit code, key output, what was actually observed; path for long logs> | PASS / FAIL / UNVERIFIED | PENDING / PASS / FAIL / UNVERIFIED |

There must be a "Known weaknesses" row (write "none" if there are none).

## Rules

- Automated verification needs the **full command + exit code + enough output to tell which
  behavior was tested**. Pasting only `1 passed` where you cannot see what behavior was tested
  is not evidence.
- **Cannot verify by command != cannot verify.** UI, animation, back paths and on-device
  permissions are verified by hand, but you must state the environment, the steps, the
  expected result and the actual observation. "I checked it" and a single screenshot do not count.
- If there is truly no feasible way to verify, or the environment lacks a dependency /
  credential / running service, mark **UNVERIFIED and state why**. Never guess, never fill in PASS.
- **If any row has an implementer verdict of FAIL / UNVERIFIED, the ticket goes back at the evidence gate. The "Advisor re-check" column starts as PENDING and is filled by the advisor during review; at merge time the cell of every acceptance-item row must be PASS (the "Known weaknesses" row has no re-check), otherwise the ticket may not be merged.**

## The only exception: structural dependency

The behavior under test can exist only after a **named ticket** is merged, so before that merge
it simply cannot be run. A missing package, missing credential, service not up, no device at
hand, inconvenience or lack of time are **all not** structural dependencies; they stay
UNVERIFIED and block the merge.

When the exception is used, the advisor records in the ledger, per item: (1) the structural
reason, (2) which named ticket must be merged first, (3) the verification environment, steps
and expected result after the merge — and **tells the user on the spot**.

- Merge only the tickets *necessary to create that verification condition*.
- Until the follow-up check is done: **no push, no deploy, no declaring it passed**, and no
  merging unrelated tickets.
- If the follow-up check fails, that item becomes FAIL and the fix loop resumes.
- The exception is granted item by item, by name; never apply it to a whole table.
