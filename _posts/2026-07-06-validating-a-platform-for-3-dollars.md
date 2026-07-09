---
title: "Validating a platform for $3: a frugal validation pyramid for solo platform engineers"
description: "No employer footing the bill, no shared sandbox with a leave-it-running culture — every hour a control plane stays up is my money. That constraint quietly made my platform worse, until I inverted who owns what."
date: 2026-07-06 09:00:00
tags: [aws, platform-engineering, terraform, argocd, gitops, finops]
asset_path: /assets/img/posts/2026-07-06-validating-a-platform-for-3-dollars
image:
  path: /assets/img/posts/2026-07-06-validating-a-platform-for-3-dollars/hero-og.png
---

I run a three-repo AWS platform portfolio on my own budget. There is no employer footing the bill, no shared sandbox account with a "just leave it running" culture. Every hour a control plane stays up is my money. That constraint is not a footnote to the architecture — it *is* an architectural force, and for a while it was quietly making my platform worse. This post is how I noticed, and the design I am migrating to.

## The setup

The portfolio is the classic layered stack, split across three repositories:

```
landing-zone   → org, accounts, guardrails, IAM baseline
platform       → EKS, ArgoCD, Crossplane, Kyverno
                 └─ gitops/platform-addons/  — GitOps desired state, in-repo
app            → the workload itself
```

![Three repositories, one layered stack — Terraform provisions the account and cluster substrate; ArgoCD reconciles it toward GitOps config that lives inside the platform repo, which describes the workload.]({{ page.asset_path }}/repo-stack.png)

Nothing exotic. Terraform stands up the infrastructure, ArgoCD reconciles the cluster toward the GitOps config — which deliberately lives inside the platform repo rather than in a separate config repo, so infrastructure and desired state version together — Kyverno enforces policy, Crossplane brokers cloud resources. The layering is deliberate and I stand by it.

The problem was not the layers. It was what frugality did to *ownership*.

## How saving money warped the machine

Because I pay real bills, I validated by hand: spin the whole stack up, check it, tear it all down, repeat tomorrow. Manual spin-up, manual teardown. Sensible on the wallet. But teardown is where the design rotted.

I had let Terraform own the Helm releases — the ArgoCD install, the controllers, the addons. That reads fine on the way *up*. On the way *down* it is a nightmare. Terraform tries to delete a Helm release whose CRDs are mid-finalizer; the release owns a LoadBalancer Service that provisioned an ELB and a security group; the security group now has cross-references that the VPC destroy trips over. So I grew machinery to compensate: bespoke teardown scripts, `terraform state rm` loops to drop stuck resources, a security-group "reaper" to sweep the ENIs and SGs that Kubernetes left orphaned.

Here is the uncomfortable read on that: **I was paying for frugality in machine complexity.** Every dollar I saved by tearing down nightly, I spent back in fragile teardown code that existed only to unstick a dependency graph I had built wrong.

## The inversion: the cluster is cattle, everything inside dies with it

The fix is to stop treating in-cluster objects as things Terraform owns and must carefully dismantle. Treat the cluster as cattle. When it goes, everything inside it goes *with* it — no careful dismantling, because there is nothing to dismantle when the EKS control plane and its nodes simply cease to exist.

![Before/after ownership: Terraform-owns-everything is a tangled dependency graph needing bespoke teardown scripts; Terraform-owns-3-things plus an ArgoCD app-of-apps is a clean tree where teardown collapses to destroying the VPC, EKS, and the bootstrap.]({{ page.asset_path }}/ownership-split.png)

Concretely, the target ownership split looks like this:

```
Terraform owns:   VPC + EKS + a bootstrap ArgoCD           (3 things)
ArgoCD owns:      every addon, controller, and policy      (app-of-apps)
                   ── Kyverno, ALB controller, external-dns,
                      Crossplane, the workload, all of it
```

Most of that migration has already merged: Kyverno, Alloy, Crossplane, Argo Rollouts, and the workload ApplicationSet all live under the app-of-apps now. What remains is the last two Terraform-owned Helm releases (the ALB controller and external-dns) and the payoff step — the ephemeral profile and deleting the teardown machinery.

Once that lands, Terraform's blast radius shrinks to three resources: the VPC, the EKS cluster, and the single bootstrap ArgoCD that pulls in the app-of-apps. Everything else becomes a child of that app-of-apps and lives entirely inside the cluster. Teardown stops being a delicate ordered dance and collapses to: destroy the VPC, destroy EKS, destroy the bootstrap. The reapers and the `state rm` loops go away, because the orphans they cleaned up are no longer created outside Terraform's knowledge — they die inside the cluster when the cluster dies.

This is an old problem with an industry answer, not a private epiphany. AWS hit the same wall at scale: [EKS Blueprints v5](https://aws-ia.github.io/terraform-aws-eks-blueprints/v4-to-v5/motivation/) deliberately removed the Terraform-managed add-on and application layer for exactly these lifecycle reasons, and the follow-on [GitOps Bridge](https://github.com/gitops-bridge-dev/terraform-helm-gitops-bridge) project exists to hand the baton from Terraform to ArgoCD cleanly. The tooling in that space is still community-evolving, but the directional call — Terraform provisions, GitOps operates — is settled.

The same migration adds an *ephemeral profile*: a validation-only variant that skips the public ALB and external-dns entirely. If I am proving a policy or a reconciliation path, I do not need a public DNS name or an internet-facing load balancer. Skipping them removes the two slowest-to-provision, slowest-to-delete, most-orphan-prone pieces from every throwaway run. The profile is scoped to non-ingress claims — policy, reconciliation, IAM and Pod Identity, IPAM; when the ALB or DNS seam is itself under test, a separate ephemeral run keeps them enabled.

## The general pattern: a validation fidelity pyramid

The teardown fix was a symptom cure. The disease was that I was validating everything on the most expensive substrate available. So I wrote down a rule and built to it:

> **Test on the cheapest substrate that can falsify the claim.**

Fidelity is a cost. You only buy the fidelity a given claim actually requires to be proven false. That gives a pyramid:

![The validation fidelity pyramid: L0 validate/plan/render at $0 per commit, L1 a local disposable kind cluster at $0 per pull request, L3 real ephemeral AWS for cloud-coupled seams only at under $3, TTL-reaped, per release.]({{ page.asset_path }}/fidelity-pyramid.png)

**L0 — $0, runs on every commit.** `terraform validate`, `terraform plan`, Helm template render, `kubeconform`, Kyverno policy render against rendered manifests. This falsifies a large class of claims — "this is valid," "this policy would match," "this plan does what I said" — for nothing, in seconds. Most of my mistakes die here.

**L1 — $0, a local disposable cluster.** A claim like "this Kyverno policy blocks a privileged pod" or "this ArgoCD ApplicationSet fans out to the right namespaces" is not cloud-coupled at all. It is Kubernetes behavior. So I prove it on a disposable kind cluster in CI, on every PR: a real API server, real admission control, real reconciliation, for free. Kyverno negative tests (assert the bad thing is *rejected*) and the ArgoCD golden path both run there today and gate the merge. This is the industry's answer, not a private trick — Kyverno's and Gatekeeper's own CI suites run on kind for exactly this class of validation, and the [Google SRE book's testing chapter](https://sre.google/sre-book/testing-reliability/) makes the underlying argument: catch failures in the cheapest, fastest tier that can catch them. I also built an Apple-Silicon k3s launcher (`aegis-apple-container-provisioner-k3s`) that could become a richer local lane later; today kind carries L1 on its own. If a behavior does not depend on an AWS API, it has no business costing AWS money to test.

**L3 — real AWS, but only for the seams that are genuinely cloud-coupled.** Some things you cannot fake locally: EKS Pod Identity, the AWS Load Balancer Controller actually reconciling an ELB, VPC IPAM handing out addresses. These are the seams where the cloud is the system under test. For those, I spin an ephemeral cluster, prove the seam, and reap it. The cost math is the whole point:

```
2-hour ephemeral EKS run
  EKS control plane   $0.10/hr × 2      = ~$0.20
  spot worker nodes   small, short-lived = ~a dollar-ish
  NAT gateway         skipped (ephemeral profile)
  public ALB          skipped (ephemeral profile)
  ─────────────────────────────────────────────────
  total                                  < $3, TTL-reaped
```

Those two skips are the non-ingress profile. When the ALB or external-dns seam is the claim under test, that run keeps them enabled — the seam you are proving is never the seam you skip.

Under three dollars, and it runs **per-release, not per-PR**. The cheap layers absorb the per-change volume; the expensive layer only fires when I am about to cut a release and need the cloud-coupled seams confirmed against the real thing.

The [dev/prod-parity](https://12factor.net/dev-prod-parity) school — and its testing-in-production cousin — would object that every tier below real AWS narrows what you have actually validated. Agreed. That is exactly why the pyramid names its cloud-coupled seams explicitly instead of pretending kind is EKS.

## Break-glass, formalized

One more change from frugality-driven habits. The platform keeps an operational-recovery break-glass: an emergency IAM role scoped to account-level recovery — IAM, KMS, SSM — for when something like a teardown fails partway through and leaves the account half-dismantled. (AWS treats IdP-outage break-glass as a separate problem; this is not that.) I will not pretend the discipline was there from day one. The role was deliberately designed — about three and a half weeks into the project — but the git history also records the ad-hoc kind: a local `terraform apply` under pressure, days after I had written the principle document that forbade exactly that. The discipline was learned, then hardened. Two controls are already in place: time-boxing (a one-hour `max_session_duration` on every break-glass role, across all seven accounts) and a written justification — a manual discipline where every use gets a documented reason and an incident entry. What remains is closing the gap between manual and enforced: alarming (today a break-glass event raises no alert — a known, documented gap), automated enforcement of the justification, and credential rotation after use, which AWS Well-Architected ([SEC03-BP03](https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_emergency_process.html)) treats as load-bearing for emergency access. Same emergency capability, but the goal is an event I can see rather than a door I left open.

## Where this actually is

Honest framing: L0 and L1 are live — the kind-based end-to-end harness gates CI today. The ownership inversion is most of the way through: the add-ons have moved under the app-of-apps, and the remaining work is the last two Terraform-owned Helm releases plus the payoff step — the ephemeral profile and deleting the teardown machinery. The k3s launcher exists as a possible richer local lane, but nothing depends on it yet. I am publishing before the last step lands because the reasoning is the reusable part, and because writing the cost math down is what forced me to stop paying for frugality in complexity.

The one-line version I would tell my past self: if saving money is making your automation more elaborate, you have inverted something. Move the ownership boundary, not the cleanup script.

If you run a solo or small-team platform on your own coin, I would like to hear where your fidelity pyramid sits — especially which seams you have found you genuinely cannot push below real cloud.

---

*This site is the lab side of my work. The polished portfolio lives at [binhsu.org](https://binhsu.org).*
