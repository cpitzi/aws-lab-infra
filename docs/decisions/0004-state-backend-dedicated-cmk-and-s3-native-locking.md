# ADR-0004: State backend — dedicated bootstrap-managed CMK + S3-native locking

**Status:** Accepted (2026-07-01; reconstructed 2026-08-13)

## Context

The Terraform state bucket started life with AES256 (SSE-S3) encryption
(decision #2 in `docs/reference-notes.md`, 2026-02-27) as a deliberate stopgap to
avoid a KMS dependency before the KMS module existed. Issue #15 tracked upgrading
it to a customer-managed key. The obvious move — reuse the Terraform-managed CMK
(`alias/solidago-dev-main`) that already encrypts everything else — turns out to
be unsafe. Separately, the backend originally used a DynamoDB table for state
locking, which Terraform 1.10+ makes unnecessary.

## Decision

**Encrypt the state bucket with a *dedicated, bootstrap-managed* CMK, not the
Terraform-managed key.** PR #75 (Closes #15) makes a three-point argument for a
separate key, all three verified in the PR:

1. **Chicken-and-egg** — the state bucket must exist and be encryptable *before*
   Terraform runs, so its key cannot itself be a Terraform resource.
2. **Circular dependency** — the Terraform-managed key is *defined in* the state
   that lives in the bucket; encrypting the bucket with it means the key depends
   on a state file only readable with that key.
3. **Destroy safety** — routine `terraform destroy` teardowns schedule the
   Terraform-managed key for deletion (30-day window); if it encrypted the state
   bucket, a teardown would lock the operator out of their own state.

The dedicated key is created in the bootstrap script, outside Terraform, and is
never touched by Terraform. This supersedes the early SSE-S3 call (decision #2)
and is recorded as decision #35 in `reference-notes.md`. (The key was aliased
`alias/foundry-tfstate` when PR #75 landed; it was re-aliased
`alias/solidago-tfstate` during the later backend migration — see
[ADR-0007](0007-full-codename-rename-including-live-aws-and-tfstate.md).)

**Drop the DynamoDB lock table for S3-native locking.** PR #74 (Closes #13)
migrates the backend to `use_lockfile = true`, removing the separate DynamoDB
lock table entirely.

## Alternatives

**Recorded at the time**

- *Reuse the Terraform-managed CMK `alias/solidago-dev-main`* — the option issue
  #15 originally suggested. Rejected in PR #75 for the three reasons above.
  **Worse**: it is the specific choice that would make a routine teardown able to
  lock the operator out of state.
- *Keep AES256 (SSE-S3)* — the incumbent (decision #2). Superseded: a
  customer-managed key gives explicit key policy control and a clean audit story
  that SSE-S3 does not. **Lateral-to-better** for a platform meant to model
  production key management.

**Retrospective — not considered at the time**

- *Keep the DynamoDB lock table despite S3-native locking being available.*
  **Strictly worse** now that `use_lockfile` exists: DynamoDB-based locking is an
  extra always-on resource, an extra IAM surface, and an extra thing to bootstrap
  and pay for, all to provide a guarantee S3 conditional writes now provide
  natively. There is no scenario at this scale where the DynamoDB table earns its
  keep once `use_lockfile` is an option.

## Consequences

- The state CMK lives entirely outside Terraform's lifecycle, so no
  `terraform destroy` can schedule it for deletion — state stays readable across
  teardown/standup cycles.
- Because the key and bucket are bootstrap-managed, creating/rotating the key or
  re-encrypting the bucket is a **human step** (run the bootstrap script, then
  `terraform init -reconfigure`), not something the CI Terraform pipeline does.
- Existing state objects written under AES256 stay readable; only new writes use
  the CMK, so the upgrade needed no state migration or downtime.
- IAM (not the key policy) grants the CI roles scoped access to the state CMK,
  because those roles do not exist at bootstrap time.
- The backend has one fewer moving part: no DynamoDB lock table to provision,
  monitor, or pay for.
