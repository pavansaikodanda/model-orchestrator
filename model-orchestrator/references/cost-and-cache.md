# Cost and cache mechanics

Load this when reasoning about cost, or when deciding batch timing, reuse or top-ups.

## Where cost comes from
- **Startup:** every agent loads a large fixed context (platform instructions and tool definitions, often around 65k tokens) before reading the brief. The first agent on a model writes it to the cache; later ones on the same model read it far cheaper. The cache is per model.
- **Every tool call re-reads the whole context** of whoever makes it. Long agent runs and long boss conversations get expensive per round.
- **Output costs more than input.** Reviewing a file is much cheaper than writing it.
- **Long prompts can cost more per token** on some models (for example, above about 100k tokens). Keep cheap agents' context small.

## Cache lifetime
- The default cache lasts about 5 minutes and resets on each use; it expires after 5 idle minutes.
- A cold top-up re-pays the agent's whole context, so top up immediately.
- Inside an app, the platform controls cache settings for sub-agents. On the API you can choose a 1-hour cache. Its write costs about 1.6× more, so it is worth it only when reuse comes after a gap of more than 5 minutes.

## Practical rules
- Warm first: one agent per model, then the rest a few seconds later. This matters most for Sonnet and Opus; Haiku's startup is cheap.
- Reuse a warm agent for similar follow-up work; retire it once its context is large.
- A parallel batch returns when its slowest agent finishes, so keep fast agents that may need follow-ups out of slow batches.
- Boss turns re-read the boss's whole conversation. In long chats, plan fewer waves and fewer turns.

## Rough break-even (illustrative, API prices)
- A Haiku agent pays off on almost any mechanical or research chunk.
- A Sonnet agent pays off only on sizeable output (large code or tests, broad research). Below that, the boss doing it inline is often cheaper.
