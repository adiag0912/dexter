# Feature Proposal — **Dexter Workflows**

> Voice-triggered, parametric, cross-app automations. A general-purpose evolution of Dexter's existing **templates / reusable actions**.

---

## One-line pitch

> *"Run my morning setup."* *"Send the usual Friday update about the Q2 launch."* *"Save this article for later and remind me Monday."*
>
> One sentence → many actions, across many apps, with the right parameters filled in by the LLM.

---

## Why this feature, why now

| Reason | Detail |
|---|---|
| **It's already on the roadmap** | The pack lists *"additional general agent capabilities such as notes, scheduling, integrations, and deeper desktop/web workflows."* This is exactly that. |
| **It generalizes an MVP feature** | "Templates / reusable actions" already exists. Workflows is the same idea, made parametric, voice-recordable, and cross-app. Low conceptual lift for users. |
| **It reuses every layer Dexter has built** | Backend orchestration · local execution · integrations · memory · profiles · analytics. No new vertical — instead a horizontal that *multiplies* the value of everything underneath. |
| **It moves Dexter beyond streaming** | A general copilot capability. Streamer use cases are a subset, not the limit. Important for the "broader creator operating layer" vision. |
| **It compounds with proactive behavior later** | Once workflows are first-class, the copilot can *suggest* them ("you do this every Monday — want me to make it a workflow?"). |

---

## What a Workflow is

A **named, parametric DAG of actions** that Dexter can run on demand, on schedule, or on trigger.

```
                  ┌──────────────┐
                  │   Trigger    │   voice / hotkey / schedule / event
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │  Parameters  │   inferred from utterance, prompted if missing
                  └──────┬───────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
   │ Step A  │      │ Step B  │      │ Step C  │   parallel where safe
   │ (Telegram)│    │ (OBS)   │      │ (Web)   │
   └────┬────┘      └────┬────┘      └────┬────┘
        └────────────────┼────────────────┘
                         │
                  ┌──────▼───────┐
                  │   Receipts   │   typed result, surfaced + logged
                  └──────────────┘
```

---

## Example workflows

| Workflow | Triggered by | Steps |
|---|---|---|
| **Morning setup** | "good morning" / 9 AM | open Slack + calendar + email · post standup template to channel · enable focus mode · play playlist |
| **Save & summarize** | "save this for later" | grab active tab URL · LLM summary · store in notes table · tag · optional Telegram self-message |
| **Friday update** | "send the usual Friday update about <X>" | pull this week's notes · LLM compose from template · post to Telegram channel · log |
| **Focus block** | "focus for 90 minutes" | mute notifications · set Slack status · start timer · play music · auto-restore on completion |
| **End of day** | "wrap up" / 6 PM | close apps · post EOD summary · file open tabs to reading list · backup notes |

Streaming use cases (pre-stream setup, post-stream wrap-up) become *one* category of workflow — not the whole feature.

---

## Architecture

### How it slots into Dexter's existing layers

| Existing layer | What Workflows adds |
|---|---|
| Realtime voice | New intent: *"run workflow"* / *"create workflow from what I just did"* |
| Frontend runtime | Workflow execution UI, status surface, parameter prompts |
| Backend orchestration | New `workflows` module: definition CRUD, parameter inference, executor, scheduler |
| Feature / domain modules | Each existing module exposes its actions to Workflows via a shared **Action Contract** |
| Local execution | Same channel — workflows just call multiple local actions in sequence |
| Persistent data | New tables: `workflows`, `workflow_runs`, `workflow_step_results` |

### The Action Contract (the linchpin)

Every action — old or new — registers itself with a typed contract:

```ts
















































Hi Astemir,I've gone through the context pack and put together two short markdown documents to organise my thinking ahead of our chat. I've pushed them to a public repo so you can take a look beforehand if you'd like:https://github.com/adiag0912/dexterdexter-overview.md — my read of the system: architecture, memory hierarchy, current focus, and the risks I think are worth discussing.feature-workflows.md — a feature proposal (Dexter Workflows — voice-triggered, parametric, cross-app automations) that generalises the existing templates module and reuses every layer you've already built.These are meant as a starting point for the discussion rather than a finished pitch — if you spot anything I've misread about the system, or an angle you'd rather we focused on instead, let me know and I'll adjust before Sunday.Looking forward to it.Best,
Aditya












































{
  id: 'telegram.post',
  description: 'Post a message to a Telegram channel',
  params: ZodSchema,                // for LLM to fill
  execute: (params, ctx) => Receipt, // returns confirmation
  idempotencyKey: (params) => string,
  scopes: ['telegram:write'],
}
```

This is also the right shape for **execution truth** — every step returns a receipt with `requested / attempted / confirmed / evidence`.

---

## Tech stack & packages

### Reuse what's already there
- **TypeScript / Node** backend, **React** frontend, **Electron** runtime, **Supabase / Postgres** storage.
- Existing voice + orchestration paths.

### New packages — chosen deliberately

| Package | Use | Why this one |
|---|---|---|
| **`zod`** | Schema validation for workflow definitions, action params, LLM structured output | Already idiomatic in Node/TS; one source of truth for runtime + types + LLM JSON schema |
| **`xstate`** | Per-run execution state machine (pending → running → paused → done / failed / cancelled) | Pausable, resumable, observable runs. Visualizable. Pays for itself once retries + branching exist. *Skip in v1 if scope is tight.* |
| **`bullmq`** (Redis) | Scheduled & async workflow runs, retries with backoff | Mature, observable, supports cron + delayed jobs + rate limits. Better than rolling our own. |
| **`ioredis`** | BullMQ + cooldowns + per-user lock keys | Standard pairing with BullMQ |
| **`pgvector`** (Supabase ext.) | Semantic workflow search ("find my workflow that posts to channels") | Free with Supabase, no new infra |
| **`@anthropic-ai/sdk`** | Parameter inference + step composition (Haiku for cheap calls, Sonnet for harder ones) | Direct SDK > heavy frameworks for tight latency control |
| **`pino`** | Structured logs | Fast, JSON, plays nicely with OTel |
| **`@opentelemetry/*`** | Distributed traces across orchestrator → executor → local | Per-turn span tree is the *only* way to debug agent latency |
| **`langfuse`** *(optional)* | LLM call traces, prompt versioning, eval | Worth it once parameter-inference quality matters |
| **`@xyflow/react`** *(v2)* | Visual workflow editor for power users | Best-in-class node graph UI for React |
| **`electron-store`** | Per-user local workflow cache + offline drafts | Standard for Electron; encrypted at rest option |

### Explicitly **not** picked (and why)
- **LangChain / LangGraph** — too much abstraction for the latency budget; we want explicit control over each tool call, not a framework's opinion.
- **Temporal** — perfect for long-running workflows, but heavy infra for runs measured in seconds. Revisit if average run time exceeds minutes.
- **A new message broker (NATS / Kafka)** — Redis + Postgres is enough at current scale. Don't add infra you can't justify with a metric.

---

## Database shape (sketch)

```sql
workflows (
  id uuid pk,
  user_id uuid,
  name text,
  description text,
  definition jsonb,         -- DAG of steps + params schema
  embedding vector(1536),   -- semantic search
  scopes text[],            -- capability allow-list
  version int,
  created_at timestamptz
);

workflow_runs (
  id uuid pk,
  workflow_id uuid,
  user_id uuid,
  trigger jsonb,            -- voice utterance / schedule / event
  resolved_params jsonb,
  status text,              -- pending / running / done / failed / cancelled
  started_at timestamptz,
  finished_at timestamptz
);

workflow_step_results (
  run_id uuid,
  step_id text,
  attempt int,
  receipt jsonb,            -- requested / attempted / confirmed / evidence
  duration_ms int,
  primary key (run_id, step_id, attempt)
);
```

---

## Phased rollout

| Phase | Scope | Goal |
|---|---|---|
| **v0 — Internal** | Action Contract refactor of 3-5 existing actions; manual JSON workflow definitions; voice trigger only; sequential execution | Prove the contract; dogfood internally |
| **v1 — General release** | Library of starter workflows; voice "create workflow from what I just did"; LLM parameter inference; receipts surfaced in UI | First user-facing release |
| **v2 — Power features** | Visual editor, scheduling, conditional branches, parallel steps, sharing/marketplace | Make it sticky |
| **v3 — Proactive** | Dexter detects repeated action sequences and *suggests* turning them into workflows | Closes the loop with proactive copilot direction |

---

## Tradeoffs & judgment calls

- **Imperative vs declarative definitions.** Declarative (JSON DAG) is shareable, versionable, sandboxable. Imperative (TS function) is more flexible but a security and portability nightmare. → **Declarative.**
- **Build the visual editor in v1?** No. Voice + JSON is enough for v1. Visual editor is the v2 stickiness lever.
- **xstate or hand-rolled executor?** Hand-rolled in v0–v1. Promote to xstate when retry / pause / resume / branching demand it. Don't pay framework cost before you need it.
- **One LLM call per step, or one per workflow?** One per workflow for parameter inference (cheap, fast). LLM should not be in the hot path of execution — actions run deterministically with pre-resolved params.
- **Where do workflows execute — backend or local?** Step-by-step decision via the Action Contract's location hint. Same routing logic that already exists for single actions.

---

## Risks & open questions

1. **Trust budget.** A workflow that misfires is much worse than a single action that misfires. Need a clear preview / dry-run mode and per-workflow scopes.
2. **Capability scoping.** Each workflow declares the scopes it needs. User approves once at creation, not per run. Revocable.
3. **Idempotency across the DAG.** What if step 3 of 5 fails on retry? Need step-level idempotency keys, not just run-level.
4. **Cancellation.** A turn-id / cancellation token must flow into every step so an interrupted user doesn't get stale actions landing 10s later.
5. **Sharing / marketplace.** Big upside, big trust surface. Defer to v2 minimum, with mandatory scope review on import.

---

## Why I'd pitch this in the interview

- It demonstrates **judgment about leverage** — picks a horizontal that multiplies existing investments instead of a vertical that adds one more thing to maintain.
- It connects directly to the team's stated **reliability / execution-truth** focus — the Action Contract is the substrate for both.
- It opens a clear **phased path** (v0 → v3) that's shippable end-to-end, not a moonshot.
- It sets up the **proactive copilot** future direction without requiring it on day one.
