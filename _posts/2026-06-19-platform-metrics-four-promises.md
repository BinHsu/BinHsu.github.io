---
title: "Platform metrics are four promises, not a list to memorize"
subtitle: "Every platform-engineering interview eventually asks how you measure the platform. The weak answer recites DORA. The strong one knows the metrics group into four promises a platform makes — and that most teams only keep two of them. The full reference table, and the one family almost everyone forgets."
date: 2026-06-19 09:00:00
tags: [platform-engineering, sre, dora, observability, devex, finops]
asset_path: /assets/img/posts/2026-06-19-platform-metrics-four-promises
thumbnail-img: /assets/img/posts/2026-06-19-platform-metrics-four-promises/four-promises.png
share-img: /assets/img/posts/2026-06-19-platform-metrics-four-promises/four-promises.png
---

Every platform-engineering interview eventually reaches the same question: *how do you
measure the platform?* And most candidates answer it as a memory test — they list DORA,
maybe SLIs and SLOs, drop a "p99" to sound senior, and stop.

That answer is fine. It's also forgettable, because it treats the metrics as a flat pile of
acronyms. The metrics aren't a list. They're **four promises a platform makes** to the
company that pays for it — and the senior move is to name the promise first, then reach for
the metric that proves it.

![The four metric families as four promises a platform makes]({{ page.asset_path }}/four-promises.png)
*The whole reference in one card. Four families, four promises — and one family, in orange,
that most teams never actually measure.*

## Promise 1 — Delivery: we ship fast without breaking prod

This is the **DORA** family, and it's the one interviewers reach for first. Two of its four
metrics measure *speed*; the other two measure *stability*. The senior framing is that these
are not a trade-off — the whole reason a platform exists is to raise both at once.

| Metric | What it measures | Elite benchmark |
|---|---|---|
| **Deployment Frequency** | Successful prod deploys per unit time | On-demand / many per day |
| **Lead Time for Changes** | Commit → running in production | < 1 day |
| **Change Failure Rate** | % of deploys that degrade prod (hotfix / rollback / patch) | 0–15% |
| **MTTR** | Incident start → service restored | < 1 hour |

## Promise 2 — Reliability: we stay up, on a budget we agreed

This is the SRE vocabulary: the **Four Golden Signals** (Latency, Traffic, Errors,
Saturation) plus the **SLI / SLO / SLA** objective layer and the error budget that makes
reliability a *negotiated* number instead of an absolute.

| Metric | What it measures |
|---|---|
| **SLI** | A measured signal of behavior — e.g. successful ÷ total requests |
| **SLO** | Internal target for an SLI — e.g. 99.9% over a 30-day rolling window |
| **SLA** | External promise, with penalties — looser than the SLO, leaving a buffer |
| **Error Budget** | `1 − SLO`. Spend it on releases; exhaust it and you freeze them |
| **Latency** | p50 / p95 / **p99** — always percentiles, never the average |
| **Traffic** | Load — requests/sec, QPS, concurrency |
| **Errors** | Failure rate — 5xx, failed-request ratio |
| **Saturation** | How close resources are to their ceiling — CPU, memory, IO, pool |
| **Availability** | Uptime — and the "nines" cost more than people think |

The availability number is where interviews catch people, because the downtime math isn't
intuitive:

| SLO | Downtime / month | Downtime / year |
|---|---|---|
| 99% | ~7.2 hours | ~3.65 days |
| 99.9% | ~43 minutes | ~8.76 hours |
| 99.99% | ~4.3 minutes | ~52.6 minutes |
| 99.999% | ~26 seconds | ~5.26 minutes |

## Promise 3 — Adoption: teams actually want to use what we built

Here is the family **almost everyone forgets** — and forgetting it is the difference between
a platform engineer and someone who runs infrastructure. Delivery and reliability measure
whether the platform *works*. Adoption measures whether it was *worth building*.

| Metric | What it measures |
|---|---|
| **Self-Service Adoption** | % of work done via the golden path / portal vs opening a ticket |
| **Cognitive Load** | How much tooling and context a developer must hold to ship (Team Topologies) |
| **Lead Time to Provision** | Request an env / DB / cluster → it's usable (ideal: minutes, self-serve) |
| **Time to First Commit** | New-hire join → first merged PR / first prod deploy |
| **Developer Satisfaction** | DevEx survey, NPS, the SPACE framework |

The principle: a platform is an internal product, and a product is measured by whether people
choose it. The example: I have seen platforms that were green on every DORA and SLO target
and still failed, because teams routed around them — they opened tickets, kept their own
pipelines, and the "platform" was a tax nobody asked for. The punchline:

> A platform that's reliable but unused isn't a platform. It's a house nobody lives in.

If you measure only Promises 1 and 2, you can build that house and never know.

## Promise 4 — Sustainability: we scale without burning cash or people

The last promise is the one that decides whether the platform survives its own success. It
has a money half (**FinOps**) and a human half (**operational toil and on-call**).

| Metric | What it measures |
|---|---|
| **Unit Economics** | Cost per request / per tenant / per transaction |
| **Resource Utilization** | Actual usage vs provisioned — catches over-provisioning |
| **Toil %** | Engineering time on automatable manual ops (SRE guidance: keep < 50%) |
| **On-call Burden** | Alert volume, night pages, MTTA (time to acknowledge) |

Cost and toil are the same failure in two currencies: a platform that scales by spending
more dollars, or by spending more of its team's nights, is not scaling. It's deferring a bill.

![All four families: meaning, what each metric measures, and the benchmark that matters]({{ page.asset_path }}/metrics-full.png)
*The complete reference — save this one. Every metric above, with its meaning, what it
measures, and (for DORA) the elite benchmark.*

## How to actually answer the interview question

When they ask "how do you measure your platform," don't open the acronym fire-hose. Lead with
the promises, and let each one pull in the metric that proves it:

> "I measure a platform against four promises. **Delivery** — are teams shipping fast without
> breaking prod; that's DORA. **Reliability** — are we up, against an error budget we agreed;
> that's SLOs and the golden signals. **Adoption** — do teams actually choose the platform
> over routing around it; that's self-service adoption and cognitive load, and it's the one
> most teams forget to measure. And **sustainability** — does it scale without burning cash or
> people; that's unit economics and toil. The first two say it works. The last two say it was
> worth building."

That answer is the same length as reciting DORA. It just shows you know what the numbers are
*for*.

---

*The card at the top is a one-screen version of this reference — save it. The depth, the
benchmark tables, and the availability math are here. The point that's worth carrying away
without any of the tables: reliability proves the platform works; adoption proves it was
worth building, and that's the promise most teams never put a number on.*
