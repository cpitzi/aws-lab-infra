# ADR-0002: GitHub OIDC only, with a dual-role trust split

**Status:** Accepted (2026-02-28; reconstructed 2026-08-13)

## Context

CI/CD needs to reach AWS to push container images, update the ECS service, and
run the Terraform pipeline. The default path — long-lived IAM access keys stored
in GitHub Secrets — puts standing credentials in a place that is copied into
every workflow run and can leak. This is a single-account learning lab with one
operator, but the stated intent is to model production security posture, so
credential handling is treated as if it mattered.

A second concern is blast radius. If one set of credentials could both deploy
application containers *and* mutate infrastructure, a compromise of the workload
deploy path would also be a compromise of the whole platform.

## Decision

**No stored AWS credentials.** Both pipelines authenticate through GitHub's OIDC
provider: the workflow presents a short-lived GitHub JWT to STS and assumes an
IAM role. This is recorded as decision #7 in `docs/reference-notes.md`
(2026-02-28).

**Two roles, not one** (`modules/iam`, and `docs/WORKLOAD_RELATIONSHIP.md`):

- `solidago-dev-github-actions` — assumed by *workload* repos to push to ECR and
  update the ECS service. Trust scoped to the workload repo's OIDC subject.
- `solidago-dev-github-actions-terraform` — assumed by *this* repo's `terraform`
  GitHub environment to plan and apply infrastructure.

Neither role can do the other's job: a compromised workload deploy cannot mutate
infrastructure, and a compromised Terraform pipeline cannot push arbitrary images
without going through Terraform.

The split became concrete when the application source was extracted out of this
repo into its own workload repo (issue #55, "Phase 11: extract workload from
platform"; issue closed 2026-05-26). That extraction replaced an earlier
cross-repo `repository_dispatch` trigger — where a content repo fired the deploy
that ran *inside* the platform repo — with the workload repo owning its own
deploy workflow and assuming the platform-owned deploy role via OIDC.

Repo/site renames have been bridged by temporary **dual-trust** windows so a
deploy never breaks mid-rename (issue #88 / PR #89, the `ice-cream-book` →
`site-icecreamtofightwith-com` site-repo rename). Newer trust entries prefer an
**immutable OIDC subject claim** over a mutable `repo:owner/name:*` string so a
future rename of the *repo* cannot silently transfer deploy rights (PR #132
dual-trust, PR #133 switching to the immutable subject claim, PR #134 pruning a
stale trust entry).

## Alternatives

**Recorded at the time**

- *IAM access keys in GitHub Secrets* (decision #7, `reference-notes.md`). The
  incumbent default. Rejected: long-lived credentials that must be rotated,
  stored, and are copied into every run. **Worse** — strictly more standing
  secret material for no offsetting benefit here.
- *The `repository_dispatch` incumbent* (issue #55). A content-source repo fired
  a dispatch that ran the build/deploy inside the platform repo, keeping the
  application coupled to platform infrastructure. Rejected in favor of a clean
  platform/workload split. **Worse** for the "platform, not a single-app
  deployment" goal — it kept application code and its deploy path inside the
  platform repo.

**Retrospective — not considered at the time**

- *A separate AWS account per workload* (e.g. via AWS Organizations). **Better**
  isolation on paper: a workload compromise cannot touch another account at all,
  and billing/quota blast radius is contained. But **heavier** than warranted
  here — a single-operator lab would pay the full cost of cross-account role
  assumption, Organizations management, and per-account bootstrap for isolation
  that the dual-role split already approximates within one account. A reasonable
  next step only if real multi-tenant isolation were required.
- *GitHub deploy keys or personal access tokens* for the deploy path. **Worse**
  than the chosen OIDC design on every axis that mattered: still long-lived
  secret material, still stored in GitHub, and with none of OIDC's per-run,
  per-subject scoping. It would have reintroduced exactly the standing-credential
  problem the decision set out to remove.

## Consequences

- There are no AWS access keys in GitHub Secrets for either pipeline; rotation of
  standing deploy credentials is a non-problem because there are none.
- The workload/platform boundary is enforced by two independently scoped trust
  policies, so onboarding a second workload is additive (a new trust subject +
  new ECR/ECS/target-group resources) rather than a change to the first
  workload — see `docs/WORKLOAD_RELATIONSHIP.md`.
- Repo and site renames now carry an OIDC-trust checklist: open a dual-trust
  window, cut over, then prune — and prefer the immutable subject claim so the
  next rename is trust-stable by construction.
