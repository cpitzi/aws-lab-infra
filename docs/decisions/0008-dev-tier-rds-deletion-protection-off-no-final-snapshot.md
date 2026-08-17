# ADR-0008: Dev-tier RDS runs with deletion protection off and no final snapshot

**Status:** Accepted (2026-03-19; documented 2026-08-17)

## Context

Solidago dev is a cost-managed lab platform with a scripted teardown/standup
cycle (`scripts/teardown.sh`, `scripts/standup.sh`, `docs/RUNBOOK.md` —
[ADR-0005](0005-selective-teardown-as-the-cost-model.md)). RDS is one of the
resources that cycle touches: the default path *stops* the instance
(`RDS_MODE=stop`) to preserve data across a routine nightly teardown, but the
runbook also documents `RDS_MODE=destroy` for reclaiming storage cost on an
idle stretch longer than the 7-day auto-start window, and nothing prevents an
ad hoc `terraform destroy -target=module.rds...` outside the scripts either.

`aws_db_instance.this` in `modules/rds/main.tf` sets `deletion_protection =
false` and `skip_final_snapshot = true`. Both are the inverse of the AWS
defaults for a reason: `deletion_protection = true` would make every destroy
path — scripted or ad hoc — fail closed until an operator manually disabled
protection first, which defeats a *scripted* teardown. `skip_final_snapshot =
false` would force a final snapshot on every destroy, which needs a unique
identifier supplied at destroy time (another manual step non-interactive
automation can't provide) and, worse, would leave a snapshot behind every
`RDS_MODE=destroy` cycle — paying storage cost forever for data this
environment treats as disposable in the first place.

Read cold, outside that context — in a security or architecture review — a
production RDS instance with deletion protection off and no final snapshot
reads as a gap, not a decision. That is what this ADR corrects.

## Decision

Keep `deletion_protection = false` and `skip_final_snapshot = true` on the
dev-tier RDS instance, deliberately, so that both the scripted teardown path
and a manual `terraform destroy -target` can complete unattended, and so that
`RDS_MODE=destroy` cycles don't accumulate orphaned final snapshots for data
nobody intends to keep.

## Consequences

- **Data in `solidago-dev-postgres` is disposable by design** and must never
  be treated as durable. Anything that needs to survive a teardown/standup
  cycle — reference data worth keeping, hard-to-reproduce fixtures, anything
  with a recovery expectation — does not belong in the dev database alone; it
  belongs somewhere outside this teardown's reach (version-controlled seed
  data, a snapshot taken and named by hand, or a different environment
  entirely).
- The default `RDS_MODE=stop` path already avoids exercising this trade-off
  most nights — stopping preserves the data without touching either setting.
  These two settings matter specifically for `RDS_MODE=destroy` and for any
  `terraform destroy` that reaches the instance outside the scripts.
- This is scoped to the **dev tier only**. It is not a statement that
  deletion protection or final snapshots are unnecessary in general — see the
  production guidance below.

## Reversal condition

If solidago-dev, or any future environment built from this module, ever holds
data that matters — real user data, anything without an independent source of
truth, anything with a recovery SLA — both settings flip for that
environment: `deletion_protection = true` and `skip_final_snapshot = false`
(with an explicit `final_snapshot_identifier`). At that point the environment
should also come out of the selective-teardown rotation ([ADR-0005](0005-selective-teardown-as-the-cost-model.md))
rather than staying in it with `RDS_MODE=destroy` disabled by convention —
a setting that only holds because everyone remembers not to use it is not a
guarantee.

## Guidance for a production tier

A production instance built from `modules/rds` should not inherit dev's
values — they should be set explicitly per environment, not left at the
module's dev-tuned defaults:

- `deletion_protection = true` — a destroy should require a deliberate,
  reviewed change (disable protection, then destroy), never a single command.
- `skip_final_snapshot = false`, with a `final_snapshot_identifier` — the
  last state of the database is always recoverable after a destroy.
- Production has no place in a scripted `teardown.sh`/`standup.sh` rotation at
  all; that automation exists to manage dev's idle cost, not to take a
  production database down nightly.
