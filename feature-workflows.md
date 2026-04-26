# Feature Proposal — Dexter Workflows

This is a proposal for a new feature called Dexter Workflows. It is a way to trigger several actions across several apps with a single voice command. It is a natural next step for Dexter's existing templates and reusable actions.

## The pitch in one line

Imagine the user says "run my morning setup," or "send the usual Friday update about the Q2 launch," or "save this article for later and remind me Monday." With Workflows, one sentence makes many things happen, across many apps, with the right details filled in by the AI.

## Why this feature

There are a few reasons this feature makes sense for Dexter right now. It is already on the roadmap, because the context pack mentions general assistant features like notes, scheduling, integrations, and automations across apps, and this is exactly that. It also builds on something users already understand, because templates already exist in Dexter. Workflows are the same idea, just smarter, voice-driven, and able to use more than one app at a time.

It also uses everything Dexter has already built. The brain, the desktop side, the integrations, the memory, the profiles, and the stats all get reused. So instead of adding a new vertical, it makes everything already built more useful. Another good reason is that Workflows work for everyone, not just streamers. Anyone using a computer can benefit from them, and streaming use cases simply become one example rather than the whole feature. Finally, it sets up future "Dexter speaks up on its own" features. Once workflows exist as a building block, Dexter can suggest them to the user. For example, "you do this every Monday — want me to make a shortcut?"

## What a Workflow actually is

A Workflow is a named shortcut that runs several steps. It can be triggered on demand, on a schedule, or when something happens. The flow looks like this:

```
                  ┌──────────────┐
                  │   Trigger    │   voice / hotkey / schedule / event
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │   Details    │   pulled out of what you said,
                  │              │   asked for if missing
                  └──────┬───────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
   │ Step A  │      │ Step B  │      │ Step C  │   run together where safe
   │(Telegram)│     │  (OBS)  │      │  (Web)  │
   └────┬────┘      └────┬────┘      └────┬────┘
        └────────────────┼────────────────┘
                         │
                  ┌──────▼───────┐
                  │  Confirmation │   each step reports back
                  └──────────────┘
```

## A few example workflows

A morning setup workflow could be triggered by saying "good morning" or by a 9 AM schedule. It would open Slack, the calendar, and email, post a standup message, turn on focus mode, and play a playlist.

A save and summarise workflow could be triggered by "save this for later." It would grab the open browser tab, ask the AI for a summary, store it in the user's notes, tag it, and optionally Telegram it to them.

A Friday update workflow could be triggered by "send the usual Friday update about the Q2 launch." It would gather this week's notes, let the AI write the message, post it to Telegram, and log it.

A focus block workflow could be triggered by "focus for 90 minutes." It would mute notifications, set the user's Slack status, start a timer, play music, and undo everything when the time is up.

An end-of-day workflow could be triggered by "wrap up" or by a 6 PM schedule. It would close apps, post a summary, send open tabs to a reading list, and back up the user's notes.

Streamer-specific shortcuts like pre-stream setup or post-stream wrap-up just become one kind of workflow rather than the whole feature.

## How it fits into Dexter

Workflows slot neatly into the layers Dexter already has. The voice layer learns to recognise new commands like "run my morning setup" or "make a workflow from what I just did." The desktop app gets a panel showing what is running, how it is going, and any details that still need to be filled in. The server, which is the brain of Dexter, gets a new module that stores workflows, fills in details, runs them, and schedules them. Each existing feature module — like Telegram posting or OBS scene changes — tells the workflow system what it can do and what details it needs. The desktop side just runs steps one after another using the same channel that already exists. And the database gets three new tables: one for workflows, one for runs, and one for step results.

The key idea that holds all of this together is what I am calling the Action Contract. Every action, whether it is old or new, describes itself in a standard way. Roughly, that looks like this in code:

```ts
{
  id: 'telegram.post',
  description: 'Post a message to a Telegram channel',
  params: ZodSchema,                   // what details are needed
  execute: (params, ctx) => Receipt,   // does the thing, reports back
  idempotencyKey: (params) => string,  // so retrying doesn't double-post
  scopes: ['telegram:write'],          // what permission it needs
}
```

This same shape is exactly what Dexter needs for its goal of making sure things really happened. Every step has to come back with proof, and the contract enforces that.

## Tech stack and packages

For the foundations, we just reuse what Dexter already has. The server side is TypeScript and Node. The screen is React. The desktop app is Electron. Storage is Supabase and Postgres. The voice layer and the brain that already routes requests are reused as well.

On top of that, there are a few packages I would add deliberately, not just out of habit. The first is `zod`, which checks that the data going in and out of each step matches what is expected. One short definition gives us live checking, code types, and a description the AI can use to fill in details — three jobs in one tool. The next one is `xstate`, which would track where each running workflow is up to. It makes pausing, resuming, watching, and visualising runs much easier, but we can skip it in the very first version if we want to keep things simple.

For background jobs and scheduling, I would use `bullmq` with Redis. It runs workflows on a schedule or in the background and retries failed steps. It is well-tested, has a dashboard, and saves us writing this part ourselves. It pairs with `ioredis`, which is just the standard connection library. For letting users find a workflow by describing it — like "the one that posts to channels" — `pgvector` is a good choice. It is a free Supabase add-on, so no new servers are needed.

For talking to Claude, the Anthropic SDK directly is the right call. Going direct keeps things fast, while heavier frameworks add layers we don't really need. For logs, `pino` is fast, readable, and plays nicely with everything else. To trace a single user request across the whole system, OpenTelemetry is more or less the only realistic option in a setup this spread out. Once the AI's quality matters in production, Langfuse becomes worth adding — it records every AI call so we can review and improve them. Later on, when we want a visual editor for power users, `@xyflow/react` is the best-looking and best-supported library for that. And for keeping a copy of workflows on the user's machine for quick access and offline drafts, `electron-store` is the standard choice for Electron apps.

There are also a few things I would deliberately not use. LangChain and LangGraph add too much framework around every AI call, and we want fast and direct control. Temporal is great for workflows that run for hours or days, but ours run for seconds, so it is not worth the extra servers. And a new message system like Kafka or NATS would be overkill — Redis and Postgres are enough at this scale, and we should not add servers we cannot justify with a real number.

## What the database tables look like

Here is a rough sketch of the new tables:

```sql
workflows (
  id uuid pk,
  user_id uuid,
  name text,
  description text,
  definition jsonb,         -- the steps and what details they need
  embedding vector(1536),   -- for "find me the workflow that does X"
  scopes text[],            -- what permissions it needs
  version int,
  created_at timestamptz
);

workflow_runs (
  id uuid pk,
  workflow_id uuid,
  user_id uuid,
  trigger jsonb,            -- voice command / schedule / event
  resolved_params jsonb,    -- the details, once filled in
  status text,              -- pending / running / done / failed / cancelled
  started_at timestamptz,
  finished_at timestamptz
);

workflow_step_results (
  run_id uuid,
  step_id text,
  attempt int,
  receipt jsonb,            -- what was tried, did it work, proof
  duration_ms int,
  primary key (run_id, step_id, attempt)
);
```

## Rolling it out in stages

I would not try to ship all of this at once. The first stage, v0, is internal only. We update three to five existing actions to use the new contract, write workflows by hand in JSON, allow only voice triggers, and run steps one after another. The goal here is just to prove the idea works and use it ourselves.

The second stage, v1, is the first public release. We give users a library of starter workflows, add the "make a workflow from what I just did" feature, let the AI fill in details, and surface what worked and what did not in the UI.

The third stage, v2, is where we add the power features. That means the visual editor, scheduling, "if this then that" branches, steps running in parallel, and sharing workflows between users. This is what makes the feature sticky.

The fourth stage, v3, is the proactive one. Dexter notices that the user keeps doing the same things in the same order and offers to turn them into a workflow. This connects directly to the bigger "Dexter speaks up on its own" direction.

## The judgment calls I would make

There are a few decisions that I think matter most. The first is to define workflows as data rather than code. Storing them as JSON makes them easy to share, version, and lock down, while letting users write actual code is more flexible but quickly turns into a security and maintenance headache. So I would go with data.

The second is whether to build the visual editor in v1, and I would say no. Voice and JSON are enough at first, and the visual editor becomes a v2 feature once people care enough to want to hand-edit. The third is whether to use xstate or write the runner ourselves. I would write it ourselves to start, and switch to xstate later, when retries, pausing, and branching demand it. Don't pay framework cost before we need it.

The fourth is about how often we call the AI. I would call it once per workflow, to fill in the details up front. The AI should not be in the middle of execution. Once we know the details, the steps run predictably. The fifth is where workflows actually run — on the server or on the user's machine. The answer is step by step. Each action says where it should run, using the same logic Dexter already uses for single actions.

## Risks and open questions

There are a few risks I would want to talk through. The first is trust. A workflow that misfires is much worse than one wrong click, so we need a "show me what would happen first" mode. The second is permissions. Each workflow should declare upfront what it can touch, like Telegram or OBS, and the user should approve once when they create it and be able to take that approval back later.

The third is making sure retries do not cause damage. If step 3 of 5 fails and we retry, we cannot end up posting to Telegram twice. Each step needs a unique key so we know what is already done. The fourth is cancelling cleanly. If the user interrupts in the middle of a workflow, every step needs to know to stop, so we don't get stale actions landing ten seconds later. The fifth is sharing workflows. There is a lot of upside here, but also a real trust risk, so I would push it to v2 at the earliest, with a mandatory permission review every time someone imports a workflow.
