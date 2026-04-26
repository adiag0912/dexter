# Dexter — My Read of the System

> A short summary of how I understood Dexter from the context pack. This is what I'll be reasoning from in the rest of the discussion.

---

## What Dexter is, in one line

A **desktop AI helper for streamers** — you talk or type to it, it answers in real time, it knows what's happening on your stream, and it can actually do things for you across your computer and your accounts.

Not just a chatbot. A real assistant — it acts, it doesn't just talk.

---

## Three things that make Dexter interesting

| | Why it matters |
|---|---|
| **It works across your computer, the cloud, and outside services together** | It's not just a website with a backend. It runs on your machine, talks to a server, stores data in a database, and connects to OBS, Twitch, Telegram, and the web — all working together for every single request. |
| **It has to actually do things, not just answer** | When Dexter says "I muted that user," it really has to have happened. Most AI products only have to talk. Dexter has to *do*, which is much harder to get right. |
| **It learns over time** | It keeps short notes during the stream, longer notes after the stream, and slowly builds up a picture of the streamer and their community. That layered memory is more useful than dumping everything into one big search. (Maybe we can add Obisidan second brain here.) |

---

## Architecture at a glance

```
   ┌────────────────────────────────────────────────────────────┐
   │                       USER (voice / text)                   │
   └───────────────────────────────┬────────────────────────────┘
                                   │
                  ┌────────────────▼────────────────┐
                  │     Realtime Voice Layer         │  ← listens, speaks back
                  └────────────────┬────────────────┘
                                   │
                  ┌────────────────▼────────────────┐
                  │   Frontend Runtime (Electron)    │  ← the desktop app itself
                  └─────┬───────────────────────┬───┘
                        │                       │
        ┌───────────────▼──────────┐   ┌────────▼───────────────┐
        │  Backend Orchestration   │   │  Local Execution Layer │
        │  (the "brain" — decides  │   │  (does things on your  │
        │   what to do)            │   │   actual computer)     │
        └───────┬──────────────────┘   └────────────────────────┘
                │
   ┌────────────▼─────────────┐    ┌──────────────────────────┐
   │   Feature Modules         │◄──►│   Database                │
   │   (giveaways, moderation, │    │   (Supabase / Postgres)   │
   │   templates, analytics…)  │    │   stores everything       │
   └──────────┬───────────────┘    └──────────────────────────┘
              │
   ┌──────────▼─────────────────────────────────────────────────┐
   │   Outside services: Twitch, Telegram, web, music, etc.      │
   └────────────────────────────────────────────────────────────┘
```

---

## The pieces it's built from

- **Electron** — turns a web app into a desktop app
- **React** — the visible part you click and read
- **Node / TypeScript** — the server-side code
- **Supabase / Postgres** — where everything is stored
- **Several different AI models** — a fast one for live conversation, a smarter one for thinking, a cheap one for cleaning up data afterwards
- **Outside connections** — your microphone, OBS, Twitch, Telegram, and the web

---

## What happens when you talk to Dexter

1. You speak or type
2. The voice layer turns your speech into text and handles you cutting in
3. The desktop app passes the request to the server
4. The server figures out: is this a question? a command? something the desktop has to do?
5. If it needs your computer to do something, the desktop side handles it
6. The server wraps things up
7. Dexter speaks or writes back

A request can be: just an answer, a command run on your computer, a normal chat, or a quick shortcut for simple commands.

---

## How memory works

```
Everything that happens   ──shorten──►   Live notes during the stream
on the stream             ──summarize─►   Highlights after the stream
                          ──learn────►   Long-term notes about the
                                          streamer and their viewers
```

When Dexter is talking to you, it doesn't dig through everything that ever happened. It gets a clean, ready-made summary so it can respond fast.

---

## What Dexter can do today

Talk and listen · control OBS and your desktop · manage Twitch and moderation · run giveaways, timers, polls · save reusable shortcuts · post to Telegram · open and use the web · control music · show stream and channel stats · keep live notes · remember things long-term · learn about the streamer and their community.

---

## What the team is working on right now

Less new features. More making what's there solid:

- Cleaning up the code
- Fixing messy or fragile parts
- **Making sure things actually happen when Dexter says they did**
- Making it faster and less buggy
- Making it ready for lots of users at the same time
- **Rethinking the voice setup** — the current one feels great but is expensive; building their own version would save money but might feel worse

---

## Where it's heading

A **broader assistant for creators**, not just one tool:

- Better OBS and stream-setup help
- More Twitch features built in
- Better stats shown to the user
- More personal, memory-aware responses
- More advice and recommendations
- Dexter speaking up on its own when useful, not just when asked
- General assistant features — notes, scheduling, integrations, **automations across apps**
- More on-screen widgets and viewer-engagement features

---

## My take — what's strong, what's risky

**Strengths**
- The system is split into clean layers, which makes it easier to change things without breaking everything else.
- The way memory is organised is the right shape — neat summaries beat raw search.
- Mixing fast simple paths and slower thinking paths is the right instinct.

**Risks worth talking about**
- **Making sure things really happened.** Dexter touches your computer, a server, and four outside services. Any one of them can quietly fail. This is the silent killer in AI assistants.
- **Voice cost vs. how it feels.** Probably needs a mix of approaches, not picking one or the other.
- **Lots of users at once.** Every user keeps an open connection between their desktop and the server — that's harder to scale than a normal website.
- **Security on the desktop side.** A program that can do anything on your computer needs careful limits on what it's actually allowed to do.
