# ADR-0007: Full codename rename, including live AWS resources and the tfstate backend

**Status:** Accepted (2026-07-08; reconstructed 2026-08-13)

## Context

The repo was renamed `foundry-platform-demo` → `solidago` on 2026-07-03. At that
point the deliberate call was to **keep** the `foundry-*` AWS resource names,
because renaming live infrastructure identifiers in Terraform is mostly
destroy-and-recreate (ECS, ALB, RDS, IAM, KMS) and therefore risky. That call was
reversed days later: a clean stack rebuild was already coming, which removed the
destroy-and-recreate objection, so the codename could be carried all the way
through to the running AWS resources and the state backend rather than stopping at
the repo name.

## Decision

Align **everything** to the `solidago` codename:

- **AWS resource names** — set `var.project = "solidago"` so all
  `${var.project}`-derived names become `solidago-dev-*` (issue #102, PR #104,
  2026-07-08). Safe because the live stack was already destroyed ahead of a clean
  rebuild, *except* the Route 53 zones, which are kept: both zones use
  `name = var.domain_name` (not a project-derived name) with `prevent_destroy`, so
  changing `var.project` is an in-place tag/comment update, never a replace.
- **The shared Terraform state backend** — migrate `foundry-tfstate-*` →
  `solidago-tfstate-*` (issue #103, PR #105, 2026-07-08). Same CMK, re-aliased
  `alias/solidago-tfstate`; the state object keys (e.g. `env/dev/terraform.tfstate`)
  are unchanged.

The rename was later **verified complete against live AWS on 2026-07-25**: a
grooming pass found zero `foundry`-named S3 buckets, ECS clusters, IAM roles, or
KMS aliases, and no `foundry` string in any `.tf`/`.tfvars`. The follow-up tracking
issue #142, filed 2026-07-21 by a fleet rename-discipline sweep, was closed
**overtaken by events** — the work it tracked had already landed via #102/#103 two
weeks before the issue was filed. What remains is historical prose in README /
`CLAUDE.md` / `docs/` correctly recording the former name.

## Alternatives

**Recorded at the time**

- *Do nothing — keep the `foundry-*` names* (the "documenting a legacy name is
  fine" position). This was explicitly the recorded option, and had been the
  active decision from 2026-07-03 until this reversal. Rejected once a clean
  rebuild removed the destroy-and-recreate risk that had justified it. **Lateral
  until the rebuild, then worse**: keeping the old names once they could be
  changed cheaply would leave a permanent mismatch between the codename and the
  running resources, which is exactly the drift the fleet rename discipline exists
  to prevent.
- *Rename the repo only, leave AWS and the backend* — the intermediate 2026-07-03
  state. Superseded by this decision because a half-rename is the drift it set out
  to remove.

**Retrospective — not considered at the time**

- *Rename live resources in place via `terraform state mv` / import, without a
  rebuild.* In principle this could have carried the codename onto running
  resources without waiting for a destroy — **better** if the goal were zero
  downtime on a stack that had to stay up. But for identifier changes that
  Terraform treats as replacements (many `*-name` attributes), `state mv` doesn't
  actually rename the AWS resource, so this would have been a large, error-prone,
  resource-by-resource migration for a stack that was about to be destroyed
  anyway. Given the pending clean rebuild it would have been **worse** — real
  effort and risk to preserve something that was being thrown away.

## Consequences

- Two operational gotchas fall out of the backend being bootstrap-managed and
  self-referential:
  - **Import the Route 53 zones before CI plans.** The kept zones exist outside
    the rebuilt stack, so the first plan must import them or it will try to
    create duplicates.
  - **The first rebuild is a local admin apply.** The state backend the CI role
    reads is the thing being renamed, so there is a bootstrap circularity — the
    first apply after the migration runs from a local admin identity, not the CI
    OIDC role.
- The state object keys were preserved across the bucket rename, so no state
  history was lost in the migration.
- Historical references to `foundry-*` in prose are intentionally retained as an
  accurate record of the former name; they are not drift and are not to be
  "fixed."
