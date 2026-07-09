---
title: "Five AI red flags that mean run away"
description: "Self-declared AI-first companies reveal themselves in single sentences. Five signals that separate real adoption from theater, one probe question each — and the closing question that fires past all five."
date: 2026-07-09 12:00:00
tags: [ai, hiring, interviews, engineering-culture]
asset_path: /assets/img/posts/2026-07-09-five-ai-red-flags
image:
  path: /assets/img/posts/2026-07-09-five-ai-red-flags/hero-og.png
  alt: "Five AI red flags that mean run away — interviewing at self-declared AI-first companies"
---

There is a strange asymmetry in tech interviews right now. Becoming "AI-first" takes a
company months of rebranding — the careers page, the job ads, the all-hands slides. Revealing
that the claim is decoration takes one sentence, usually said casually, by someone senior
enough that their mental model is the budget. As a platform engineer interviewing in the AI
era, I've started collecting those sentences.

This post is the collection: five sentence archetypes, what each one actually reveals, and a
probe question for each. None of them is a quote from a specific person at a specific
company — each is a *class* of utterance. You will hear some version of these sentences. The
question is what you do in the thirty seconds after.

## First, a filter for the filter

A red flag has to **discriminate**. If every company emits a signal, the signal carries no
information about any particular company.

The clearest example is the one I deliberately left off the list: *"We don't hire juniors
anymore — AI lets one person do the work of three."* It sounds ominous. It is also the
industry-wide default bet in 2026, made by good and bad companies alike. The base rate is
too high for it to separate anything. Whether a company makes that bet tells you nothing;
whether they can *defend* it does — and there's a question at the end of this post built
exactly for that.

Every flag below passes the discrimination test: real AI-first companies don't say these
sentences, and fake ones can't help saying them.

![The five red flags: the sentence you'll hear and what it actually reveals]({{ page.asset_path }}/five-red-flags.svg)
*The five flags on one card. Each row: the axis, the sentence archetype, the tell.*

## Flag 1 — The capability ceiling

> "LLMs are just another kind of Google."

The speaker's mental model of AI stops at retrieval: you ask, it answers, like search with
better grammar. There is no concept of agentic workflows, delegation, or verification loops
in that model — and if the speaker sets direction, there won't be in the org either. **An
organization's AI usage ceiling is its leadership's imagination ceiling.** You cannot out-adopt
the mental model of the person who approves the tooling budget.

This flag doesn't need its own probe. The closing question below is built to catch it —
someone whose model of AI is a better search box cannot answer it.

## Flag 2 — The process punishes the skill it claims to hire

> "You had AI write the take-home — how is that fair to paying customers?"

Read that sentence twice. The company says it hires for AI-era delivery; its hiring process
treats AI-assisted delivery as cheating. Notice the *moralizing* — a tooling question framed
as a guilt question. That framing doesn't stay in the interview. Downstream, engineers learn
that admitting AI usage is dangerous, so they hide it: shadow AI, no review norms, no
governance anyone can enforce, no team-level learning loop. The company gets all the risk of
AI adoption and none of the compounding.

**Probe:** *"What do your AI-usage norms look like for the engineering team?"* A real answer
names tools, review expectations, and boundaries. A moralizing answer — or a blank one —
is the flag confirming itself.

## Flag 3 — AI theater

> "We've adopted AI — everyone has Copilot."

The most common flag of the five. A license purchase is mistaken for a transformation.
Nobody redesigned a workflow; nobody measured before and after; there are no evals. The
tooling arrived and everything else stayed where it was — which means the company bought
autocomplete and issued a press release.

**Probe:** *"Can you name one workflow you redesigned because of AI — not accelerated,
redesigned?"* Accelerated means the old process runs faster. Redesigned means a step
disappeared, moved, or got a new owner. Companies doing the real thing answer this in
seconds, with specifics. Companies doing theater repeat the word "productivity."

## Flag 4 — Governance at the poles

> "Security banned LLMs company-wide."
>
> "No rules — use whatever you want."

Opposite sentences, same failure: nobody did the threat-model work of tiering — which data
class can go to which tool, and how output gets reviewed before it ships. A blanket ban and
a blanket shrug are both the absence of that work. The ban variant is the worse one at a
self-declared AI-first company: engineers use the tools covertly anyway, so the company
carries the data exposure *and* believes it doesn't — the worst of both.

**Probe:** the same norms question from Flag 2 does double duty here. Listen specifically for
tiering: different rules for different data classes is the sound of someone having actually
thought. One rule for everything — whichever rule — is the sound of nobody having thought.

## Flag 5 — Velocity without verification

> "Almost all our code is AI-written now — we ship ten times faster."

The mirror image of Flag 2. This company embraced generation and skipped verification: no
story for who reviews AI output, no eval coverage, no rollback discipline — just a speed
number, said proudly. Generation speed is the *easy* half of AI adoption; the hard half is
catching what the model got wrong before customers do. If they've solved the hard half, they
brag about that. If they brag about the easy half, they haven't.

And be clear about what the role is at such a company: you are being hired as the on-call
scapegoat for an unaudited codebase.

**Probe:** *"When was the last time AI output caused an incident here, and how did you catch
it?"* A company using AI seriously has an answer — a real incident, a catch mechanism, a fix.
"We haven't had one" means they're either not using it seriously or not looking. Both bad.

## The closer: "So what do you need humans for?"

Deploy the probes in the "any questions for us?" slot — it turns passive listening into
active elicitation, and the *manner* of answering is itself data. Save this one for last,
because it fires past all five flags at once:

> "So what do you need humans for?"

A real AI-first company has a concrete answer, because it has actually drawn the line:
AI does X, humans own Y — judgment, verification, accountability, cross-system taste.
The division of labor is specific and the human side is load-bearing.

A fake one says some version of: *"Someone has to supervise the AI until it gets better."*

Sit with that answer. If the human role is "supervise until it gets better," then you are not
being hired as an engineer. You are a placeholder for the model's next version — and they
already think of you that way. The interview just told you, politely, what the org chart will
do less politely later.

![The meta-probe: two answer branches, one verdict each]({{ page.asset_path }}/meta-probe.svg)
*One question, two branches. The concrete-human-role answer earns the next round; the
placeholder answer is the exit cue.*

---

*A closing caveat on weight: one sentence from one interviewer is a signal, not a verdict —
people misspeak, and one skeptic doesn't define a company. But unlike most of what you're
trying to learn in an interview, these five are directly observable from the candidate's
chair, at zero cost. The company is interviewing you; there's no reason to stop interviewing
them.*
