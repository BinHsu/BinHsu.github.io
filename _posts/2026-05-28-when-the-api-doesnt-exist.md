---
title: "When the API doesn't exist"
subtitle: "A browser agent for the job search loop"
date: 2026-05-28
tags: [agents, automation, playwright]
asset_path: /assets/img/posts/2026-05-28-linkedin-fieldtest
---

70 % of the things I want to automate at home don't expose a public API.
Job boards. Banking. Internal tools at every company I've worked.
The German government portal where I track my Anmeldung.
Most SaaS dashboards if you don't pay for the enterprise tier.

For one of those — my own job search — I built an agent that does the morning scan for me.
It cut roughly 90 minutes a day to about 5 minutes of review over coffee.
Here is what worked, where the boundaries actually are, and what I think this pattern is good for.

> Companion piece: [Webwright setup and how agents meet the real world](/webwright-setup-and-how-agents-meet-the-real-world/),
> which is the install + craft side of the same loop.

## The pain, exactly

I look at five surfaces every day: my personalised recommendations on a major professional network,
two German job-board aggregators, two company career pages I'm following.
For each surface:

- Pull new postings since yesterday
- Deduplicate against ones I've already applied to (I keep a tracking README in a local Git repo)
- Extract structured fields from each posting: title, company, location, key requirements, posted date
- For strong matches, tailor a cover-letter hook from a template

Each pass is mechanical but tedious — maybe 90 minutes,
often interrupted by the "wait, did I apply here?" check.
I wanted an agent to do it overnight.

## Why an agent with eyes (and not yet another scraper)

The phrase "web scraping" comes from a 2005 mental model: write XPath, parse HTML, store in CSV.
That model dies the moment a site rerenders with React, gates content behind auth, or rotates its DOM.
Most modern surfaces fall into at least one of those categories.

The pattern that emerged across several labs in 2024–2025:

- **Body = Playwright** (or another browser controller)
- **Eyes = an LLM that reads screenshots and decides what to do next**
- **The orchestration layer asks the LLM to author a small Python script per task, then runs it**

Microsoft Research published [Webwright](https://github.com/microsoft/Webwright) as a reference implementation.
[browser-use](https://github.com/browser-use/browser-use) and Anthropic's `computer-use` family take similar shapes.
The framing the field settled on: **code-as-action** — the agent doesn't call structured tools,
it writes a short program that uses Playwright and runs it.

The technique isn't new; the novelty is **judgement at the right level**.
The LLM doesn't have to follow a brittle DOM rule because it can look at the rendered page and decide.

For tasks like "scan five job boards every morning and report what's new," this fits exactly.

## The minimal stack

The smallest thing that worked, end to end:

1. **Playwright Firefox** — one-time `pip install playwright && playwright install firefox`
2. **A dedicated Python venv** — keeps the agent's runtime isolated from system Python
3. **`storage_state`** — Playwright's mechanism for saving cookies + localStorage to JSON,
   then loading it next session. Log in once manually in a headed browser, save the auth state,
   reuse it on every subsequent automated run
4. **No credentials in code** — the password is never typed by a script.
   The setup runs `headless=False`, the browser opens, I type the password by hand,
   the script captures the resulting state

That's the entire infrastructure footprint.

```python
# setup_auth.py  (run once, headed, you log in manually)
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.firefox.launch(headless=False)
    ctx = browser.new_context(
        viewport={"width": 1280, "height": 900},
        locale="en-US",
        timezone_id="Europe/Berlin",
    )
    page = ctx.new_page()
    page.goto("https://<your-target-site>/login")
    input("Log in in the browser, press Enter here when done… ")
    ctx.storage_state(path="auth.json")   # chmod 600, gitignore
    browser.close()
```

```python
# every subsequent run
ctx = browser.new_context(storage_state="auth.json")
# pages from this context are already authenticated
```

> *Disclaimer: the code in this article is illustrative. It depicts the shape, not a runnable package.
> If you build something like this yourself, do it against a service you legitimately own a session on,
> at a natural human pace, and respect that service's Terms of Service.*

## Session persistence — the load-bearing trick

The piece that makes the whole loop practical, rather than "log in fresh on every run
and get flagged for it within a week," is Playwright's **`storage_state`**.

`BrowserContext.storage_state(path=...)` serialises the context's cookies + `localStorage`
for every origin you've visited into a JSON file.
Next session, `new_context(storage_state=path)` deserialises them back into a fresh context.
The server sees the same cookies it issued during your manual login — it can't tell
whether your browser is the same physical process or a different one a week later.

The lifecycle in three states:

1. **One-time setup** — headed browser, you log in by hand,
   the script calls `ctx.storage_state(path="auth.json")` → JSON lands on disk
2. **Every subsequent run** — `new_context(storage_state="auth.json")` → pages start authenticated
3. **When the session expires** — the target site starts redirecting you to `/login`;
   detect that, surface a re-setup prompt, log in again. Sessions typically survive a few weeks
   if the site lets you keep "Remember me" checked, days otherwise

What's actually in the file (abridged):

```json
{
  "cookies": [
    {
      "name": "session_id",
      "value": "…",
      "domain": ".target.com",
      "path": "/",
      "expires": 1746000000,
      "httpOnly": true,
      "secure": true,
      "sameSite": "None"
    }
  ],
  "origins": [
    {
      "origin": "https://target.com",
      "localStorage": [{ "name": "…", "value": "…" }]
    }
  ]
}
```

Two operational notes that matter more than they look:

- **The file is a credential.** `chmod 600`, gitignore the path,
  and never copy it across machines without thinking about what you're doing.
  A leaked `auth.json` is a leaked session — equivalent to handing your password to whoever picks it up
- **Detect expiry early, fail loud.** The first thing every run does after `new_context` is load a
  cheap authenticated page (the home feed, an account-settings URL) and check whether the URL
  redirected to `/login` or `/authwall`. If it did, exit with a "re-run setup_auth.py" message
  rather than silently scraping a logged-out shell

```python
# every run, immediately after loading state
page.goto("https://target.com/home")
if "/login" in page.url or "/authwall" in page.url:
    raise SystemExit("Session expired — re-run setup_auth.py")
```

That's the entire session-management story. Two scripts, one JSON file, three states.

## Probing what the agent can actually see

Before automating the loop, I wanted to know:
when I load my logged-in target sites via a fresh Playwright Firefox carrying my saved state,
does the server treat me as me, or does it serve a degraded view?

I ran a five-step probe ladder of increasing depth.

| Probe | Endpoint type | What I checked |
|---|---|---|
| 1 | Personal feed | Does the page render as my real authenticated home? |
| 2 | Own profile page | Do owner-only edit affordances appear? |
| 3 | Search results | Does the lazy-loaded list populate? |
| 4 | Personalised recommendations | Does the algorithm serve content tailored to my profile? |
| 5 | Direct-messaging inbox | Does the WebSocket-driven UI hydrate correctly? |

All five passed. The server delivered the same content the agent would have shown me in my everyday browser.

![Probe 1 — personal feed]({{ page.asset_path }}/probe1-feed-masked.png)
*Probe 1 — personal feed. All identifying content pixelated; the three-column scaffold remains visible
so you can see the layout populated.*

![Probe 3 — search results]({{ page.asset_path }}/probe3-jobs-search-masked.png)
*Probe 3 — search-results page. Master-detail layout, the full filter row (date / salary / title / workplace),
and the authenticated cookie banner all rendered.*

I had expected at least one of these to challenge me with a CAPTCHA or "verify it's you" prompt.
None did. The hypothesis I formed:

> **At the read tier, anti-bot is behaviour-dominated, not fingerprint-dominated.**

Browser fingerprint (Firefox vs Chrome, headless vs headed, automation flags) seems to matter less
than the public lore had led me to believe. What seems to matter more: typing rhythm,
focus events during login (which my session never had — I clicked and typed by hand),
the cadence between requests, and the *kind* of action being performed.

This is a working hypothesis, not a theorem. n = 1 session, one IP, one account.
I state it because it's actionable: if you want a read-tier agent to stay invisible,
**don't pace it like a robot**.

## What this doesn't promise

This experiment shows that **one logged-in account, on one browser, on one IP, behaving naturally,
can be driven by an agent to read its own data**. It does **not** show:

- That you can do this at scale (rotating accounts, IPs, residential proxies — different game, different ethics)
- That write actions have the same threshold as reads (they appear calibrated more strictly — separate report)
- That the same approach survives across weeks of sustained use (sample size is one session)
- That sites you don't already own a session on can be accessed this way
  (that's still scraping, with all the legal baggage that carries)

Treat the finding as: **automating your own logged-in account to do what you'd manually do yourself,
at a natural human pace, is a tractable engineering problem**.
Whether your target service is fine with it is a question of their ToS and your relationship with them.

## The mental model that helped me

I'd been stuck for a while thinking about web automation through the "scraping" frame.
The frame that finally clicked:

> **A browser agent is a Slack bot for the web.**

You'd use a Slack bot to summarise channels while you sleep.
It logs in as you (or as a bot user you control), reads what's there,
performs tasks you'd otherwise do manually, and reports back.
Same pattern. The only difference is that web surfaces are messier than Slack's API,
so the bot needs eyes.

Once I framed it that way, several decisions got obvious:

- I shouldn't try to scale beyond what I'd do manually
- I should use my own logged-in session, not extracted cookies from another browser
- I should match human cadence — there's no reason to hammer the server faster than I would in person
- I should keep the surface narrow: this agent does one thing, on a small set of sites I already use daily

## What it does for me, now

The current loop:

1. A cron-style trigger at 7:30 in the morning
2. A fresh Playwright context carries my `storage_state` in
3. The agent visits each target surface and captures structured data
4. It compares results against my local applications tracker (a Markdown table in a Git repo)
5. It writes a daily digest: new postings + dedup status + a match score against my profile
6. The digest lands at `~/Documents/job-digest/<date>.md` for me to scan over coffee

90 minutes a day reduced to a 5-minute review.
The agent does not apply for anything — applying is a judgement layer I keep manual.
It scouts.

## Where I'd take this next

Three obvious extensions, in order of effort:

- **Cover-letter tailoring from JD structure** —
  feed the extracted JD into a prompt that drafts a one-paragraph cover-letter hook against my profile.
  I currently do this by hand for the top three matches each day.
- **Multi-site adapter pattern** —
  abstract over the per-site DOM differences so adding a new job board is a small adapter,
  not a fresh probe ladder.
- **Push to a personal Slack** —
  instead of a Markdown digest, ping me where I already am.

Each is incremental on the same minimal stack.
The expensive work was the probe ladder — once that established what's possible,
everything else is straightforward Python.

---

*This site is the lab side of my work. The polished portfolio lives at [binhsu.org](https://binhsu.org).*
