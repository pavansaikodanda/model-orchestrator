---
name: "model-orchestrator"
description: "Orchestrate multi-part tasks: split into chunks, route each to the cheapest capable sub-agent model (Haiku/Sonnet/Opus, Fable if available), review, escalate on failure. Use when user says orchestrate or hands over large multi-step work."
---

# Model Orchestrator

You are the boss. You plan, route, review, and decide. Sub-agents do the bulk work on cheaper models. The goal is the same quality at lower usage cost: limits are weighted by model, so work done on Haiku/Sonnet drains less quota than the same work on Opus.

## 0. Who is the boss

- If this chat runs on Fable, Fable is the boss and Opus becomes a sub-agent tier.
- Otherwise Opus is the boss and the top sub-agent tier.
- Never route to Fable unless it is the session model or the user says it is available (it may run on paid credits).

## 1. Decide whether to orchestrate at all

If the boss can finish the task in a single pass (one step, one file, a quick answer), do it inline. Spinning up agents for that wastes time and quota: every sub-agent starts cold and must be briefed.

Orchestrate only when there are several substantial chunks, or the work clearly benefits from a cheaper model doing volume.

If something is unclear before splitting, ask the user first. Otherwise, don't ask for plan approval; just run.

## 2. Split and route

Break the task into chunks. Map dependencies before launching anything, because parallel agents that depend on each other build against wrong assumptions.

Route each chunk by what the model can actually handle (use your own knowledge of model capability; the examples below are only guides):

| Tier | Typical work |
|---|---|
| Haiku | file/code search, lookups, one-line or small mechanical edits, formatting |
| Sonnet | generating classes/components, standard features, tests, focused research |
| Opus | architecture, design trade-offs, complex logic, tricky debugging |
| Fable | only when available; the hardest reasoning |

Send obvious Opus-level work straight to Opus. Climbing the ladder on it wastes runs.

**Flow:**
- Independent chunks run in parallel.
- Dependent chunks run in series: when an agent finishes, it reports to you, and you launch the next dependent agent. Agents cannot message each other; you are the relay.
- Each shared file has exactly one owner agent, or the edits happen in series. Never let two agents edit the same file at once.

**Caps:** at most 5 agents running at once; at most 15 agents per task. If a task needs more than 15, stop and check with the user. Within those caps, choose the parallel/series mix yourself.

**Cold start (cache) rules.** Every agent loads the same large fixed context (platform instructions and tool definitions, often around 65k tokens) before reading your brief. The first agent on each model pays to write it to the cache; later agents on the same model within about 5 minutes read it far cheaper. The cache is per model.
- **Warm first:** launch one agent per model first, then launch the rest of that model's agents a few seconds later, so they hit the warm cache. Never cold-start several agents on the same model at the same moment.
- **Reuse before respawn:** for the next chunk of similar work, message an agent that's already running instead of spawning a new one; its context is already warm. Retire it after a few tasks, once its context gets large.

**Generic work (lookups, web search, multi-file reads).** The cost driver is the number of tool rounds the boss takes, because each round re-reads the boss's whole context.
- 1–2 rounds (one file read, one fetch): do it inline.
- 3+ rounds (several searches, pages, or files): delegate to a Haiku agent.

## 3. Briefs

Keep briefs short and crisp: the goal, inputs, files the agent owns, the definition of done, and the summary format. Briefs are written by the boss, so every extra line costs top-tier output.

Do not paste conventions into every brief. Point agents to the conventions source (see section 6).

## 4. Summaries from sub-agents

Require this format, never a bare "done":

```
Done: <what was actually completed>
Files: <paths touched>
Assumptions: <anything assumed>
Issues: <problems found, open questions, anything not done>
```

**Research and lookup agents** must not summarize what they found. Return the exact relevant excerpts verbatim, each with its source (URL or file path and line), and give exact figures (prices, versions, limits) as written in the source. The boss does the interpreting, so nothing is lost or distorted in a cheaper model's paraphrase.

If you need more detail, message the same agent again; it keeps its context for this session. Pull detail only when the question needs it, since everything returned adds to your context.

## 5. Review

- **Logic-heavy work** (triggers, handlers, integrations, core algorithms, key reasoning): read the full output. Reading costs input, which is far cheaper than you generating the output yourself.
- **Trivial changes** (a field, a label, a one-line edit): the summary is enough.
- Check consistency across agents against the conventions source.

## 6. Conventions (priority order)

1. What the user says in chat
2. A conventions file in the project, if one exists
3. Your own knowledge and best practice

## 7. Failure and escalation

A chunk has failed when the boss rejects it in review, or the result doesn't match what the user asked for. An agent that needs repeated clarifying questions to make progress also counts as under-powered.

On failure:
1. Check the brief first. If it was unclear, fix it and retry on the same model.
2. If it still fails, move up one tier: Haiku → Sonnet → Opus (→ Fable if available).
3. For the rest of this session, send that type of task to the higher tier from the start.
4. If the top sub-agent tier fails, the boss reasons through that chunk only, not the whole task.
5. If the boss fails twice on a chunk, stop and return to the user with what was tried. Do not loop.

## 8. Pipeline with side effects

For work that ends in an action (deploy, publish, send, data change):

generate (agents) → boss review → action (agent) → boss reviews the result → inform the user

- Sandbox, dev, or other reversible targets: run automatically.
- Production or anything irreversible: get the user's explicit yes before acting.

## 9. Salesforce layer (apply only when the task is Salesforce work)

- Dependency order: objects/fields → Apex/flows → LWC → tests → permissions.
- One deployment per org at a time: batch deploys and run them as one step, never from parallel agents.
- Validate before deploying (validate-only deploy plus tests); treat failed tests as a failed chunk.
- Review Apex for bulkification and governor limits; Haiku should not generate Apex classes.
- Before relying on org access, confirm sub-agents actually have the Salesforce tools/connector in this session.

## 10. Finishing

Tell the user, briefly: what was done, what was deployed or changed, any open issues, and which task types were escalated during the session.