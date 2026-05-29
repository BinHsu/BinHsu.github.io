---
title: "After Jenkins, GitHub Actions feels fragmented"
subtitle: "The thing you miss isn't Jenkins — it's a single control plane"
date: 2026-05-29 08:00:00
tags: [cicd, jenkins, github-actions, platform-engineering]
asset_path: /assets/img/posts/2026-05-29-after-jenkins-gha-feels-fragmented
thumbnail-img: /assets/img/posts/2026-05-29-after-jenkins-gha-feels-fragmented/diagram.png
share-img: /assets/img/posts/2026-05-29-after-jenkins-gha-feels-fragmented/diagram.png
---

> Companion piece: [AWS CLI or AWS MCP?](/aws-cli-vs-mcp/) — same line of thinking: keep the control plane separate from the thing that executes.

I grew up on Jenkins. A year into GitHub Actions, the word I keep reaching for is **fragmented**.

Not worse. Fragmented. On Jenkins my pipelines, my shared logic, my self-service jobs, and my run history all lived in one place I could reason about. On GHA the same capability is real — but it's scattered across YAML files, repos, marketplace actions, reusable workflows, and ephemeral runners. The feeling has a cause, and it has a fix that isn't "go back."

![Jenkins is one surface; GitHub Actions scatters it; a control plane re-unifies it]({{ page.asset_path }}/diagram.png)

## What you're actually missing

People say they miss Jenkins. Pin it down and it's almost never the tool — it's the **single, programmable control surface**.

![What you actually miss about Jenkins]({{ page.asset_path }}/miss-table.png)

| What you miss | Jenkins had it | GHA's substitute (and the gap) |
|---|---|---|
| Real logic | Scripted pipeline + shared libraries (Groovy: loops, functions) | YAML + expressions; reusable workflows help but can't express logic; ~4-level nesting cap |
| Stage-level parallel + dynamic | Parallel stages; stages generated at runtime | Only jobs run parallel; dynamic = matrix-from-JSON (two-step, awkward) |
| Mid-pipeline human gate | `input` step pauses anywhere for approval | Environment protection rules — coarser, env-scoped |
| State + debuggability | Build history, workspace, SSH in, replay one stage | Ephemeral black box; rerun whole jobs; no real local run |
| One control surface | Shared libs = golden paths, jobs = self-service, all in one place | Scattered across YAML / repos / marketplace |
| Control + cost | Self-hosted iron, no per-minute meter | Hosted minutes cost; self-host runners = the ops you fled, returning |
| Portability | Runs anywhere | Welded to GitHub — leave it, rewrite CI |

And the honest other half: nobody misses Groovy upkeep, plugin CVEs, the controller as a single point of failure, script approval, or a UI stuck in 2012. The migration is usually right. The nostalgia is for the *coherence*, not the maintenance.

## Why it fragments

An IDP has three planes. Jenkins, grown into a platform, **collapses all three into one tool** — which is why it feels coherent and why it ages badly:

- **Control** — catalog, ownership, self-service (Jenkins: folders + parameterized jobs)
- **Orchestration** — turn intent into config (Jenkins: Groovy shared libraries)
- **Execution** — actually build and deploy (Jenkins: agents)

GitHub Actions is excellent at **execution** and deliberately doesn't pretend to be the other two. So the moment your needs grow past "build and test," the control and orchestration that used to be one Groovy library are now spread across a dozen YAML files. That spread *is* the fragmentation.

## The fix: re-unify on top, don't retreat

Two wrong moves: mourn it in YAML, or rebuild Jenkins-as-IDP. The right move is to **put a thin control plane back on top of GHA** and let GHA stay the execution engine.

In order of effort:

1. **Use GHA's own structure first.** Org-level reusable + starter workflows, repository templates, `workflow_dispatch` for self-service, CODEOWNERS for ownership. Many teams stop here — and should, until the pain is real.
2. **Add a control plane when you've outgrown that.** [Port](https://www.port.io) is the shortest path: it auto-builds a catalog from your repos and triggers your existing workflows via `workflow_dispatch` — the execution layer doesn't move. Backstage is the same idea if you want to *build* it rather than buy it.
3. **Know what your suite does and doesn't do.** If you're on Atlassian, **Compass observes GHA (catalog + scorecards from build/deploy metrics) but cannot trigger it** — its only scaffolding feature was removed at the end of 2025. It's a governance layer, not a self-service portal over GHA.

The migration posture is strangler-fig: stand the control plane in front, move the highest-traffic self-service actions to it (backed by the GHA workflows you already have), and let "Jenkins-as-IDP" thinking fall away one job at a time.

## The one line to keep

What you miss isn't Jenkins. It's **being able to orchestrate everything from one place you can reason about.** GHA modernised execution and gave that coherence away. You get it back by adding a control plane on top — not by retreating to the monolith you already outgrew.

---

*Field note. The Jenkins-vs-GHA gaps are well-trodden; the Compass and control-plane specifics were checked against current vendor docs as of 2026-05-29.*
