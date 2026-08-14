# ADR-0003: Shared ALB + shared ECS cluster as a multi-site platform

**Status:** Accepted (2026-06-16; reconstructed 2026-08-13)

## Context

The repo's headline claim is that it is a *platform, not a single-app
deployment*. Making that true required hosting more than one site on the same
infrastructure without standing up a parallel stack per site. Issue #19 ("Design
multi-domain architecture for portfolio sites") recorded the design space as a
set of open questions rather than a settled answer: separate ECS services per
site? CloudFront in front? separate ALBs? a single multi-domain certificate? The
issue framed the trade-offs; it did not decide them.

## Decision

One shared Application Load Balancer and one shared ECS Fargate cluster host
multiple sites. Each tenant site is a `modules/site` instance that rides the
shared ALB + ECS cluster + app security group with its own ECR repo, task
definition, target group, and **host-header listener rule**. TLS is per-domain
via **SNI** — each domain's ACM certificate is attached to the shared HTTPS
listener rather than bundled into one multi-domain certificate. A separately
*registered* apex domain is fronted by a `modules/apex-domain` instance (its own
Route 53 zone + ACM cert on the shared listener) pointing at an existing site's
target group.

The first site on the shared platform was the Lentago (originally Pitzi Labs)
landing site (PR #67, "Host the ... landing site on the shared platform
(modules/site)", merged 2026-06-16). `modules/apex-domain` then fronted sites
with real registered domains: pondviewlane.com (PR #135) and
essexcrossingatmontserrat.com in front of the shared `site_pondview` backend
(issue #137 / PR #138). Listener-rule priorities are kept unique per site/apex
(see `CLAUDE.md` § Module Dependency Graph). In this sense the **code is the
verdict** on the questions issue #19 left open: separate services *were* chosen
(one target group per site), a single shared ALB *was* chosen over separate ALBs,
and SNI per-domain certs *were* chosen over one multi-domain certificate.

## Alternatives

**Recorded at the time** (issue #19, held open as live questions until the code
resolved them)

- *Separate ECS service per site.* Adopted in part — each site gets its own task
  definition and target group — but on the **shared** cluster and ALB rather than
  isolated infrastructure. **Lateral-to-better**: the isolation that mattered
  (independent images, health checks, scaling surface) without duplicating the
  load balancer and cluster.
- *Separate ALB per site.* Rejected. **Worse** for this scale — an ALB is ~$16/mo
  of always-on cost each, and the whole point of the exercise was to show many
  sites sharing one entry point.
- *A single multi-domain (SAN) certificate.* Rejected in favor of per-domain
  certs attached via SNI. **Lateral**: SNI keeps each domain's cert lifecycle
  independent (add/remove a domain without reissuing a shared cert), at the cost
  of one ACM cert + validation record per domain.

**Retrospective — not considered at the time**

- *CloudFront + S3 static hosting.* Most of the hosted sites are largely static,
  and CloudFront+S3 would be **cheaper** and lower-ops for that content — no NAT,
  no Fargate, no ALB. But it is **worse for the thing this repo exists to
  demonstrate**: a shared *container* platform with per-tenant ECS services, ALB
  host-header routing, and OIDC-scoped deploys. Choosing static hosting would
  optimize the hosting bill while deleting the capability the project is built to
  show. A defensible choice for a production static site; the wrong one here.

## Consequences

- Onboarding a new site is additive: a new `modules/site` (or `modules/apex-domain`)
  instance with a unique listener-rule priority, no change to existing sites.
- Listener-rule priorities are a shared, finite namespace that must stay unique;
  `CLAUDE.md` tracks the assignments so two sites don't collide.
- A site's listener rule is a hard dependency of its ECS service's target-group
  wiring — removing a rule can break the service it fronts, so even sites whose
  public hostname moved to an apex domain keep their original rule
  (`create_dns_record = false`), as noted in `CLAUDE.md`.
- All tenants share the ALB and cluster failure domain; an ALB or cluster-wide
  problem is a problem for every hosted site at once.
