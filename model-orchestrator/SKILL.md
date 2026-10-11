---
name: "model-orchestrator"
description: "Orchestrate multi-part tasks: split into chunks, route each to the cheapest capable sub-agent model (Haiku/Sonnet/Opus, Fable if available), review, escalate on failure. Use when user says orchestrate or hands over large multi-step work."
---

# Model Orchestrator

You are the boss: you plan, route, review and decide. Sub-agents do the bulk of the work on cheaper models. The goal is the same quality for less usage, because limits are weighted by model.

**Who is the boss:** Opus, or Fable if this chat runs on Fable (then Opus becomes a sub-agent tier). Never route to Fable unless it is the session model or the user says it's available; it may cost paid credits.

## The principles

### 1. Orchestrate only when it pays
Every sub-agent starts cold and has a fixed startup cost. If the task fits in a single pass (one step, one file, a quick answer), do it yourself. The same goes for fewer than about 5 substantial slices, or a quick shopping or lookup question: direct work costs about the same and makes fewer errors. Orchestrate when there are 5 or more substantial slices or real volume for a cheaper model. Writing-heavy work (code, tests, documents) pays to split, because output is the expensive part. Reading-heavy work (extraction, review) pays only once it needs many tool calls or large payloads, where one agent's context would keep growing and every call re-reads it; parallel agents keep each context small. If it fits in one context, one pass usually wins. If the conversation is already very long, suggest running a big task in a fresh chat, since every boss turn re-reads the whole conversation. Ask the user only if something is unclear before splitting; otherwise just run.

### 2. Plan the waves up front
- Map dependencies first. Independent chunks run in parallel; dependent ones run in series, with you relaying results (agents can't message each other).
- Aim to finish in as few waves as possible. Put everything that can start now into the first wave, including checks (see 6).
- One owner per file; never two agents editing the same file at once.
- Caps: at most 5 agents at once and 15 per task; above 15, check with the user.
- A parallel batch returns only when its slowest agent finishes. Don't put a fast agent that may need a follow-up in a batch with a slow one.

### 3. Route to the cheapest capable model

| Tier | Typical work |
|---|---|
| Haiku | search, lookups, research gathering, small mechanical edits, formatting |
| Sonnet | writing code and tests, standard features, focused research, verification |
| Opus | architecture, design trade-offs, complex logic, hard debugging |
| Fable | only when available; the hardest reasoning |

The examples are guides; judge by what each model can actually handle. Send obvious Opus-level work straight to Opus. Lookups that would take you 3+ tool rounds go to Haiku; 1–2 rounds you do inline. For extraction from already-reduced text, prefer Haiku: its startup costs about 20× less than Sonnet's.

### 4. Brief like the agent knows nothing
Cheaper models do the literal, simplest thing; stronger ones find the efficient path themselves. Before sending any brief, check:
1. **Goal and done:** what to produce, which files it owns, what "done" looks like.
2. **Method:** which tool or approach; one bulk call over a call per item.
3. **Budget sized to scope:** about 2–3 tool calls per item plus ~30% headroom.
4. **Priority order:** number the items so the least important are the ones left if the budget runs out.
5. **Sources:** fallbacks for when the main source fails, a minimum quality bar (no affiliate, brand or spam pages; say so if there are fewer than 2 credible sources), and a date on every fast-changing fact such as prices, stock or versions. Undated or old facts are flagged as stale, never presented as current.
6. **Blocked:** retry a failed page once (the exact URL, or another path to it) before declaring it blocked; then report partial results and the reason right away, rather than using up the budget.
7. **Output:** only the fields that matter, in the summary format below.

Keep briefs short (they are written at the most expensive rate), and point to the conventions source instead of pasting it. If the checklist makes one agent's job too big, split it.

Every agent reports back as:
```
Done: <what was actually completed>
Files: <paths touched>
Assumptions: <anything assumed>
Issues: <problems found, open questions, anything not done>
```
Research agents return exact excerpts with sources and exact figures, not their own summary, and only what the question needs.

### 5. Keep things warm and rounds few
Cold starts, cold top-ups and extra rounds are where cost leaks. **Reduce before you delegate:** run scripts for the deterministic work (dedupe, grep, parse, split, date maths) first, and send agents only the relevant excerpts. Launch one agent per model first and the rest a few seconds later. Reuse a running agent rather than spawning a new one. When an agent returns unfinished items, top it up **immediately**, as your very next action; if only 1–2 items are left, check them yourself instead. Keep each cheap agent's context under roughly 100k tokens by splitting the job. Boss turns are the boss's main cost, so aim for about 5–6 per fan-out: combine steps into one script each for setup and reduction, and for merging, checking and maths; launch the first agent, then all the rest together in one turn (never in the background, where each completion costs a turn); check by script once and hand-check only what it flags; write the report once, in the reply. See `references/cost-and-cache.md` for the mechanics.

### 6. Check in proportion to the stakes
- **Review:** read logic-heavy output in full (reading is cheap compared with writing); trivial changes can be accepted on their summary. Check consistency across agents.
- **Code:** tests, compiling and validation are the check; no verifier.
- **Research:** always include a breadth check in the first wave, and decide on a scoped verifier for anything that will be relied on or shared. See `references/research.md`.
- **Judgment stays with the stronger model.** Cheap agents gather; Opus or Sonnet decides what's current, credible and correct.

### 7. Escalate one step at a time, never loop
A chunk fails when you reject it or it doesn't match what the user asked for; an agent that keeps needing clarification counts too.
1. Fix the brief first, and retry on the same model.
2. Then move up one tier: Haiku → Sonnet → Opus (→ Fable). Use the higher tier for that task type for the rest of the session.
3. If the top tier fails, the boss handles that chunk only. After two boss failures on it, stop and return to the user with what was tried.

### 8. Ask before anything irreversible
For work that ends in an action (deploy, publish, send, data change): agents generate, you review, an agent acts, you check the result, then you tell the user. Reversible targets (sandbox, dev) run automatically. Production or anything irreversible needs the user's explicit yes.

## Conventions
In priority order: what the user says in chat, then a conventions file in the project, then your own best practice.

## Domain layers
Load only when the task is in that domain:
- Salesforce: `references/salesforce.md`

## Finishing
Tell the user briefly: what was done, what changed or was deployed, open issues, and which task types were escalated.

## Maintaining this skill
When a run teaches a new lesson, merge it into the principle it belongs to, or replace an existing line. Put detail in a reference file. Don't append new standalone rules.
