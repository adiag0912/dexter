# Dexter — My Read of the System

This is a short summary of how I understood Dexter from the context pack. It is the picture I will be working from in the rest of the discussion.

## What Dexter is

Dexter is a desktop AI helper built for streamers. The user can talk to it or type to it, and it answers in real time. It also stays aware of what is happening on the stream, and on top of that, it can actually do things across the streamer's computer and online accounts. So it is not a chatbot in the usual sense. It is closer to a real assistant, because it acts and not just talks.

## What makes Dexter interesting

There are three things about Dexter that stand out to me. The first is that it works across the user's computer, the cloud, and outside services all at the same time. It is not just a website with a backend. It runs on the user's machine, talks to a server, stores data in a database, and connects to OBS, Twitch, Telegram, and the web — and all of these have to work together for every single request.

The second thing is that Dexter actually does things. Most AI tools just talk, but Dexter goes further. It changes scenes in OBS, posts messages, runs giveaways, and plays music. That is what makes it a real co-pilot instead of a chatbot. It also means every action needs to be checked, so when Dexter says it did something, the user can trust it.

The third thing is that Dexter learns over time. It keeps short notes during the stream, longer notes after the stream, and slowly builds up a picture of the streamer and their community. That layered way of remembering things is more useful than dumping everything into one big search. (Maybe we can add Obsidian second brain here.)

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

## What Dexter is built from

Dexter is built as an Electron desktop app, with a React frontend for the screen and a Node and TypeScript backend for the server-side logic. Everything is stored in Supabase, which sits on top of Postgres. It uses a few different AI models for different jobs: a fast one for live conversation, a smarter one for thinking through harder questions, and a cheaper one for cleaning up data afterwards. On top of that, it connects to outside services like the user's microphone, OBS, Twitch, Telegram, and the web.

## What happens when the user talks to Dexter

When the user speaks or types, the voice layer turns the speech into text and handles things like the user cutting in mid-sentence. The desktop app then passes the request over to the server. The server figures out what kind of request this is — whether it is a question, a command, or something that needs the desktop side to take action. If the user's computer needs to do something, the desktop side handles that part. The server wraps things up, and Dexter speaks or writes back to the user.

A request can take a few different shapes. It might just be a normal answer, or a command that runs on the user's computer, or a normal back-and-forth conversation, or a quick shortcut for very simple commands.

## How memory works

```
Everything that happens   ──shorten──►   Live notes during the stream
on the stream             ──summarize─►   Highlights after the stream
                          ──learn────►   Long-term notes about the
                                          streamer and their viewers
```

When Dexter is talking to the user, it does not dig through everything that ever happened. Instead, it gets a clean, ready-made summary so it can respond quickly.

## What Dexter can do today

Today Dexter can talk and listen, control OBS and the desktop, manage Twitch and moderation, run giveaways, timers, and polls, and save reusable shortcuts. It can also post to Telegram, open and use the web, control music, and show stream and channel stats. On the memory side, it keeps live notes, remembers things long-term, and learns about both the streamer and their community.

## What the team is working on right now

The current focus is less about adding new features and more about making what already exists solid. That means cleaning up the code, fixing parts that are messy or fragile, and making sure that when Dexter says it did something, it really did happen. The team is also making things faster, less buggy, and ready for many users at the same time. One area under review is the voice setup. The current one feels great to use but is expensive to run. Building a custom version would save money but might not feel as good, so it is a real tradeoff.

## Where Dexter is heading

The longer-term direction is for Dexter to become a broader assistant for creators, not just a single tool. That means better OBS and stream-setup help, more Twitch features built in, and better stats shown to the user. It also means more personal, memory-aware responses, more advice and recommendations, and Dexter speaking up on its own when it has something useful to say. Beyond that, the team plans to add general assistant features like notes, scheduling, integrations, and automations across apps, along with more on-screen widgets and viewer-engagement features.

## My take on what is strong and what is risky

There are a few things I think Dexter is getting right. The system is split into clean layers, which makes it easier to change one part without breaking the others. The way memory is organised is the right shape, because neat summaries are more useful than raw search. And mixing fast simple paths with slower thinking paths is a good instinct.

There are also a few risks worth talking about. The biggest one is making sure things really happened, because Dexter touches the user's computer, a server, and several outside services, and any one of them can quietly fail. Voice cost versus how it feels is another one — the answer is probably a mix of approaches rather than picking just one. Scaling to many users at the same time is harder for Dexter than for a normal website, because every user keeps an open connection between their desktop and the server. And finally, security on the desktop side matters a lot, because a program that can do anything on the user's computer needs careful limits on what it is actually allowed to do.
