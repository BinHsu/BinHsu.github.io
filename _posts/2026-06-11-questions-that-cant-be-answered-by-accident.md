---
title: "Questions that can't be answered by accident"
subtitle: "Years prove work happened near someone. Six probes for the engine that did the thinking — and where each one comes from."
date: 2026-06-11 11:00:00
tags: [hiring, interviews, engineering-culture]
asset_path: /assets/img/posts/2026-06-11-questions-that-cant-be-answered-by-accident
thumbnail-img: /assets/img/posts/2026-06-11-questions-that-cant-be-answered-by-accident/artifact-vs-generator.png
share-img: /assets/img/posts/2026-06-11-questions-that-cant-be-answered-by-accident/artifact-vs-generator.png
---

A shower thought, after a season of sitting on the candidate side of a lot of interviews: what should an interviewer ask to find the engineer who is excellent at *whatever* they touch — and to filter out the one who merely stood near the work, or whose "senior" is tenure arithmetic?

![A CV proves artifacts — years, titles, tool lists. The engineer you want owns the generator that produced them. Interview the generator.]({{ page.asset_path }}/artifact-vs-generator.png)

Here is the frame I keep landing on. An engineer who "happened to do it" owns an **artifact**: the project existed, they were on it, the bullet point is true. The engineer you actually want owns the **generator** — the internal model that produced the artifact, and would produce a different, equally sound artifact under different constraints. A CV can only prove artifacts. Years and titles are artifact-proofs too. So the interview has one job the CV can't do: **probe the generator**.

Which gives the design rule for every question you ask:

**Ask questions whose answers can't be owned by accident.**

An artifact can be acquired by being present. A generator can't. Every probe below works because its answer is only derivable, never inherited.

## The six probes

### 1. The why-ladder

Take their proudest project and descend the why-chain: why this design? What else did you consider? What constraint decided it? *What would have to change for your decision to be wrong?*

The person who happened to be there stalls at level two — they can describe *what*, not *why-this-not-that*, because the decision either wasn't theirs or never passed through a model. The person with the generator can name the rejected alternatives and the condition that would flip the choice. That last question is the sharpest rung: people who own decisions know their decisions' expiry conditions.

### 2. Perturb their own story

Change one constraint inside *their* system: "Same design — now the load is 10×. Where does it break first?" "Same pipeline — the compliance requirement is gone. What do you delete?"

This is the cleanest discriminator I know between *did it* and *understands it*. A memorized experience does not survive perturbation; a model does. And because the scenario is their own system, the escape hatch — "I haven't worked with that" — is closed. They built it. They just never had to re-derive it.

### 3. The unfamiliar-domain question

Deliberately pose a problem from a domain they have *not* worked in — and grade the approach, not the answer. Do they ask the right clarifying questions? Do they map the new problem onto invariants they already own — backpressure, idempotency, failure domains, blast radius?

"Excellent at whatever they touch" literally means *performs like a veteran in a domain they haven't visited*. So test the new-domain behavior directly instead of inferring it from old-domain recall. The candidate's questions carry more signal than their answers here.

### 4. Drop one layer down

Pick the layer *below* their daily work and ask why it exists. Why do private IP addresses exist? Why does TCP bother with a handshake? Why does your build tool need a lockfile?

The understander holds the layer below, because their model needs a foundation. The experiencer holds only the layer they touched. This is the oldest probe on the list — it is exactly what a PhD oral examination does — and it works on engineers of every seniority. (I'll vouch for its effectiveness from the receiving end: being dropped a layer below your fluent zone is instantly, honestly revealing. The gap it finds is fixable knowledge — which is precisely why it's a fair question.)

### 5. Teach-back

"Explain the hardest thing you've built to a junior engineer."

Understanding compresses. Experience-without-understanding pads — jargon, length, or both. Thirty seconds in, you know which one you're hearing.

### 6. Failure archaeology — the upgraded version

The naive version ("tell me about a failure") is so common it's been gamed into uselessness; every candidate carries a rehearsed humble-brag. The version that still works asks for the *mechanism*: "Tell me about a decision you got wrong. **What was the model in your head at the time, and what evidence corrected it?**"

A tenure-senior often has only external attribution ("the project slipped"). Someone with a real generator can describe the wrong model they held and the moment the world disagreed — and the ability to narrate your own model-update is the strongest evidence that a model exists at all.

## Where these come from — textbook, field, or new

I went looking for the provenance, because "is this established or am I inventing it?" matters for how much you trust it.

**Textbook-rooted (the foundations are decades old):**

- The why-ladder is **Behavioral Event Interviewing** — McClelland's 1973 paper is the founding document of competence-over-credentials assessment — crossed with Toyota's **5 Whys** and the chronological deep-dive that Topgrading industrialized. Amazon's Bar Raiser follow-up culture is this, operationalized.
- Drop-one-layer-down is the **oral examination** tradition, centuries old. Feynman's "knowing the name of something is not knowing something" is its philosophy.
- Teach-back is a literally named clinical practice in medicine (verifying patient comprehension), repurposed.
- Failure questions are behavioral-interview canon — only the model-update upgrade is newer.
- The statistical backbone is **Schmidt & Hunter's 1998 meta-analysis**: work samples and structured interviews top the predictive-validity table; years of experience sit near the bottom. Google's internal replication (Laszlo Bock, *Work Rules!*, 2015) added the famous corollary: brainteasers predict nothing. The reason "senior by tenure" passes most funnels is that most funnels still measure the lowest-validity signal on the chart.

**Field-derived (practitioner craft, no citation to give):**

- Perturbation *on the candidate's own story*. "Now scale it 10×" is standard system-design-interview craft; pointing it at their lived system instead of a hypothetical is the upgrade — it closes the "haven't done that" exit.
- The model-update version of the failure question.
- The artifact/generator frame itself is my synthesis — I haven't seen it named elsewhere, though everything underneath it is older than I am.

**New since ~2024 (the AI era actually changed this):**

- **AI-allowed interviews** — let the candidate use the assistant, and watch prompting quality, verification discipline, and judgment about what to trust. The question "how deep have you gone into AI tooling?" followed by three live why-rungs is a 2025-era probe, and it's a generator test: guardrail thinking can't be owned by having a Copilot license.
- **Comprehension over production** — "review this code / critique this design" instead of "write this code," because production is what AI commoditized and comprehension isn't.
- **Anti-cheat by latency** — with real-time AI whispering tools in remote interviews, the counter turns out to be the probes above: a why-ladder at conversation speed outruns a model relaying through a screen. Follow-up velocity is now itself a filter.

There's an irony worth saying out loud: AI cheating is killing the artifact-style interview — the take-home, the trivia checklist, the rehearsed STAR monologue. What survives the arms race is exactly the generator test. **AI is forcing interviews to get better.**

## If you only keep two

Keep the **perturbation** and the **why-ladder**. They cost nothing to prepare, run inside the candidate's own material, and are the two hardest to fake — with or without an AI in the room.

---

*Caveat on vantage point: I'm writing this from the candidate's chair after many rounds across many companies this season — the classification of what's textbook versus field-craft is my read of the literature, not a survey. The McClelland and Schmidt & Hunter papers and* Work Rules! *are the places to check my homework.*

## Further reading

- David McClelland (1973), *Testing for Competence Rather Than "Intelligence"* — the founding argument for probing what people can do over what credentials they hold.
- Frank Schmidt & John Hunter (1998), *The Validity and Utility of Selection Methods in Personnel Psychology* — the meta-analysis behind "work samples and structured interviews win; tenure barely predicts."
- Laszlo Bock (2015), *Work Rules!* — Google's internal hiring data; the death certificate for brainteasers.
- Bradford Smart, *Topgrading* — the chronological deep-dive interview, ancestor of the why-ladder.
- Taiichi Ohno's **5 Whys** — root-cause descent, the same ladder pointed at incidents instead of candidates.
- The **teach-back method** (health-literacy practice) — comprehension verification by re-explanation.
