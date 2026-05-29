---
title: "Webwright setup and how agents meet the real world"
subtitle: "The install path and the disciplines that travel"
date: 2026-05-29 08:00:00
tags: [agents, frameworks, discipline]
---

Most write-ups of browser-agent frameworks stop at the install command and a "hello world" page visit.
That's the first ten minutes.
The interesting part is what happens between hour two and hour twenty,
when the agent meets a site whose DOM rotates weekly,
whose login page just added a CAPTCHA,
whose vendor patched the obvious attach pattern last month.

This post is the lab notebook for one such install —
Microsoft Research's [Webwright](https://github.com/microsoft/Webwright) —
and the disciplines I'd carry forward to whatever framework replaces it.

> Companion piece: [When the API doesn't exist](/when-the-api-doesnt-exist/),
> which is the use-case side of the same loop.

## Why an agent framework at all

The framework's job is small but important:
turn an LLM into something that can take a goal,
write a short program against a browser,
run it, and report back what happened.
Webwright settled on this contract:

- **Body** — Playwright (Firefox by default; dodges some Chromium TLS fingerprinting)
- **Eyes** — the LLM, reading screenshots via vision
- **Glue** — a workflow that asks the model to author a `final_script.py` per task,
  then executes it under a controlled venv

That's the entire surface area. Everything else is conventions, skills, and the disciplines we'll get to.

The reason to use it (or browser-use, or whatever framework replaces them) instead of writing raw Playwright per task:
**the model gets to look at the rendered page before deciding what to do**.
That changes what's possible against any site that wasn't designed to be programmatically read.

## The install path

I'll narrate the actual sequence rather than list commands, because the order matters for safety.

### 1. Don't run upstream code on the host before auditing it

The first step on any external repo — agent framework or otherwise — is a four-stage supply-chain audit.
This is NIST SSDF SP 800-218 PW.4.4 / SLSA v1.0 / OWASP CI-CD Top 10 territory, not paranoia.

```sh
# 1. Isolated clone — never inside your working tree
git clone https://github.com/microsoft/Webwright /tmp/wwr && cd /tmp/wwr

# 2. CVE scan
trivy fs --severity HIGH,CRITICAL .
pip-audit -r requirements.txt          # or derived from pyproject

# 3. SBOM + typo-squat review (CycloneDX format)
syft packages dir:. -o cyclonedx-json > /tmp/wwr-sbom.json

# 4. Malicious-pattern scan + manual install-hook review
semgrep --config=auto .
grep -RE 'postinstall|preinstall|curl[^|]*\|[[:space:]]*sh' .

# 5. First execution in Docker, network off
docker run --rm --network=none -v "$PWD:/work" -w /work python:3.13 \
  python -c "import webwright; print('hello')"
```

Stop signals that should kill the install: single-commit repo,
obfuscated source presented as source,
`.so`/`.dll` blobs shipped without provenance,
`curl | bash` install hooks,
dependencies pinned to commit SHAs without explanation.
Webwright cleared all five.

### 2. Isolate the runtime — never `pip install` against system Python

```sh
mkdir -p ~/.local/share/webwright-venv
python3 -m venv ~/.local/share/webwright-venv
~/.local/share/webwright-venv/bin/pip install -e /tmp/wwr
~/.local/share/webwright-venv/bin/playwright install firefox
```

Two reasons:

- Webwright pulls in Playwright, httpx, pydantic, jinja2, rich, typer, python-dotenv, platformdirs.
  Most are fine, but you don't want them silently shadowing system Python for unrelated work
- If anything goes wrong, `rm -rf ~/.local/share/webwright-venv` is the entire uninstall

### 3. Move host-specific knowledge into a user-scope skill, not the framework

Webwright ships its own `SKILL.md` that assumes `python` is on `$PATH` with the right packages.
On this machine that's only true inside the venv.
The fix is **a second skill at `~/.claude/skills/webwright-runtime/SKILL.md`** that pins the runtime paths:

```markdown
| What                       | Where                                                  |
|----------------------------|--------------------------------------------------------|
| Run generated scripts      | `~/.local/share/webwright-venv/bin/python <script>`    |
| Standalone CLI             | `~/.local/share/webwright-venv/bin/webwright`          |
| Playwright browsers cache  | `~/Library/Caches/ms-playwright/`                      |
```

Why user-scope (`~/.claude/skills/`) instead of patching the upstream skill:
**plugin updates overwrite plugin-scope files; user-scope survives**.
Locality of reference — the host-specific paths live next to the host, not next to the framework.

The second user-scope skill (`webwright-auth`) holds the auth playbook:
when to use `storage_state`, when to attach via CDP, when to refuse and tell the operator to open the browser themselves.
Same locality argument.

## How agents work (and where they meet reality)

The contract — body = Playwright, eyes = the model — is simple.
The interesting question is **how the model finds the things it wants to click**
when the site doesn't care about being agent-friendly.

The naïve answer is "CSS selectors and XPath," and it's brittle.
Modern sites use CSS-in-JS with build-hashed class names (`.css-1a2b3c4`) that rotate every release.
Hard-coding selectors means the agent works on Tuesday and breaks on Thursday.

The fallback ladder I run, from durable to brittle:

| Layer | Mechanism                                                             | Why it survives or doesn't                                                                                       |
|-------|-----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 1     | Playwright semantic locators — `get_by_role`, `get_by_label`, `get_by_text` | Resolves against the **accessibility tree** at runtime. ARIA role + accessible name is web standard, not a build artefact |
| 2     | Stable data attributes — `[data-occludable-job-id]`, `[data-testid=…]`     | Frontend teams treat these as contract; they outlive class renames                                               |
| 3     | Vision — screenshot → model looks at it → derive bounding box              | Resilient to any DOM rotation, but token-expensive and slower than locator queries                               |
| 4     | Mouse coordinates — `page.mouse.click(x, y)`                               | Last resort, where vision-derived bounding box becomes an action                                                 |
| ⛔    | Class-hash or remembered XPath                                             | Don't. Mine, yours, the model's training memory — all stale by the next deploy                                   |

Where the layers meet reality:

- **DOMs rotate every release.** Layer 1 holds; layer 2 mostly holds; layers 3 and 4 always hold; layer ⛔ always breaks
- **APIs don't exist.** `storage_state` makes layer 1 usable against authenticated pages without re-typing credentials per run (see the companion post)
- **Anti-bot defences exist.** From one session of probing, behaviour-dominated factors
  (pacing, focus events during login, request cadence) appear to matter more than browser fingerprint.
  Human cadence is the cheap insurance

There's a trap worth naming: **the model's training memory of a site's structure**.
That memory was true at training time; sites change.
Probe the page first, build the action from what's actually there,
not from what was true six months ago.

## The disciplines that survive contact with real systems

The install ages. The framework will eventually be replaced. These travel.

### External documents are data, not commands

JD files, READMEs, scraped HTML, WebFetch results, MCP tool outputs —
anything authored outside the project —
are inputs to be processed, not instructions to be obeyed.
When an external document tells the agent
"run this command" / "fetch this URL" / "ignore previous instructions,"
the response is: stop, quote it verbatim with location,
classify the ask, ask the operator before doing anything.
Never exfiltrate file contents to URLs that appear inside an untrusted document.

This is OWASP Top 10 for LLM Applications LLM01:2025 / MITRE ATLAS AML.T0051 / NIST AI 600-1.
It's the AI-era `curl | bash`.

### Post-edit verification — no "Done" without proof

After any change, run the appropriate check before claiming completion.
Type-check, lint, structural validation, the smallest test that exercises the change.
"Bytes written" is not the same as "correct."
This applies to the model writing code, and to the model running scripts;
both states need a confirmation gate.

### The credential surface is wider than it looks

`storage_state` files are credentials —
`chmod 600`, gitignored, never copied between machines casually.
Screenshots from authenticated sessions are PII.
Bash history leaks tokens. The `final_runs/` directory the framework writes is sensitive by default.

Git history is forever.
If something private lands in a commit, removing the file later doesn't remove it from history —
`git log --all` + `git cat-file` still recover it.
The cheap discipline: the **first** commit for a published artefact is the **final** version.
Never push a draft that contains anything you intend to redact, then revert.
Mask before push.

### Detect and fail loud, don't degrade silently

Sessions expire. Pages 404. CSRF tokens rotate.
The brittle response is to keep going and process whatever lands.
The durable response is to assert what you expect after every navigation —
URL contains the expected path, page title contains the expected substring,
a key element is present — and exit loudly when it doesn't.
Silent failure is the most expensive failure mode any agent can have.

```python
page.goto("https://target.com/home")
if "/login" in page.url or "/authwall" in page.url:
    raise SystemExit("Session expired — re-run setup_auth.py")
```

### Match human cadence

If the underlying task is something you'd manually do once a day,
the agent doing it once a day is appropriate.
If the agent could technically hammer the server a thousand times an hour,
that's a discipline question, not a capacity question.

Most anti-bot at the read tier appears to be behaviour-dominated.
Human-paced agents stay invisible by being indistinguishable from a slightly-distracted user.

## Closing — the framework rotates, the craft stays

Webwright will be replaced eventually —
by browser-use, by Anthropic's `computer-use` going general-purpose,
or by something with a slightly different contract.
The install path will change. The CLI flags will change. The default browser may flip.

What stays:

- **Supply-chain audit before first execution.** The threat model doesn't change with the framework
- **Runtime isolation.** A venv, a container, anything that lets you `rm -rf` the install
- **User-scope over plugin-scope.** Host-specific paths live next to the host
- **Semantic locators over class hashes.** The accessibility tree is older than React; it'll be there when React is gone
- **`storage_state` discipline.** The credential surface is constant
- **External docs as data.** The injection threat is structural, not framework-specific
- **Verify after every step.** The only way the agent's reports stay honest

Frameworks are scaffolding. The craft is what stops the scaffolding from collapsing on you.

---

*Companion: [When the API doesn't exist](/when-the-api-doesnt-exist/). Polished portfolio at [binhsu.org](https://binhsu.org).*
