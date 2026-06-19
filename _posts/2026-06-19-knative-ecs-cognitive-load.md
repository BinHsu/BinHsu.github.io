---
title: "Knative, ECS, and the cognitive load that won't disappear"
subtitle: "ECS and Cloud Run feel effortless because someone else runs the substrate. Knative can match that developer experience — but self-host it and the load doesn't vanish, it moves to your platform team. A look at where the simplicity actually comes from."
date: 2026-06-19 05:00:00
tags: [knative, ecs, kubernetes, platform-engineering, serverless]
asset_path: /assets/img/posts/2026-06-19-knative-ecs-cognitive-load
thumbnail-img: /assets/img/posts/2026-06-19-knative-ecs-cognitive-load/load-is-conserved.png
share-img: /assets/img/posts/2026-06-19-knative-ecs-cognitive-load/load-is-conserved.png
---

> Companion piece: [AWS Blocks is the platform team you don't have](/aws-blocks-is-the-platform-team-you-dont-have/) — the same question, from the infrastructure-from-code angle: who owns the layer the abstraction hides.

Do you still miss the low cognitive load of ECS and Cloud Run? You push a container, you
get a URL, it scales, and you never think about a cluster. Then you move to Kubernetes and
suddenly you're reasoning about nodes, control planes, autoscalers, and forty lines of YAML
before "hello world" answers a request.

Knative is the open-source thing people reach for to get that feeling back. And it can —
mostly. But "low cognitive load" turns out to be two different things bundled together, and
self-hosted Knative only hands you one of them for free. The other one doesn't disappear.
It moves.

![Cognitive load is conserved: the developer's half shrinks the same way in all three; only who carries the operator's half changes]({{ page.asset_path }}/load-is-conserved.png)
*The whole argument in one picture. The green "Dev" block is small in all three — that's
the gift. The big operator block is the same size everywhere; only the owner changes.*

## "Low cognitive load" is two things, not one

ECS on Fargate feels effortless because two separate burdens are gone at once:

1. **The application-facing burden** — the model you deploy against. ECS's task definition
   is simpler than a pile of Kubernetes Deployments, Services, HPAs, and Ingresses.
2. **The operator-facing burden** — who runs the thing underneath. With Fargate you never
   see a cluster, a node, or a control plane. AWS runs all of it.

Bundle those and you get "I push a container and forget about it." Unbundle them and you can
see exactly what Knative does and doesn't give you.

| | The developer's view | Who runs it underneath |
|---|---|---|
| **ECS + Fargate** | Task definition → a running service | AWS runs everything — no cluster, no nodes |
| **Cloud Run** | Container → a URL, scale-to-zero | Google runs everything (it's Knative underneath) |
| **Self-hosted Knative** | One `Service` → a URL, scale-to-zero — ECS-or-better | **You** run the cluster, Knative, the networking layer, the Activator |

## Cloud Run is the tell

Here's the detail that explains everything: **Google Cloud Run is Knative Serving with
Google's operations layer on top.** Not "inspired by" — that's the architecture. Cloud Run
feels as good as ECS, better even, with scale-to-zero, for one reason: Google absorbed the
operator-facing burden. You get Knative's application model, and Google runs the substrate.

So "can Knative feel like ECS" already shipped an answer. It's called Cloud Run. The
interesting question is what happens when *you* run Knative instead of Google.

## Self-hosted Knative: the developer keeps the gift, the platform team gets the bill

Point Knative at your own EKS cluster and, for the application developer, it is genuinely
ECS-or-better:

- one `Service` resource → a URL with request-driven autoscaling and scale-to-zero
- immutable revisions and traffic splitting for canary and rollback, built in

For the developer, cognitive load drops to Cloud-Run levels. But someone now runs Knative:
the cluster itself, the Knative install, the networking layer it needs (Istio, Kourier, or
Contour), the Activator that sits in the request path during cold starts, autoscaler tuning,
and version upgrades. That burden didn't evaporate when the developer stopped seeing it. It
moved to the platform team.

This is the whole point. **Cognitive load is conserved.** ECS and Cloud Run don't delete it
— they relocate it to the cloud provider, and you pay in lock-in and dollars. Self-hosted
Knative relocates it to your platform team, and you pay in operational surface — in exchange
for portability and scale-to-zero on infrastructure you control.

## Where it's the right tool: fast validation on a cluster you already run

The relocation math flips in your favor in one common case: **you already run Kubernetes.**
If a platform team already operates EKS, the marginal cost of adding Knative is small and the
payoff is large:

- every team gets ECS-like self-service on the substrate the platform team already runs
- scale-to-zero makes ephemeral validation environments nearly free to leave running — they
  cost nothing at idle
- per-revision URLs and traffic splitting make preview-and-canary a default, not a project

The one caveat is cold starts. Scale-to-zero means the first request after idle waits for a
pod to come up (the Activator buffers it in the meantime). For validation environments that's
a non-issue. For latency-critical production you pin a minimum scale above zero — which trades
away the cost saving that made scale-to-zero attractive in the first place. That trade is the
design decision, not a bug.

## Can it be true production?

Yes — and it already is. Knative graduated in the CNCF in October 2025, and Cloud Run runs
production workloads on Knative Serving at scale. But "production-grade" and "right for your
workload" are different claims. Knative's design center is request-driven, stateless, bursty,
event-driven work where scale-to-zero pays off. It's a poor fit for long-running stateful
services, daemons, or anything needing stable long-lived connections — exactly the steady
workloads ECS handles without fuss. And running Knative in production means running its full
stack reliably: the networking layer and the Activator become critical-path components you own.

ECS and Knative aren't competing for the same production. ECS is a managed, general-purpose,
long-running orchestrator with no scale-to-zero and no cluster to think about — at the cost of
being AWS-only. Knative is an elastic, portable, serverless layer for request-driven work — at
the cost of owning, or renting, the substrate underneath.

## The question isn't "can Knative match ECS"

It can, for the developer. The real question is who you want holding the cognitive load you
stopped seeing. Hand it to a cloud provider and you buy effortlessness with lock-in. Keep it
on your platform team and you buy portability and elasticity with operational surface. "I push
a container and forget about it" is never free — someone, somewhere, is remembering it for you.
The engineering decision is choosing who.

## One honest limit on this read

I've mapped this from the architecture and from running Kubernetes in production, not from
operating self-hosted Knative at scale myself. The boundary I'm drawing — developer-load
versus operator-load, and where each one lands — is structural. If I run Knative in anger and
the operational surface turns out smaller or larger than I've framed it, I'll say so.

---

*This site is the lab side of my work. The polished portfolio lives at [binhsu.org](https://binhsu.org).*
