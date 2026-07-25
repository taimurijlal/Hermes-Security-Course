# Hermes as a Security Force Multiplier

*From Simple Automation to a Self-Improving Security Analyst*

## Part 1: Why Hermes Is Different as a Force Multiplier

> *"Hermes did not win on reasoning. It won on memory. What it shipped that the others did not was an automated learning loop."*
> — Communications of the ACM, May 2026

Every AI agent can help a security professional with research, monitoring, and reporting. OpenClaw can do it. Claude can do it. ChatGPT can do it. On day one, Hermes is no better than any of them.

But Hermes is the only major agent with a closed learning loop. After it completes a complex task, it writes a reusable skill file capturing how it solved the problem. The next time a similar task appears, it loads that skill and works faster and more consistently. It compounds.

This is transformative for security work. Your security workflows are repetitive by nature: the same daily briefing, the same triage process, the same weekly report, the same enrichment steps. A normal agent approaches each one cold, every time. Hermes captures the first run and improves on every subsequent run. Over weeks, it stops being a general assistant and becomes a specialised security analyst tuned to your exact environment.

> **⚠️ The same security rules still apply**
>
> Everything here uses public information and personal productivity tools. As established earlier in this resource, you should not connect Hermes to corporate systems, production infrastructure, identity providers, or sensitive data. Hermes is an intelligent but untrusted assistant that lives outside your corporate perimeter. The learning loop makes it more valuable there — not a reason to give it more access.

### The Three Stages of This Guide

This guide builds in three stages:

- **Part 2 — Simple Use Cases:** Things Hermes can do on day one. These work with any capable agent, but Hermes's persistent memory and natural-language scheduling make them smoother. Start here.
- **Part 3 — Acting Through Zapier MCP:** How to let the agent take safe, governed actions in your everyday apps — draft emails, log to a sheet, route alerts — through a single connection instead of wiring up individual MCP servers.
- **Part 4 — Learning Loop Use Cases:** Workflows designed specifically to exploit Hermes's self-improvement. This is where Hermes pulls ahead of every other agent, and where the actions from Part 3 become crystallised skills.

## Part 2: Simple Use Cases — Day One Value

These are the foundational workflows. They deliver value immediately, and — importantly — they are the workflows that will later trigger skill creation as Hermes learns your preferences. Run them a few times and Part 4 becomes possible.

> **🎯 Daily Threat Intelligence Briefing**
>
> *Hermes searches public sources every morning and delivers a briefing to your messaging app.*
>
> **Example prompt:**
> ```
> Every morning at 7am, search for cybersecurity news from the
> last 24 hours. Focus on new CVEs with CVSS 7+, CISA KEV
> additions, active ransomware campaigns, and major breaches.
> Categorise as Critical / High / Notable, include source links,
> and send me the briefing on Telegram.
> ```

> **🎯 CVE Lookup and Assessment**
>
> *Ask about any vulnerability and get a structured, analyst-ready answer.*
>
> **Example prompt:**
> ```
> Look up CVE-2026-XXXXX. Give me the CVSS score, affected
> products and versions, exploitation status, whether it's in
> the CISA KEV catalogue, any public proof-of-concept, and a
> plain-English recommendation on urgency.
> ```

> **🎯 Vendor Breach Monitoring**
>
> *Hermes watches public sources for incidents affecting your suppliers.*
>
> **Example prompt:**
> ```
> I'm going to give you a list of our vendors: Okta, CrowdStrike,
> Snowflake, Atlassian, Cloudflare. Every day, search public
> sources for any breaches, security incidents, or CVEs affecting
> them. If you find anything from the last 30 days, alert me on
> Telegram with a summary and source link. Otherwise, tell me
> all-clear.
> ```

> **🎯 Security Article Summarisation**
>
> *Turn any long security article or advisory into an analyst brief.*
>
> **Example prompt:**
> ```
> Read this article: [URL]. Give me a summary under 200 words:
> who and what, affected technologies, severity, and whether it
> requires action from us. Flag any indicators of compromise
> (IPs, domains, hashes) you find.
> ```

> **🎯 Weekly Security Content Digest**
>
> *A curated reading and listening list from public researchers and podcasts.*
>
> **Example prompt:**
> ```
> Every Friday, search for new content from the last week from
> Krebs on Security, Darknet Diaries, SANS Stormcast, John
> Hammond, and The Hacker News. Categorise as Breaking / Deep
> Dive / Career. Send me the digest on Telegram with the single
> must-read and must-listen highlighted.
> ```

> **Why these matter for what comes next**
>
> Each of these simple use cases is repetitive by design. That repetition is exactly what Hermes's learning loop needs. As you run these workflows and give feedback ("actually, always include the EPSS score too" or "put ransomware items at the top"), Hermes captures those preferences into skills. Part 4 shows what happens when you lean into that — and Part 3 first gives the agent the ability to act, not just report.

## Part 3: Acting Through Zapier MCP

Everything in Part 2 ends the same way: Hermes finds something and tells you about it. That is already useful. But the real force-multiplier effect appears when the agent can also act — draft the email, log the finding to a tracker, route the alert to the right channel, put the compliance date on a calendar.

Hermes can act through MCP servers. The Model Context Protocol lets the agent call external tools, and you can connect individual MCP servers one at a time — a Gmail server, a Slack server, a Google Sheets server, each configured separately with its own credentials in your Hermes config. That works, but it means every new capability is another server to install, another set of credentials sitting in your environment, and another thing to maintain.

> *Instead of wiring up a separate MCP server for every app, connect Zapier MCP once. One governed endpoint, thousands of actions, and your credentials never touch the agent.*

### 3.1 Why Zapier MCP Is the Better Interface

Zapier exposes its whole automation platform — thousands of connected apps — as a single MCP endpoint. You point Hermes at one Zapier MCP URL, and the agent gains access to whichever specific actions you have chosen to expose, across Gmail, Slack, Google Sheets, Calendar, Notion, Jira, and thousands of other apps. For a security professional running Hermes as an untrusted assistant outside the perimeter, this is not just more convenient — it is a materially better security posture.

| Direct MCP servers | Zapier MCP |
|---|---|
| One server to install and maintain per app | One connection for thousands of apps |
| Each server holds its own credentials in your environment | Zapier holds the OAuth tokens; the agent never sees them |
| Access is all-or-nothing per server's tool set | You choose exactly which actions are exposed, per app |
| Auditing is per-server, if it exists at all | Every action runs through Zapier's task history and logs |
| You vet and trust each server yourself | Zapier's app integrations are already vetted and maintained |

> **The security argument in one line**
>
> With direct MCP servers, a compromised or buggy server sees the credentials you gave it. With Zapier MCP, the agent asks Zapier to perform a specific pre-authorised action — "create a Gmail draft" — and Zapier performs it using credentials the agent can never read. You have put a governed broker between your untrusted assistant and your accounts. That is exactly the boundary you should be building throughout this resource.

### 3.2 Connecting Zapier MCP to Hermes

Setup is a one-time step. You generate a Zapier MCP endpoint, choose which actions to expose, and register the URL with Hermes as an MCP server.

> **📄 Registering Zapier MCP (`~/.hermes/config.yaml`)**
> ```yaml
> mcp_servers:
>   zapier:
>     url: "https://mcp.zapier.com/api/mcp/s/YOUR-ID/mcp"
>     # transport is SSE; the URL is generated at mcp.zapier.com
>     # where you also pick WHICH actions are exposed.
> # Or from the CLI:
> # hermes mcp add zapier --url https://mcp.zapier.com/...
> ```

> **⚠️ Scope it down at the Zapier end**
>
> The most important configuration is not in Hermes — it is in the Zapier dashboard, where you choose which actions the endpoint exposes. Expose only what you need, and prefer the safe verb every time: "Gmail: Create Draft", never "Gmail: Send Email". "Slack: Send Channel Message" scoped to one specific channel, never a broad post-anywhere action. The agent can only ever call actions you have deliberately switched on.

### 3.3 The Golden Rule: Drafts, Not Sends

Configure Zapier actions so the agent prepares work for your review rather than taking irreversible action:

- **Email:** "Create Draft" only. The agent writes; you press send. Never expose a send action.
- **Messaging:** Scope Slack or Teams to a single personal alerts channel. Reads and posts to that one channel, nothing else.
- **Trackers:** "Create Row" / "Create Item" are additive and safe. Avoid delete or bulk-update actions entirely.
- **Calendars:** Create events for compliance deadlines; do not grant delete or modify.

With that in place, here are the Zapier-powered versions of security workflows. Notice how each one takes a Part 2 use case that ended in "tell me" and extends it to "and act on it, safely."

> **🎯 Draft-Only Email Briefing Delivery**
>
> *Take the daily briefing from Part 2 and have Hermes leave it as a Gmail draft, ready for you to review and forward.*
>
> **Example prompt:**
> ```
> Run my morning threat briefing as usual. Then use Zapier to
> create a Gmail DRAFT addressed with the briefing as the
> body and subject "Threat Briefing — [today's date]"
> ```

> **🎯 Route Findings to a Personal Slack Channel**
>
> *When vendor monitoring finds something, push a clean alert to one dedicated channel instead of burying it in chat.*
>
> **Example prompt:**
> ```
> When your daily vendor check finds a confirmed incident, use
> Zapier to post a message to my #hermes-alerts Slack channel with:
> vendor name, one-line summary, severity, and the source link.
> One message per incident. If nothing is found, do not post.
> ```

> **🎯 Log CVEs to a Google Sheet Tracker**
>
> *Every CVE you assess becomes a row in a running spreadsheet — an audit trail you did not have to build.*
>
> **Example prompt:**
> ```
> After you assess a CVE, use Zapier to append a row to my
> "CVE Tracker" Google Sheet with columns: date, CVE ID, CVSS,
> EPSS, KEV status, in-scope (yes/no), your urgency call, and
> the source URL. Add the row; never edit or delete existing
> rows.
> ```

> **🎯 Put Compliance Dates on a Calendar**
>
> *When research surfaces a regulatory deadline, Hermes places it on your calendar so it does not get lost.*
>
> **Example prompt:**
> ```
> My company has to comply with PCI DSS and the EU AI Act. Check
> the internet and if you find a security or compliance deadline
> use Zapier to create a Google Calendar event on that date with
> the detail in the description
> ```

> **🎯 Open a Review Ticket — for You to Action**
>
> *Turn a significant finding into a draft task in Notion or Jira that a human then triages properly.*
>
> **Example prompt:**
> ```
> If a finding looks like it needs follow-up, use Zapier to
> create a Notion item in my "Security Review" database with a
> title, the summary, the severity, and the source link. Set it
> to status "Needs triage". Create the item only — do not change
> anything already in the database.
> ```

> **Why this belongs before the learning loop**
>
> Each of these actions is a step that used to be manual — copying a briefing into email, logging a CVE into a sheet, opening a ticket. Once Hermes can do them through Zapier, the whole workflow from detection to action lives in one place. And crucially: these action steps are exactly the kind of multi-step, tool-calling work that triggers Hermes's learning loop. "Search, assess, then log to the sheet in my format" is a five-tool-call task. Part 4 shows what happens when the agent starts crystallising these Zapier-powered workflows into skills.

> **🔒 Governance: the Zapier account is part of your attack surface**
>
> Use a dedicated Zapier account for the agent, not your primary one. Review the exposed action list periodically and remove anything you are no longer using. Keep the two-actions-per-call discipline: broad, chained automations are harder to reason about. And remember the boundary — Zapier is a governed broker, but it is still a third party holding tokens to your productivity apps. Expose the minimum, prefer read and draft over write and send, and keep it all on the public-information side of your perimeter.

## Part 4: Learning Loop Use Cases — Where Hermes Pulls Ahead

### First, a Refresher: How the Learning Loop Works

When Hermes completes a task that involved five or more tool calls, a background process reflects on the trajectory and, if the pattern is worth keeping, writes a reusable skill file to `~/.hermes/skills/`. The official name for this cycle is the closed learning loop:

> **🔄 The learning loop in action: the five phases**
>
> **1. Observe** — Hermes receives a task, checks its existing skills and memory for anything relevant, and gathers context.
>
> **2. Execute** — It completes the task using its 70+ tools — searching, reading, correlating, writing, and now acting through Zapier.
>
> **3. Reflect** — After a complex task (5+ tool calls), it reviews what actually worked and what did not.
>
> **4. Crystallise** — It writes the successful approach into a reusable Markdown skill file with a YAML header.
>
> **5. Reuse** — Next time a similar task appears, it loads that skill first. Reasoning becomes the fallback, not the default.

The practical effect: the workflows below start as detailed, effortful requests. After a few runs, Hermes has crystallised them into skills, and the same results come from a two-word prompt. You are not just automating a task — you are training a specialist.

### Learning Use Case 1: The Self-Improving Morning Briefing

Start with the simple daily briefing from Part 2. But this time, give feedback every day. Watch what happens over two weeks.

> **🔄 The learning loop in action: a briefing that learns your priorities**
>
> **Day 1** — You ask for a daily briefing. Hermes produces a generic one — all categories, alphabetical, no particular focus. You reply: "Good, but I care most about anything affecting cloud infrastructure and identity providers. Always lead with those."
>
> **Day 3** — You add: "Include the EPSS exploitation-probability score for every CVE, and skip anything below CVSS 6 unless it's actively exploited." Hermes adjusts.
>
> **Day 5** — After five days of this pattern, Hermes crystallises a skill: `daily-security-briefing.md`. It now encodes your stack focus, your scoring preferences, your severity threshold, and your preferred format.
>
> **Day 10** — You simply say "Morning briefing." Hermes loads the skill and produces exactly the briefing you spent a week shaping — cloud and identity first, EPSS scores included, low-severity noise filtered, formatted the way you like, and left as a Gmail draft via Zapier. No re-explanation.
>
> **The payoff** — A stateless agent would need the full instructions every single day forever. Hermes learned them once and applies them automatically. The briefing is now tuned to you.

> **📄 What Hermes wrote: `~/.hermes/skills/daily-security-briefing.md`**
> ```yaml
> ---
> name: daily-security-briefing
> description: Generate the operator's daily threat briefing
>   with their specific priorities and format.
> triggers: ["morning briefing", "daily briefing", "threat update"]
> ---
> ```
> ```markdown
> # Daily Security Briefing
>
> ## Learned Preferences
> - LEAD with cloud infrastructure and identity provider news
> - Include EPSS score for every CVE alongside CVSS
> - Skip anything below CVSS 6 UNLESS actively exploited
> - Group: Critical > High > Notable
> - Deliver as a Gmail draft via Zapier, under 500 words
>
> ## Workflow
> 1. Search NVD, CISA KEV, vendor advisories (last 24h)
> 2. Apply the severity filter above
> 3. Sort cloud/identity items to the top
> 4. Create Gmail draft via Zapier; do not send
> ```

### Learning Use Case 2: A CVE Triage Skill Built From Your Judgement

This is where the learning loop becomes genuinely powerful. You are not just teaching Hermes a format — you are teaching it your professional judgement.

> **🔄 The learning loop in action: encoding your triage logic**
>
> **The first few CVEs** — Each time a notable CVE appears, you ask Hermes to assess it, then you make the real decision: "This one is urgent because it's internet-facing and has a public PoC." "This one can wait — it needs local access we don't expose." "This one is noise, we don't run that product."
>
> **Hermes observes the pattern** — Over a dozen assessments, Hermes notices the factors you consistently weigh: internet exposure, PoC availability, KEV status, whether the product is in your stack, and whether authentication is required.
>
> **It crystallises a triage skill** — Hermes writes `cve-triage.md` encoding YOUR decision logic — not a generic CVSS lookup, but the specific prioritisation reasoning you applied over and over. The skill ends by logging the verdict to your CVE-tracker sheet through Zapier.
>
> **The result** — Now when you paste a CVE, Hermes does not just report the score. It applies your triage logic: "Urgent — internet-facing, public PoC, matches your stack. Recommend patching within 72h," and appends the row to your tracker automatically. It reaches your conclusion the way you would, and records it the way you would.

> **Why this is different from a prompt template**
>
> You could write a triage prompt template by hand. But you would have to know, in advance, exactly how you weigh every factor — and articulate it perfectly. The learning loop extracts that logic from your actual decisions over time. It captures the judgement you apply intuitively but might struggle to write down. That is the difference between configuring a tool and training an analyst.

### Learning Use Case 3: Compounding Vendor Intelligence

The persistent-memory half of Hermes pairs with the skill half here. Hermes does not just learn how to check vendors — it remembers what it has already found.

> **🔄 The learning loop in action: intelligence that accumulates**
>
> **Week 1** — You set up daily vendor monitoring. Hermes checks your supplier list and reports incidents, routing each one to your #sec-alerts channel via Zapier.
>
> **Week 3** — Hermes remembers it already reported a particular vendor's breach two weeks ago. Instead of re-alerting, it tracks the story: "Follow-up on the Snowflake incident I flagged on the 3rd — they've now confirmed the scope and released an advisory."
>
> **Week 6** — Hermes has crystallised a vendor-monitoring skill AND built a memory of each vendor's security history. It can now answer: "Which of our vendors has had the most incidents this quarter?" — because it has been accumulating that data all along.
>
> **The payoff** — A stateless tool re-checks everything from scratch daily and has no memory of what it found. Hermes builds a living intelligence picture of your supply chain that gets richer every week — exactly what a human threat-intel analyst would do, if they never forgot anything.

### Learning Use Case 4: The Async Overnight Analyst

Hermes runs as a persistent daemon and supports natural-language scheduling. Combine that with the learning loop and you get a security analyst that works while you sleep and improves while it works.

> **🎯 Fire-and-Forget Overnight Research**
>
> *Set a complex recurring task once. Hermes runs it, learns from it, and delivers results to your messaging app.*
>
> **Example prompt:**
> ```
> Every night at 2am, do a deep search for any new proof-of-concept
> exploits published for CVEs in the CISA KEV catalogue. For each
> one, assess whether it affects common enterprise technologies.
> Write up anything significant, leave it as a Gmail draft via
> Zapier, and ping my Slack when it's ready. Get better at
> spotting what I care about each time.
> ```

The phrase "get better at spotting what I care about each time" explicitly invites the learning loop. As you react to the overnight reports — marking some as useful, others as noise — Hermes crystallises what "significant" means for you specifically. The 2am report on night 30 is far sharper than the one on night 1.

> **The compounding effect**
>
> This is the single biggest advantage Hermes has over every other agent for security work. A stateless tool runs the same overnight job with the same generic quality forever. Hermes runs it, learns from your feedback, writes a skill, and runs a better version the next night. Thirty nights in, it is producing analyst-grade output tuned to your exact concerns — with zero additional effort from you.

### Learning Use Case 5: Building a Personal Detection Library

For detection engineers and SOC analysts, the learning loop can build something genuinely durable: a growing library of detection and analysis skills, each crystallised from real work.

> **🔄 The learning loop in action: from one-off analysis to reusable capability**
>
> **The task** — You ask Hermes to map a described attack technique to MITRE ATT&CK, identify the relevant data sources, and suggest detection logic. It takes several tool calls and some back-and-forth to get right.
>
> **The crystallisation** — Because the task was complex (5+ tool calls), Hermes writes `attack-mapping.md` capturing the successful approach: how you like techniques mapped, which data sources you prioritise, what format your detection pseudo-rules take.
>
> **The library grows** — Over months, this happens dozens of times across different security tasks: log-pattern analysis, IOC extraction, threat-actor profiling, report structuring. Each becomes a skill. Your `~/.hermes/skills/` directory becomes a personal security capability library.
>
> **The payoff** — You have built — without ever sitting down to write it — a documented, reusable playbook of how YOU do security analysis. It is portable (Markdown files), auditable (you can read every one), and shareable (compatible with the agentskills.io standard).

> **⚠️ Governance reminder**
>
> As established elsewhere in this resource: your growing skill library is proprietary institutional knowledge and a security surface in its own right. Audit it regularly, keep it under version control, and treat auto-generated skills as data that needs the same governance as any knowledge base. The learning loop is powerful precisely because it persists — which means a poisoned or drifted skill also persists. Review what your agent learns.

## Part 5: Putting It Together — The 90-Day Trajectory

The best way to understand Hermes as a force multiplier is to see how a security professional's relationship with it evolves over three months.

| Timeframe | What You Do | What Hermes Becomes |
|---|---|---|
| Week 1 | Run the simple use cases from Part 2. Connect Zapier MCP. Explain everything in detail each time. | A capable but generic assistant that can now also act — draft, log, alert — through one connection. |
| Weeks 2–4 | Give feedback. Correct its priorities. Refine its output. It starts crystallising skills. | Starting to specialise. The briefing knows your stack. CVE assessments apply your logic and log themselves. |
| Weeks 5–8 | Lean into async scheduling. Set overnight jobs. React to results. | A background analyst. It works while you sleep, acts through Zapier, and sharpens with every cycle. |
| Weeks 9–12 | Mostly two-word prompts. Occasional new tasks that spawn new skills. | A specialised security analyst tuned to your environment, with a growing skill library that documents how you work. |

> *"The distinction between reasoning and memory matters more in production than any benchmark score acknowledges. In environments where the same classes of problems repeat constantly — most enterprise environments — that is the gap that costs real time and real money."*
> — Communications of the ACM

Security work is one of the most repetitive knowledge disciplines there is. The same briefings, the same triage, the same enrichment, the same reports — over and over. That repetition is precisely what the learning loop is built to exploit. This is why Hermes, more than any other agent, is suited to becoming a genuine security force multiplier: not because it is smarter, but because it remembers, and it improves.

## Key Takeaways

1. On day one, Hermes is just another capable agent. Its advantage is what it becomes over time.
2. Start with simple, repetitive public-information use cases: briefings, CVE lookups, vendor monitoring, summarisation, content digests.
3. Connect Zapier MCP once instead of wiring up individual MCP servers — one governed endpoint, thousands of actions, and your credentials never touch the agent.
4. Prefer safe verbs: drafts not sends, create-row not delete, one scoped Slack channel. Zapier is a governed broker between your untrusted assistant and your accounts.
5. The learning loop (Observe > Execute > Reflect > Crystallise > Reuse) turns repeated tasks — including Zapier actions — into reusable skills after 5+ tool-call tasks.
6. Give feedback deliberately. Your corrections become encoded preferences and, eventually, encoded professional judgement.
7. The killer use cases exploit the loop: self-improving briefings, triage skills built from your decisions, compounding vendor intelligence, async overnight analysts that sharpen nightly, and a personal detection library.
8. The same security rules apply throughout: public information only, no corporate systems, treat both the skill library and the Zapier account as governed surfaces.
