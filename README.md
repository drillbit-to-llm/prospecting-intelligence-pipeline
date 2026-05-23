# Prospecting Intelligence Pipeline

Multi-agent AI system for enterprise sales development.  
Built without a CS degree, by a petroleum engineer.

---

## What it does

Two coordinated pipelines. The first classifies and
researches any list of companies to identify recurring
revenue opportunities. The second takes a single named
account and produces a personalized, validated,
fully-written outreach sequence for every qualified
buyer — researched, debated, voice-checked, and
accuracy-gated before it ships.

---

## Pipeline 1 — Classification & Research

Accepts any CSV of companies (`name` + `url`).  
Runs unattended: `node pipeline.js mylist.csv`

Six agents run in sequence per company:

**Classifier** (`classifier.js`)  
Territory classification. Uses Claude Sonnet + Playwright
browser automation for DTC verification of subsidiary
brands. Handles parent company expansion into individual
brands. Skips non-DTC subsidiaries automatically.

**Research Agent** (`research-agent.js`)  
Revenue, ecommerce revenue, and store count via
web search.

**Subscription Analyst** (`subscription-agent.js`)  
Browser automation detects subscription UI signals and
platform signatures on product pages, subscription pages,
and shop subdomains. Identifies Ordergroove, Recharge,
Skio, Bold, Stay.ai, Recurly, Stripe, Chargebee, Yotpo,
Smartrr, Loop, Appstle, and others.

**Cart Agent** (`cart-agent.js`)  
Navigates product pages, adds items to cart, detects
subscription signals that only appear post-add-to-cart.
Handles cookie banners, age gates, variant selectors,
cart drawers. Runs conditionally — only when needed.

**QA Agent** (`qa-agent.js`)  
Merges output from all four agents. Runs deterministic
consistency checks, computes quality score (0–100),
generates sales relevance summary. Requires 2+ agent
sources to cross-validate before grading.

**Intent Agent** (`intent-agent.js`)  
Strategic intelligence via web search: earnings calls,
executive interviews, press releases, job postings.
Returns alignment score, specific sourced quotes,
outreach hook, and recommended persona.

**Pipeline features:** checkpoint saves every 25
companies so interrupted runs resume where they left off.
Rate-limit-aware retry (15s backoff on 429). Parent
company expansion into subsidiary brands. 500ms between
API calls.

**Has run on:** NRF Top 100 retailers (77 companies
scored), Shoptalk 2026 attendee list (2,960 companies
in queue), demo and test batches.

---

## Pipeline 2 — Per-Account Sequence Generation

Takes a company name. Outputs one `.docx` per qualified
buyer — 12 tabs each: a persona cover page plus an
11-touch sequence across 12 days (4 emails, 2 LinkedIn
touches, 4 call scripts with live-answer and voicemail
variants, 1 breakup email). Also generates an
account-level persona overview summary `.docx`.

Runs across two execution environments. Stages 0–1
require live MCP tool access inside a Claude Code
session. Stages 2–7 run autonomously from terminal.

### Claude Code — Stages 0–1

**Stage 0 — Engagement Check**
(`engagement-check-agent.js`)

Three required sub-steps before any outreach runs:

*Step A (terminal):* Standalone script queries the local
Gong MCP server's SQLite store. Classifies the account
as cold-open, warm re-engagement, or halt entirely.
Sets `mcp_steps_pending: true` — a gate that blocks
Stage 3 until Steps B and C complete. Also auto-loads
intent signals from disk if a signals file exists for
this account (Step D, non-blocking).

*Step B (Claude Code MCP):* Three sources that require
live MCP connectors only available inside a Claude Code
session:
- Gmail: 18-month two-way thread search, filtered for
  substantive engagement (calls, demos, proposals).
  Noise excluded: unsubscribe, noreply, event platforms.
- Slack: four searches run and merged — workspace-wide
  signal search, `#p-[account]` prospect channel
  (archived = prior sales cycle, warm re-engagement),
  `#c-[account]` customer channel (active = halt
  pipeline immediately and route to CS team), and
  `#you-got-mail` for prior sequence engagement.
  Always uses `slack_search_public_and_private` — never
  `slack_search_public`, which silently misses private
  channels.
- Google Drive: searches for sales-relevant files
  (proposals, SOWs, decks, contracts, briefs).

*Step C (merge):* MCP findings merged into
`pipeline_context.json` alongside the Gong data.
`mcp_steps_pending` set to `false`. Stage 3 unblocked.

No LLM in Stage 0. Pure deterministic synthesis.

**Stage 1 — Persona Sourcing** (`persona-agent.js`)

Six-step dual-source workflow:
1. Apollo MCP: two searches (exec layer + manager layer)
2. Apollo bulk enrichment: required, provides
   `employment_history` for career background scoring
3. Lusha MCP: two searches (seniority + title-targeted)
4. Lusha enrichment: net-new contacts only
5. Deduplication: Apollo records take precedence
6. Write merged candidates file

Candidates then qualified against a weighted scoring
rubric: title match 30%, seniority 25%, department 20%,
career background 15%, priority signals 10%. Concurrency-3
worker pool. UK/EU accounts skew Lusha-primary (Apollo
returns 0–5; Lusha returns 15–25).

Outputs ranked qualified personas with role
classification (champion / technical blocker / economic
buyer), persona brief, outreach hook, and typed signals.
Model: `claude-sonnet-4-6` + web search.

### Terminal — Stages 2–7

```
node run-pipeline.js --account "Company Name"
node run-pipeline.js --accounts "Co1,Co2,Co3"
node run-pipeline.js --pending
node run-pipeline.js --pending --from "Stage 4"
```

**Runner capabilities:**  
`--pending` auto-discovers accounts with incomplete
stages and queues them sorted by staleness and
completion count. `--accounts` runs multiple accounts
with a configurable concurrency pool (each account's
stages remain sequential). Staleness detection flags
accounts where Stage 0 or 1 ran more recently than
Stage 2. Fuzzy account resolution via
`account-resolver.js` prevents duplicate folders from
name variants.

**Stage 2 — Research Synthesis**
(`research-synthesis-agent.js`)

Five ordered steps:
- A: Extract prior-engagement directives from
  `pipeline_context.json` — establishes cold vs. warm
  framing for all downstream stages
- B: Fuzzy-match existing account briefs in
  `universe-research/briefs/`
- C: Targeted web enrichment — prior-engagement
  questions run first, then standard gap fills
- D: Program policy and ToS intelligence pass —
  cancellation mechanics, pause/skip rules, friction
  signals, tier mechanics
- E: Generate `company_brief.md` with no additional
  API calls. `stripTier3Urls()` applied after every
  web pass.

Models: `claude-haiku-4-5` + `claude-sonnet-4-6`.

**Stage 3 — Debate Agent** (`debate-agent-v2.js`)

Four-agent adversarial debate per qualified persona.
Three execution phases:

*Phase 1 (true parallel):* Advocate and Challenger each
propose an outreach angle from different right-to-win
categories. Category diversity enforced — they cannot
propose the same category.

*Phase 2:* Devil's Advocate stress-tests both angles.

*Phase 3:* Arbitrator picks winner and runner-up with
persona-role-type-aware evaluation — different criteria
applied for champions vs. economic buyers vs. technical
blockers.

No web search — all reasoning operates on the dossier
only.

Model assignments: Advocate = `claude-haiku-4-5`
(1,024 tokens), Challenger = `claude-haiku-4-5`
(1,024 tokens), Devil's Advocate = `claude-haiku-4-5`
(1,024 tokens), Arbitrator = `claude-sonnet-4-6`
(2,048 tokens).

Outputs: winning angle, runner-up angle, per-angle
scores, arbitration rationale. Cost: ~$0.06–0.10 per
company.

In progress: right-to-win analysis across 36 accounts
and 378 Gong calls feeds directly into debate agent
angle selection — closing the loop between account
intelligence and messaging strategy.

**Stage 4 — Writing Agent** (`writing-agent.js` v6)

Writes the full 11-touch sequence. Renders one `.docx`
per persona, 12 tabs: persona cover page (identity,
signals, winning angle, runner-up angle) plus one tab
per touchpoint. Also generates an account-level persona
overview summary `.docx` listing all qualified personas
ranked by score with winning and runner-up angles.
Model: `claude-sonnet-4-6`.

**Stage 5 — Voice Agent** (`voice-agent.js` v2)

Three-pass style enforcement against a 20-principle
voice doctrine built from 18 months of documented
outreach patterns.

*Pass 1 (deterministic):* Auto-corrects em dashes,
prohibited openers, bump re-pitch language, generic
CTAs. Runs specificity check on first-touch email with
rewrite loop (max 3 attempts). Flags rhetorical
inner-monologue patterns for review.

*Pass 2 (API):* Rewrites all touches against examples
corpus while preserving factual content.

*Pass 3:* Re-runs validator checks against rewritten
content. Reverts any touch that fails. Logs reversions.

Never blocks — pipeline always continues.
Model: `claude-sonnet-4-6`.

**Stage 6 — Validator** (`validator-agent.js` v6)

Pre-export quality gate. Named rules, mostly
deterministic. Enforces 3-tier source provenance
doctrine at the output layer:
- Tier 1 (merchant-owned sources): required for all
  capability claims
- Tier 2 (3rd-party marketplaces): allowed only as
  liability signals
- Tier 3 (aggregators — ZoomInfo, Apollo.io, coupon
  sites, data brokers): rejected entirely

Doctrine enforced at three independent layers:
research agent (Tier 3 URLs scrubbed before writing to
context), writing agent (prompt-level), and validator
(output-level). A contaminated source has three
independent chances to be caught.

Annotates rather than blocks.
Models: `claude-haiku-4-5` + `claude-sonnet-4-6`.

**Stage 7 — Drive Upload**

Finished `.docx` files are uploaded to Google Drive
manually via Claude.ai chat session. The runner
currently references a deprecated upload script that
errors on execution — Stage 7 is a known open item.
Actual upload path is manual.

---

## Shared state

All stages share a single `pipeline_context.json` per
account located at `sequences/{AccountName}/`. Each
stage reads from prior sections and appends to its own.
Re-running a completed stage triggers an interactive
confirmation prompt unless `PIPELINE_NONINTERACTIVE=1`
is set.

Stage keys: `engagement_check`, `persona_sourcing`,
`research_synthesis`, `debate`, `writing`,
`voice_check`, `validation_results`.

---

## Cost tracking

`lib/usage-logger.js` instruments every API call across
all agents. Appends one entry per call to
`sequences/{Account}/cost-log.json`: timestamp, agent,
stage, account, persona, model, token counts, and
`cost_usd`. `printCostSummary()` prints a per-agent
cost table after every run.

---

## What it has produced

Built a custom Gong MCP server alongside this pipeline.
Used it to query 363 calls across 65 accounts and
identify the precise product architecture gap costing
deals across an entire sales vertical — 6 themes, 15
findings, verbatim transcript evidence, causation vs.
correlation analysis. Report delivered to product
leadership May 2026.

Right-to-win analysis in progress: 36 accounts, 61
opportunities, 378 calls.

Sequence pipeline has run end-to-end on 38 accounts
across two territories (US retail/CPG and UK/EU).

---

## Pipeline visualization

See `viz/pipeline-2-sequence-generation.html` for the
full interactive architecture diagram.

---

## Built by

John Swanson — petroleum engineer, former operator of
Hawaii's largest watersports operation, current Market
Development Manager at Ordergroove.  
No CS degree. Started building in late 2024.

[github.com/drillbit-to-llm](https://github.com/drillbit-to-llm/prospecting-intelligence-pipeline)
