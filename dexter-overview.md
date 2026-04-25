# Dexter — My Read of the System

> A one-page mental model of Dexter as I understood it from the context pack. This is what I'll be reasoning from in the rest of the discussion.

---

## What Dexter is, in one line

A **desktop AI copilot for streamers** — voice/text in, real-time response, with awareness of what's happening live and the ability to trigger real actions across the streamer's machine, accounts, and integrations.

Not a chatbot. A **co-pilot** — meaning it acts, not just talks.

---

## The three things that make Dexter interesting (architecturally)

| | Why it matters |
|---|---|
| **It spans local + cloud + external systems** | Not "frontend + backend." Electron runtime, backend orchestration, Supabase, OBS, Twitch, Telegram, web, multiple model paths — all coordinated per turn. |
| **It must act, not just answer** | Execution truth (did the action *actually* happen?) is a first-class concern. Most AI products don't have this problem. |
| **It learns over time** | Short-term live context, post-stream digestion, long-term streamer + community profiles. A real memory hierarchy, not just a vector DB. (Maybe we can add Obisidan second brain here.) |

---

## Architecture at a glance

```
   ┌────────────────────────────────────────────────────────────┐
   │                       USER (voice / text)                   │
   └───────────────────────────────┬────────────────────────────┘
                                   │
                  ┌────────────────▼────────────────┐
                  │     Realtime Voice Layer         │  ← live STT / TTS, barge-in
                  └────────────────┬────────────────┘
                                   │
                  ┌────────────────▼────────────────┐
                  │   Frontend Runtime (Electron)    │  ← turn handling, coordination
                  └─────┬───────────────────────┬───┘
                        │                       │
        ┌───────────────▼──────────┐   ┌────────▼───────────────┐
        │  Backend Orchestration   │   │  Local Execution Layer │
        │  (planning, reasoning)   │   │  (OBS, desktop, files) │
        └───────┬──────────────────┘   └────────────────────────┘
                │
   ┌────────────▼─────────────┐    ┌──────────────────────────┐
   │   Feature / Domain       │◄──►│   Persistent Data Layer   │
   │   Modules (moderation,   │    │   (Supabase / Postgres)   │
   │   giveaways, templates,  │    │   streams, memory,        │
   │   analytics, etc.)       │    │   profiles, settings      │
   └──────────┬───────────────┘    └──────────────────────────┘
              │
   ┌──────────▼─────────────────────────────────────────────────┐
   │   External integrations: Twitch, Telegram, web, music, ... │
   └────────────────────────────────────────────────────────────┘
```

---

## Tech foundation

- **Electron** desktop app
- **React** frontend
- **Node / TypeScript** backend
- **Supabase / Postgres** persistent store
- **Multiple model paths** — live interaction, backend reasoning, data digestion (different latency/cost profiles)
- **External**: mic, OBS, Twitch, Telegram, web tooling

---

## How a turn flows (simplified)

1. User speaks or types
2. Realtime voice layer transcribes and handles interruption
3. Frontend runtime forwards into backend
4. Backend decides: conversational? backend-only? local action? fast-path?
5. Local execution runs anything machine-bound
6. Backend finalizes
7. Reply delivered through the live layer

A turn can be: **backend-only**, **backend-decided + locally executed**, **conversational**, or **fast-path deterministic**.

---

## The memory hierarchy (their real differentiator)

```
Raw stream evidence  ──compress──►  Short-term live context   (in-stream)
                     ──digest───►   Memory / highlights        (post-stream)
                     ──learn────►   Streamer + community       (long-term profiles)
                                    profile facts
```

Live layer never reasons over raw history — it gets **structured**, pre-selected context.

---

## Current product surface (MVP)

Voice/text copilot · OBS + desktop actions · Twitch controls + moderation · giveaways / timers / polls · templates · Telegram posting · web actions · music / player · per-stream + channel analytics · short-term live context · long-term memory · streamer + community profile learning.

---

## What the team is focused on now

Less new features. More:

- Cleaning up architecture
- Simplifying weak / messy flows
- **Reliability and "execution truth"**
- Reducing bugs and latency
- Preparing for many concurrent users
- **Reviewing the voice stack** — fast realtime model is great UX but expensive; custom STT → LLM → TTS would be cheaper but quality parity is hard

---

## Where the system is heading

A **broader creator operating layer**, not a single-purpose assistant:
- Stronger OBS / stream-setup help
- Richer Twitch-native management
- Better surfaced analytics
- Stronger memory- and profile-driven personalization
- More creator guidance / recommendations
- More **proactive** copilot behavior
- General agent capabilities — notes, scheduling, integrations, **deeper desktop / web workflows**
- More engagement features, widgets, overlays

---

## My read — strengths & open risks

**Strengths**
- Strong architectural separation (six clean layers).
- The memory hierarchy is the right shape — structured beats raw retrieval.
- Mixing fast / deterministic and reasoning paths is the right instinct.

**Risks worth discussing**
- **Execution truth** across Electron + backend + 4 external APIs is the silent killer in agent products.
- **Voice stack cost vs UX** — likely needs hybrid routing, not a single-vendor swap.
- **Multi-tenant scale** — every user is a long-lived stateful session (Electron ↔ backend), which is harder than stateless web traffic.
- **Local execution security** — Electron + arbitrary desktop actions = high blast radius; needs strict capability model.
