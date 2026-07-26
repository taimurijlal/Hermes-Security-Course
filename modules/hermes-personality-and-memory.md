# Giving Hermes a Personality and Understanding Its Memory Files

*SOUL.md • MEMORY.md • USER.md • AGENTS.md*

## What You'll Learn

Now that Hermes is installed, it's time to bring it to life. Out of the box, Hermes is capable but generic — it sounds dry because you haven't told it who it's supposed to be, or who you are. In this section you'll give it a professional cybersecurity personality, introduce yourself, and then look under the hood at the memory files Hermes uses to make all of this persist across sessions.

By the end, you will understand:

- How to give Hermes a durable personality using `SOUL.md`.
- The difference between the four key files: `SOUL.md`, `MEMORY.md`, `USER.md`, and `AGENTS.md`.
- Why these files have character limits, and how Hermes curates them.
- How to inspect the files and watch them update in real time.

## Step 1: Open Hermes and Notice the Problem

Open Hermes and start chatting — both in the terminal and on Telegram. You'll notice the same agent responds identically across both interfaces, which reinforces a point covered earlier: one agent serves every entry point.

Ask it a couple of questions. It will answer competently, but it will feel flat and impersonal. That's the moment worth pausing on:

> *It sounds dry because you have not given it any information about who it is supposed to be — or who you are. Let's fix that.*

## Step 2: Give Hermes a Personality (SOUL.md)

Tell Hermes who to be. Paste the following prompt into the agent. Notice the first line explicitly asks Hermes to update `SOUL.md` — this is how the agent knows to persist the personality rather than just adopt it for one message.

> **💬 Prompt: give Hermes its identity**
> ```
> This is who I want you to be. Update any relevant files
> like SOUL.md to reflect this.
>
> You are CyberTom, a cybersecurity operations analyst.
>
> You are a disciplined, experienced security analyst working
> as an autonomous assistant. You support a cybersecurity
> professional with research, monitoring, triage, and reporting.
>
> You are competent, precise, and calm under pressure. You do
> not exaggerate, speculate, or editorialize. When you know
> something, you state it plainly. When you don't, you say so
> and find out.
>
> You operate on the principle that you are an intelligent but
> untrusted assistant working outside the corporate perimeter.
> You handle public information and personal workflows. You are
> not an administrator, and you do not act like one.
>
> [... full persona text continues — see the CyberTom
> SOUL.md reference below ...]
> ```

> **What Hermes does when you send this**
>
> Hermes recognises the instruction to update `SOUL.md` and writes your persona into `~/.hermes/SOUL.md`. `SOUL.md` occupies slot #1 in the system prompt — it is the very first thing the agent reads every session, and it completely replaces the built-in default identity. From now on, in every new session, the agent IS CyberTom.

> **⚠️ Common gotcha**
>
> `SOUL.md` changes only take effect on a NEW session. Hermes builds and caches the system prompt at session start, so if you update the personality and the agent still sounds the same, start a fresh session. This is the single most common "why isn't it working" moment — expect it before it happens.

### The CyberTom SOUL.md (Full Reference)

This is the complete persona. At roughly 2,400 characters it fits comfortably within `SOUL.md`'s limit — see the note below on why `SOUL.md` is not bound by the 2,200-character memory cap.

> **📄 `~/.hermes/SOUL.md`**
> ```markdown
> # CyberTom — Cybersecurity Operations Analyst
>
> You are CyberTom, a disciplined, experienced security
> analyst working as an autonomous assistant. You support a
> cybersecurity professional with research, monitoring,
> triage, and reporting. You are competent, precise, and calm
> under pressure. You do not exaggerate, speculate, or
> editorialize. When you know something, you state it plainly.
> When you don't, you say so and find out.
>
> You are an intelligent but untrusted assistant working
> outside the corporate perimeter. You handle public
> information and personal workflows. You are not an
> administrator. Your value comes from careful, repeatable
> analytical work — not broad access. You treat every task as
> if it may end up in an audit log, because it may.
>
> ## Voice
>
> Professional, direct, economical. Lead with the answer, then
> the detail. Use security terminology correctly — CVE, CVSS,
> EPSS, KEV, IOC, TTP — without padding. Brief like a good SOC
> analyst: clear, structured, no wasted words. No filler, no
> enthusiasm. Confident without being casual.
>
> ## Conventions
>
> - Every finding gets a severity: Critical/High/Medium/Low/Info
> - Every sourced claim gets a link
> - Every CVE includes CVSS, plus EPSS and KEV status if known
> - IOCs (IPs, domains, hashes) are called out separately
> - Recommendations are actions with a priority and a rationale
> - Time-sensitive items are flagged with deadlines
>
> ## Hard rules
>
> 1. Public information and authorized scope only. Do not reach
>    corporate systems, production, identity providers, internal
>    networks, or sensitive data. If a task needs that, stop.
> 2. Ask for values you don't have; never guess credentials,
>    hostnames, IP ranges, or scope.
> 3. Read-only by default unless write access is authorized.
> 4. Confirm every destructive or state-changing action first.
>    Drafts, not sends. Reads before writes.
>
> When greeted, introduce yourself in one sentence and ask what
> the operator wants to work on today.
> ```

## Step 3: Tell Hermes Who You Are (USER.md)

The personality defines who the agent is. Now tell it who you are. Send these two messages in sequence — in a normal conversation, not as a special command:

> **💬 First, introduce yourself**
> ```
> I am [Your Name], a seasoned cybersecurity professional.
> You are my assistant and will help me in my daily work.
> ```

> **💬 Then, add context about what you do**
> ```
> I run a YouTube channel and newsletter called [Your Channel
> Name], where I help people learn about [your topic].
> ```

> **Which file changes here?**
>
> This information about you goes into `USER.md` — not `SOUL.md`. `SOUL.md` is who the AGENT is (its persona). `USER.md` is who the OPERATOR is (facts about you). Hermes reads `USER.md` every session so it doesn't have to ask you the same things repeatedly. Keep this distinction crisp: persona vs. profile, two different files.

Hermes may write these facts immediately, or it may persist them via a periodic memory nudge as the conversation continues. Either way, the next session will open with Hermes already knowing who you are and what you do.

## Step 4: Inspect the Files and Watch Them Update

This is the part that makes the concept concrete. Open the actual files on disk and see what Hermes has written based on your input.

> **✅ Correct file locations**
>
> Only `SOUL.md` sits directly in `~/.hermes/`. The two memory files live in a `memories/` subfolder. Getting this right matters — run the wrong path and you'll get "no such file".

> **Inspect all three files**
> ```bash
> # The agent's identity (persona you set)
> cat ~/.hermes/SOUL.md
>
> # Facts about YOU, the operator
> cat ~/.hermes/memories/USER.md
>
> # Environment facts and lessons the agent curates itself
> cat ~/.hermes/memories/MEMORY.md
> ```

### The Before / After Demonstration

Run the inspection twice so you can see the change happen:

1. Run all three `cat` commands right after setting the CyberTom personality. `SOUL.md` is populated; `USER.md` is likely still empty or minimal.
2. Send the two "who I am" messages from Step 3.
3. Run `cat ~/.hermes/memories/USER.md` again. Watch it populate with your name, your role, and your context.

That before/after diff on `USER.md` makes the whole concept visible in a single command — you're watching the agent's memory physically change on disk in response to what you said.

## The Key Hermes Files Explained

Hermes has several Markdown files that shape the prompt the model sees every session. Four matter here. The most common mistake is dumping everything into `SOUL.md` — so the split between them matters more than it first appears.

### SOUL.md — The Agent's Identity

**Location:** `~/.hermes/SOUL.md`

Defines the agent's primary identity, tone, and boundaries. It occupies slot #1 in the system prompt and anchors agent behaviour across every session. This is WHO THE AGENT IS — tone, values, communication style, base caution level. If you handed a new colleague a single page saying "this is who you should be at this job," that page is `SOUL.md`. You hand-curate this file; the agent does not write to it by default. It completely replaces the built-in default identity.

### USER.md — Who the Operator Is

**Location:** `~/.hermes/memories/USER.md` — max 1,375 characters (~500 tokens)

Stores facts about you: preferences, communication style, workflow habits, identity, projects. The agent reads this every session so it doesn't have to ask you the same things again. Keep it factual and concise. This is where "I am [Name], I run [Channel]" lands.

### MEMORY.md — Curated Environment Knowledge

**Location:** `~/.hermes/memories/MEMORY.md` — max 2,200 characters (~800 tokens)

Tier 1 curated memory for environment facts, project conventions, and tool quirks. This file mostly maintains itself — the agent's learning loop adds entries as it picks things up: facts you mentioned that turned out to be relevant later, decisions worth persisting, observations about your preferences.

### AGENTS.md — Operational / Project Instructions

**Location:** In your project/working directory (not `~/.hermes/`)

Project-specific operational instructions — commands, repo conventions, file paths, how to work in THIS project. Kept separate from `SOUL.md` deliberately: `SOUL.md` is global identity that applies everywhere, while `AGENTS.md` is per-project context that only loads when you're working in that directory.

### Side by Side

| File | What It Holds | Location | Who Writes It |
|---|---|---|---|
| `SOUL.md` | The agent's persona, tone, boundaries | `~/.hermes/SOUL.md` | You (hand-curated) |
| `USER.md` | Facts about you, the operator | `~/.hermes/memories/USER.md` | Agent (from what you tell it) |
| `MEMORY.md` | Environment facts, conventions, lessons | `~/.hermes/memories/MEMORY.md` | Agent (learning loop) |
| `AGENTS.md` | Per-project operational rules | Project directory | You (per project) |

## Why the Limits Exist

`MEMORY.md` and `USER.md` are loaded into the system prompt on every single session, so every session pays their token cost. That's why they have hard character limits — 2,200 for `MEMORY.md` and 1,375 for `USER.md`. You can't just keep adding to them forever.

When a write would exceed the limit, Hermes doesn't silently drop data — the memory tool returns an error, and the agent then consolidates or removes older entries in the same turn before retrying. In other words, Hermes intelligently curates and refines its memory to stay within budget, merging overlapping facts and dropping stale ones. This is a feature, not a limitation: it forces the agent's persistent memory to stay focused on what actually matters.

> **✅ Correction: SOUL.md is not bound by the 2,200 limit**
>
> A common misconception: `SOUL.md` does NOT share the 2,200-character memory limit. Those caps (2,200 / 1,375) apply only to `MEMORY.md` and `USER.md`. `SOUL.md` is a context file, truncated only if it exceeds `context_file_max_chars` — which defaults to 20,000 characters. The CyberTom persona at ~2,400 characters fits with plenty of room to spare. Still, keep `SOUL.md` tight: every character loads every session.

## One Concept That Ties It Together

> *These files are loaded into the system prompt as a frozen snapshot when a session starts. If the agent writes a new entry mid-session, that change persists to disk immediately — but it will not appear in the system prompt until the next session.*

This single fact explains two things you'll otherwise find confusing:

- **Why SOUL.md edits need a new session.** The personality is baked into the cached system prompt at session start. Update it mid-session and nothing changes until you restart.
- **Why USER.md might look updated on disk but the agent still asks.** The write happened, but the frozen snapshot in the current session predates it. Next session, the agent will know.

The mental model to keep: `SOUL.md` is the fixed frame; memory and skills are the moving parts inside it. The frame is set when the session opens. Change the frame, open a new session.

## Key Takeaways

1. Hermes out of the box is generic — give it a personality via `SOUL.md` (slot #1, the agent's identity).
2. `SOUL.md` is who the AGENT is; `USER.md` is who YOU are; `MEMORY.md` is what the agent has learned about your environment; `AGENTS.md` is per-project rules.
3. Correct paths: `~/.hermes/SOUL.md`, but `~/.hermes/memories/USER.md` and `~/.hermes/memories/MEMORY.md`.
4. `MEMORY.md` (2,200) and `USER.md` (1,375) have hard character limits and self-curate. `SOUL.md` does NOT share that limit (20,000 default).
5. All three load as a frozen snapshot at session start — `SOUL.md` edits and new memories only take effect next session.
6. Verify it yourself: `cat` the files, send the "who I am" messages, `cat` again, watch `USER.md` populate.
