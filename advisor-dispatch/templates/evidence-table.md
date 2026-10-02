# Acceptance evidence table template

ticket-<N>#a<K>

| Acceptance item | How verified | Reproducible evidence | Implementer verdict | Advisor re-check |
|---|---|---|---|---|
| <acceptance criterion 1, verbatim> | `<full command>` | exit 0; <key output showing which behavior was tested>; log: <path> | PASS | PENDING |
| <acceptance criterion 2, verbatim (manual)> | Environment: <...>; steps: <...>; expected: <...> | Observed: <...> | PASS | PENDING |
| <acceptance criterion 3, verbatim> | <...> | <reason: missing dependency / service will not start, etc.> | UNVERIFIED | PENDING |
| Known weaknesses | n/a | <weakness and why; write "none" if none> | n/a | n/a |

The advisor fills the "Advisor re-check" column. A ticket with any FAIL / UNVERIFIED verdict, or any acceptance-item row whose re-check is not PASS at merge time,
may not be merged (structural-dependency exception: see references/evidence-table.md).
