# Feature Proposal — **Dexter Workflows**

> Voice-triggered shortcuts that do many things across many apps in one go. A natural next step for Dexter's existing **templates / reusable actions**.

---

## The pitch in one line

> *"Run my morning setup."* *"Send the usual Friday update about the Q2 launch."* *"Save this article for later and remind me Monday."*
>
> One sentence → many things happen, in many apps, with the right details filled in by the AI.

---

## Why this feature, why now

| Reason | Detail |
|---|---|
| **It's already on the roadmap** | The pack mentions *"general assistant features like notes, scheduling, integrations, and automations across apps."* This is exactly that. |
| **It builds on something users already understand** | Templates already exist in Dexter. Workflows are the same idea — just smarter, voice-driven, and able to use more than one app at a time. |
| **It uses everything Dexter has already built** | The brain, the desktop side, the integrations, memory, profiles, stats. It doesn't add a new vertical — it makes everything already built more useful. |
| **It works for everyone, not just streamers** | Anyone using a computer can benefit. Streaming use cases become one example, not the whole feature. |
| **It sets up future "Dexter speaks up on its own" features** | Once workflows exist, Dexter can suggest them: "you do this every Monday — want me to make a shortcut?" |

---

## What a Workflow is

A **named shortcut that runs several steps** — on demand, on a schedule, or when something happens.

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

---

## Example workflows

| Workflow | Triggered by | What it does |
|---|---|---|
| **Morning setup** | "good morning" / 9 AM | open Slack + calendar + email · post a standup message · turn on focus mode · play a playlist |
| **Save & summarize** | "save this for later" | grab the open browser tab · ask the AI for a summary · store it in your notes · tag it · optionally Telegram it to yourself |
| **Friday update** | "send the usual Friday update about <X>" | gather this week's notes · let the AI write the message · post to Telegram · log it |
| **Focus block** | "focus for 90 minutes" | mute notifications · set Slack status · start a timer · play music · undo everything when done |
| **End of day** | "wrap up" / 6 PM | close apps · post a summary · send open tabs to a reading list · back up your notes |

Streamer-specific shortcuts (pre-stream setup, post-stream wrap-up) become *one kind* of workflow — not the whole feature.

---

## How it fits into Dexter

### Slotting into the existing layers

| Existing layer | What Workflows adds |
|---|---|
| Voice layer | Recognises new commands like *"run my morning setup"* or *"make a workflow from what I just did"* |
| Desktop app | A panel showing what's running, how it's going, and prompts for any missing details |
| Server (the brain) | A new module that stores workflows, fills in details, runs them, and schedules them |
| Feature modules | Each existing feature (Telegram post, OBS scene change, etc.) tells the workflow system "here's what I can do and what I need to know" |
| Desktop side | Just runs steps one after another using the same channel that already exists |
| Database | Three new tables: workflows, workflow runs, and step results |

### The "Action Contract" — the key idea

Every action (old or new) describes itself in a standard way:

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

This is also exactly what's needed for Dexter's "make sure it really happened" goal — every step has to come back with proof.

---

## Tech stack & packages

### What we already have and reuse
- **TypeScript / Node** for the server, **React** for the screen, **Electron** for the desktop app, **Supabase / Postgres** for storage.
- The voice layer and the brain that already routes requests.

### New packages — picked on purpose, not by habit

| Package | What it does | Why this one |
|---|---|---|
| **`zod`** | Checks that the data going in and out of each step matches what's expected | One short definition gives us live checking, code types, and a description the AI can use to fill in details. Three jobs, one tool. |
| **`xstate`** | Tracks where each running workflow is up to (waiting, running, paused, done, failed) | Easy to pause, resume, watch, and visualise. *Skip in the first version if we want to keep things simple.* |
| **`bullmq`** (with Redis) | Runs workflows on a schedule or in the background, retries when things fail | Battle-tested. Has a dashboard. Saves us writing this ourselves. |
| **`ioredis`** | The connection to Redis that BullMQ needs | Standard pairing — nothing fancy. |
| **`pgvector`** (Supabase add-on) | Lets users find a workflow by describing it ("the one that posts to channels") | Free with Supabase. No new servers. |
| **`@anthropic-ai/sdk`** | Talks to Claude for filling in details and writing messages | Going direct keeps things fast. Heavier frameworks add layers we don't need. |
| **`pino`** | Writes logs in a structured way | Fast, readable, plays nicely with everything else. |
| **`@opentelemetry/*`** | Lets us trace one user request across the whole system | The only realistic way to debug "why is this slow" or "where did this fail" in a system this spread out. |
| **`langfuse`** *(optional)* | Records every AI call so we can review and improve them | Worth adding once the AI's quality matters in production. |
| **`@xyflow/react`** *(later)* | A visual editor for power users to drag steps around | Best-looking, well-supported library for this. |
| **`electron-store`** | Saves a copy of workflows on the user's machine for quick access and offline drafts | Standard for Electron apps. |

### What we're **not** using (and why)

- **LangChain / LangGraph** — too much framework wrapping every AI call. We want fast, direct control.
- **Temporal** — designed for workflows that run for hours or days. Ours run for seconds. Not worth the extra servers.
- **A new message system like Kafka or NATS** — Redis and Postgres are enough for this scale. Don't add servers we can't justify with a real number.

---

## What the database tables look like (rough sketch)

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

---

## Rolling it out in stages

| Stage | What's in it | Goal |
|---|---|---|
| **v0 — Internal** | Update 3–5 existing actions to the new contract; workflows written by hand in JSON; voice trigger only; steps run one after another | Prove the idea works. Use it ourselves. |
| **v1 — Public release** | A library of starter workflows; "make a workflow from what I just did"; AI fills in details; user sees what worked and what didn't | First version users actually get. |
| **v2 — Power features** | Visual editor, scheduling, "if this then that" branches, steps in parallel, sharing between users | Make it sticky. |
| **v3 — Proactive** | Dexter notices you doing the same things repeatedly and offers to turn them into a workflow | Connects to the bigger "Dexter speaks up" direction. |

---

## The judgment calls I'd make

- **Define workflows as data, not code.** Storing them as JSON makes them easy to share, version, and lock down. Letting users write code is more flexible but a security and maintenance headache. → **Go with data.**
- **Visual editor in v1?** No. Voice and JSON are enough at first. The visual editor is a v2 feature once people care enough to hand-edit.
- **Use xstate or write the runner ourselves?** Write it ourselves to start. Switch to xstate when retries, pausing, and branching demand it. Don't pay framework cost before we need it.
- **AI per step or AI per workflow?** Once per workflow, to fill in the details up front. The AI should not be in the middle of execution — once we know the details, steps run predictably.
- **Run on the server or on the user's machine?** Step by step — each action says where it should run. Same logic as today, just applied to a sequence.

---

## Risks & open questions

1. **Trust.** A workflow that misfires is much worse than one wrong click. Need a "show me what would happen first" mode.
2. **Permissions.** Each workflow says upfront what it can touch (Telegram, OBS, etc.). User approves once when they create it. Can take it back.
3. **Retrying without doing it twice.** If step 3 of 5 fails and we retry, we can't post to Telegram twice. Each step needs a unique key so we know what's already done.
4. **Cancelling cleanly.** If the user interrupts mid-workflow, every step needs to know to stop — no stale actions landing 10 seconds later.
5. **Sharing workflows.** Big upside, big trust risk. Push to v2 minimum, with mandatory permission review when someone imports a workflow.

---
