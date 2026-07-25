# Hermes Skills and the Self-Improvement Loop

*What skills are, where they live, and how the agent writes its own — with the trust model, live demos, and governance controls a security-minded operator needs.*

> A skill is content that goes directly into your agent's prompt. Installing one is closer to importing a system prompt than to running `pip install`.

---

## Part 1 — What Are Skills?

A Hermes skill is a reusable workflow instruction written in Markdown. It tells the agent what a task is, when to use this approach, which steps to follow, which tools to reach for, and how to verify the result worked.

Skills do not add code to Hermes and do not replace the model or the core tools. They are procedural memory — the difference between an analyst who works out a process from scratch every time and one who follows a proven runbook.

### 1.1 The Three Things a Skill Is Not

- **Not a plugin.** No code changes to the agent, no Python, no registration. A skill is a Markdown file.
- **Not a tool.** Tools are what the agent can *do* (run a command, fetch a URL). Skills tell it *how* and *when* to use those tools for a specific job.
- **Not a template.** The agent adapts variables inside a skill to new context rather than replaying it verbatim, and it can improve a skill when it finds a better approach.

### 1.2 Why This Matters for Security Work

A skill is a runbook that executes. For a security professional, that framing is the whole point: your CVE triage process, your enrichment workflow, your briefing format, your incident write-up structure — each is a repeatable procedure that currently lives in your head or in a wiki nobody reads. A skill turns it into something the agent applies consistently.

> **🔒 Security lens**
> It also means a skill is a high-privilege surface. Its content lands directly in the prompt, and a skill directory can carry executable scripts. Treat installing a third-party skill with the same caution you would apply to importing a system prompt written by a stranger — because that is exactly what it is.

---

## Part 2 — Where Are Skills Stored?

All active skills live in one place on your machine:

```
~/.hermes/skills/
├── <category>/
│   └── <skill-name>/
│       ├── SKILL.md       # Required: the instructions
│       ├── scripts/       # Optional: helper code
│       ├── references/    # Optional: reference docs
│       ├── templates/     # Optional: reusable templates
│       ├── examples/      # Optional: worked examples
│       └── assets/        # Optional: supporting files

# Example:
~/.hermes/skills/security/cve-triage/SKILL.md
```

Bundled skills ship inside the Hermes repository under `skills/`. On install and on every `hermes update`, a sync pass copies them into `~/.hermes/skills/` and records a manifest — while respecting your local deletions and edits. If you delete a bundled skill and want it back, `hermes skills reset <name> --restore` brings it home.

### 2.1 What Is Actually on Your Machine Right Now

Run this on a fresh install and you will see the bundled categories:

```
$ ls ~/.hermes/skills/
apple  creative  email  media
autonomous-ai-agents  data-science  github  mlops
computer-use  dogfood  hermes-desktop-plugins  note-taking
productivity  research  smart-home  social-media
software-development  yuanbao
```

Named bundled skills inside those categories include `plan`, `systematic-debugging`, `test-driven-development`, `github-pr-workflow`, `github-code-review`, `research/arxiv`, and `productivity/ocr-and-documents`. There is also an `autonomous-ai-agents` category containing a skill about Hermes itself — the agent ships with documentation on how to operate the agent.

> **🔎 Notice what is missing**
> Look at that category list again. There is no security category. The bundled skills skew heavily toward software development, ML/data science, and personal productivity. Out of the box, Hermes knows how to run a test-driven development loop and manage a GitHub PR. It does not know how to triage a CVE, profile a threat actor, or structure an incident write-up. For a cybersecurity professional this is the single most important observation in this section: your security skills will come from the official optional tier, the trusted registries, or — mostly — from the self-improvement loop writing them as you work. Nobody hands you a security library. You build it.

### 2.2 The Skills Hub: A Package Manager for Agent Knowledge

> If `pip` installs code and `npm` installs packages, the Hub installs procedures.

The Skills Hub is Hermes's built-in skill marketplace. It is not a website you browse and download from by hand — it is a set of CLI commands that search, inspect, install, update, and publish skills from several registries at once, all following the [agentskills.io](https://agentskills.io) open standard.

You search for a capability, inspect what it actually does, install it, and it lands in `~/.hermes/skills/` ready to use. Later you can check whether the upstream version has changed and update it.

**Discovery**

```bash
hermes skills browse         # browse everything available
hermes skills search QUERY   # search the hub by keyword
hermes skills list           # what do I already have installed?
```

**Evaluation and installation**

```bash
hermes skills inspect ID     # preview WITHOUT installing --- always do this
hermes skills install ID     # install a skill
hermes skills uninstall NAME # remove a hub-installed skill

# The ID can be a hub identifier OR a direct SKILL.md URL:
hermes skills install skills-sh/anthropics/skills/pdf
hermes skills install https://example.com/path/SKILL.md --name my-skill
```

**Maintenance and publishing**

```bash
hermes skills check          # which installed skills changed upstream?
hermes skills update         # update the outdated ones
hermes skills update <name>  # update just one
hermes skills config         # enable/disable skills per platform
hermes skills publish PATH   # publish your own skill to the registry
```

### Taps: Running Your Own Trusted Registry

A tap is an additional GitHub repository registered as a skill source. This is the mechanism most worth caring about: instead of pulling from arbitrary community repos, you publish a vetted internal repository and install from there.

```bash
hermes skills tap list                     # show configured taps
hermes skills tap add myorg/skills-repo    # add (default path: skills/)
hermes skills tap remove myorg/skills-repo

# Taps are stored in ~/.hermes/.hub/taps.json
```

> **🔒 Security lens**
> New taps are assigned COMMUNITY trust by default — adding your own repo does not automatically make it trusted. Skills installed from it still run through the standard security scan and show the third-party warning on first install. That is the correct default: your repo has to earn trust through review, not through configuration. Your own vetting is the trust signal, and the scanner still runs as a backstop.

### 2.3 The Four Sources and Their Trust Levels

Skills come from four places, and Hermes assigns each a different trust level that governs what the security scanner will let through.

| Trust level | Source | Scanning | Install policy |
|---|---|---|---|
| **builtin** | Ships with Hermes (repo `skills/`) | None — always trusted | Allowed |
| **official** | `optional-skills/` in the repo, reviewed by Nous Research | Built-in trust; no third-party warning | Allowed |
| **trusted** | Known registries: `openai/skills`, `anthropics/skills`, `huggingface/skills`, `NVIDIA/skills` | Scanned | More permissive than community |
| **community** | skills.sh, custom GitHub repos, most marketplaces — everything else | Scanned | Non-dangerous findings overridable with `--force`; dangerous stays blocked |

**builtin — already on your machine.** Covered in 2.1 above. No scanning, always trusted, synced on every `hermes update`. Development and productivity heavy; no security category.

**official — Nous-reviewed optional skills.** These live in `optional-skills/` in the Hermes repository. Reviewed by the Nous Research team, carrying built-in trust and no third-party warning panel. Not installed by default — you opt in.

```bash
hermes skills install official/security/1password
hermes skills install official/migration/openclaw-migration
hermes skills install official/mcp/fastmcp
hermes skills install official/blockchain/solana
hermes skills install official/mlops/flash-attention
```

Note the first two. `official/security/1password` is credential management; `official/migration/openclaw-migration` moves an existing OpenClaw setup across to Hermes — a path worth knowing if you're migrating from OpenClaw.

**trusted — known registries.** Four registries carry trusted status with a more permissive scanning policy than arbitrary community sources: `openai/skills`, `anthropics/skills`, `huggingface/skills`, and `NVIDIA/skills`.

```bash
# Inspect first, as always
hermes skills inspect skills-sh/anthropics/skills/pdf

# The example the official docs themselves use:
hermes skills install skills-sh/anthropics/skills/pdf
```

The Anthropic collection is the one worth exploring first — it includes a large body of structured cybersecurity skills mapped to MITRE ATT&CK. That is the closest thing to an off-the-shelf security library in the ecosystem, and it partially answers the gap identified in 2.1.

> **A provenance detail worth knowing**
> The `NVIDIA/skills` registry ships signed skills — a `skill.oms.sig` signature file plus a governance `skill-card.md` — and the sync pipeline drops anything missing them. It is the only registry doing cryptographic provenance today, and it is hardcoded rather than pluggable. This is what "signed and governed" looks like in the agent skill ecosystem today, and it is currently the exception rather than the rule.

**community — everything else.** skills.sh, custom GitHub repos, and most marketplaces. Scanned, and blocked on caution-level findings unless you explicitly pass `--force`. Dangerous verdicts stay blocked regardless.

```
$ hermes skills install mvanhorn/last30days-skill --force
Fetching: mvanhorn/last30days-skill
Quarantined to .hub/quarantine/last30days-skill
Running security scan...
Decision: BLOCKED --- Blocked (community source + dangerous
verdict, 56 findings). --force does not override a dangerous
verdict.
```

> **🔒 Security lens**
> That is a real, publicly documented block. Note three things: the skill was quarantined before scanning rather than installed first; `--force` was passed and still did not get through; and 56 findings triggered the dangerous verdict. Then look at the quarantine directory yourself — this single terminal output teaches the trust model better than any table.

At the time of writing the Hub aggregates several hundred skills across these registries. Counts drift; the trust tiers do not.

---

## Part 3 — How the Self-Improvement Loop Works

This is the single most differentiating architecture in Hermes — and the single most novel attack surface in any agent framework.

```
The Closed Learning Loop

Observe → Execute → Reflect → Crystallise → Reuse
  ↑                                          │
  └──────────── Automatically invoked ───────┘
```

### 3.1 The Judgement Step: Is This Worth Keeping?

The agent decides whether the experience contains valuable, reusable knowledge. This judgement is what separates Hermes from agents that log everything and retrieve nothing useful.

The official documentation lists four conditions that trigger skill creation:

- After completing a complex task (5+ tool calls) successfully
- When it hit errors or dead ends and found the working path
- When the user corrected its approach
- When it discovered a non-trivial workflow

For memory, the agent is documented to reject trivial data, easily searchable facts, large code blocks, and anything ephemeral to a single session. The capacity limits enforce that discipline: `MEMORY.md` caps at 2,200 characters and `USER.md` at 1,375.

### 3.2 Create or Update Skill

If the answer is yes, Hermes writes or improves a skill using the `skill_manage` tool — its procedural memory. The house format has four sections: **When to Use** (trigger conditions), **Procedure** (numbered steps), **Pitfalls** (known failure modes and fixes), and **Verification** (how to confirm it worked).

Crucially, updates are not rewrites. The `skill_manage` tool has six actions, and the documentation is explicit that `patch` is preferred over `edit`, because only the changed text appears in the tool call — a genuine token-efficient diff rather than a full-document replacement.

| Action | Use for | Security note |
|---|---|---|
| `create` | New skill from scratch | New file appears in the skills tree |
| `patch` | Targeted fixes — the preferred method | Small diffs; easy to review, easy to miss |
| `edit` | Major structural rewrites | Whole-document replacement |
| `delete` | Remove a skill entirely | The agent can delete its own knowledge — unless pinned |
| `write_file` | Add or update supporting files | Can write executable scripts |
| `remove_file` | Remove a supporting file | — |

> **🔒 Security lens**
> Read that table as a permissions matrix, because that is what it is. `skill_manage` is a full read-write-delete interface to the directory that shapes every future session — and `write_file` means an agent-authored skill can carry agent-authored executable scripts. The one brake available is pinning, covered in Part 6.

---

## Part 4 — The SKILL.md Format

The official format — which agent-created skills also follow. Here it is with a security workflow rather than a devops one, so you can see the shape you'll actually build:

```
📄 ~/.hermes/skills/security/cve-triage/SKILL.md
---
name: cve-triage
description: Assess a CVE against our stack and assign urgency
version: 1.0.0
metadata:
  hermes:
    tags: [cve, vulnerability, triage, nvd, kev]
    category: security
---

# CVE Triage

## When to Use
Operator supplies a CVE ID and asks for an assessment,
or a monitoring job surfaces a new high-severity CVE.

## Procedure
1. Pull CVSS base score and vector from NVD
2. Check EPSS exploitation probability
3. Check CISA KEV catalogue for active exploitation
4. Search for public proof-of-concept code
5. Compare affected products against the operator's
   declared technology list
6. Assign urgency: internet-facing + public PoC + in-stack
   = highest priority

## Pitfalls
- NVD enrichment now lags; do not assume a missing CVSS
  means low severity
- Vendor advisories often disagree with NVD scoring
- 'Affected version' ranges frequently exclude backports

## Verification
Every output carries: CVSS, EPSS, KEV status, PoC status,
stack relevance, and a one-line urgency rationale.
```

Note the Pitfalls section. That is where the agent records what it got wrong last time — and it is the section that most repays a human read, because it shows you what the agent believes about your environment.

---

## Part 5 — Live Demos: Watch Skills Write Themselves

Each demo below is a multi-step security task that comfortably clears the five-tool-call threshold and uses only public information. Run one, inspect what the agent crystallised, then run the paired follow-up to watch it reuse the skill.

### Demo 1: Threat Actor Profile

> **💬 Prompt**
> Build me a threat profile on a ransomware group.
> Pick one that has been active in the last 90 days. For it:
> 1. Search for recent activity and confirmed victims
> 2. Identify the sectors and regions it targets
> 3. Find its documented initial access methods
> 4. Map its known TTPs to MITRE ATT&CK technique IDs
> 5. Collect any publicly published IOCs
> 6. Note whether any decryptor or law-enforcement action exists
>
> Format it as: Summary, Targeting, Initial Access, ATT&CK Mapping, IOCs, Current Status. Cite every source.

**Follow-up:** "Same profile for [different group]." The agent loads the skill it just wrote, adapts the variables, and finishes noticeably faster. That second run is the whole lesson.

### Demo 2: CVE Assessment Against a Declared Stack

> **💬 Prompt**
> Assess a CVE for me and tell me how urgently we need to act.
> CVE: [pick a recent high-severity CVE]
> Our stack: AWS, Nginx, PostgreSQL 15, Kubernetes, Redis, Node.js, Cloudflare.
>
> Work through: NVD details and CVSS vector, EPSS score, CISA KEV status, whether public PoC code exists, whether it touches anything in our stack, and whether the affected component is internet-facing.
>
> Finish with an urgency call and a one-line rationale I could defend in a change review.

**Follow-up:** "Same assessment for [different CVE]." Watch it skip the exploration and go straight to the sequence it learned.

### Demo 3: Phishing Email Analysis

> **💬 Prompt**
> Analyse this reported phishing email and write it up.
> Sender: [paste sender address]
> Subject: [paste subject]
> Body summary: [describe or paste, redacting anything real]
> Links: [paste domains only, do not visit them]
>
> Steps: check sender domain registration and reputation via public sources, analyse the link domains for typosquatting, identify the social-engineering pressure tactics used, map the technique to MITRE ATT&CK, extract every IOC, and classify as Confirmed Phishing / Suspicious / Legitimate.
>
> Output an analyst write-up with a clear verdict and the IOC list separated out.

This demo is particularly instructive: the pitfalls the agent records will be specific and visible — things like "do not fetch the suspicious URL directly." You get to see the agent writing its own safety rules into procedural memory.

### Demo 4: Security Tool Comparison

> **💬 Prompt**
> I need a comparison to take into a vendor selection meeting.
> Compare three open-source SIEM options on: deployment model, data ingestion limits, detection rule ecosystem, community activity in the last 12 months, known CVEs in the product itself, and licensing constraints.
>
> Use public sources only. Cite everything. Finish with a recommendation matrix scored against: small team, limited budget, cloud-native environment.

### Demo 5: Compliance Control Mapping

> **💬 Prompt**
> Map a control across frameworks for me.
> Control: multi-factor authentication for privileged access.
> Find the corresponding control ID and requirement wording in: ISO 27001:2022, NIST CSF 2.0, CIS Controls v8, PCI DSS 4.0, and SOC 2 Trust Services Criteria.
>
> Produce a mapping table, note where the frameworks disagree on scope or strictness, and flag which one sets the highest bar. Cite the source for each mapping.

**Follow-up:** "Now do the same for encryption of data at rest." The crystallised skill should carry the framework list, the table format, and the disagreement-flagging step.

### Demo 6: Post-Incident Public Case Study

> **💬 Prompt**
> Build me a case study on a publicly disclosed breach.
> Pick a significant breach disclosed in the last 6 months. For it: reconstruct the timeline from public reporting, identify the initial access vector, map the attack chain to MITRE ATT&CK, identify which controls would have broken the chain and where, and extract three transferable lessons.
>
> Cite every claim. Where reporting conflicts, say so rather than picking a version.

### Inspecting What the Agent Wrote

```bash
# Isolate ONLY what was just created --- cleanest view
find ~/.hermes/skills/ -name 'SKILL.md' -mmin -30

# Or anything from the last day
find ~/.hermes/skills/ -name 'SKILL.md' -mtime -1

# Read it
cat ~/.hermes/skills/<category>/<skill-name>/SKILL.md
```

The `-mmin -30` filter matters: a plain `ls -R` buries the new skill among the ~70 bundled ones; filtering by modification time shows you exactly what the agent just authored.

> **This is not template reuse**
> The agent adjusts variables inside the skill based on new context. And it improves existing skills during use — if it finds a better approach on the second run, it patches the skill with a targeted diff rather than rewriting the document. The skill you read after run one is not necessarily the skill you will read after run five.

---

## Part 6 — The Curator: Managing an Accumulating Library

Skills accumulate. The curator is a background maintenance pass for agent-created skills: it tracks how often each skill is viewed, used, and patched, moves long-unused skills through active → stale → archived states, and can spawn an auxiliary-model review that proposes consolidations.

It exists because unmanaged skill libraries rot. After months of use a profile accumulates skills for workflows that no longer exist, and the agent has no built-in sense of when its own knowledge has expired.

### 6.1 The Scope Rules — Read These First

| Skill type | What the curator may do to it |
|---|---|
| Agent-created (`created_by: agent`) | Fully managed — state transitions, archiving, consolidation, patching |
| Bundled built-in | Only archived after `archive_after_days` of non-use, and only when `prune_builtins: true` (the default). Never patched, consolidated, or deleted. |
| Hub-installed | Always off-limits. Never touched. |
| Pinned (any type) | Exempt from every auto-transition and from the LLM review pass entirely |

> **The curator never deletes**
> The maximum destructive action is archive. Archived skills move to `~/.hermes/skills/.archive/` and are excluded from the prompt, but they are recoverable with `hermes curator restore <skill>`. Auto-deletion never happens. Telemetry lives in a sidecar at `~/.hermes/skills/.usage.json`, tracking per-skill `use_count`, `view_count`, `patch_count`, `last_activity_at`, `state`, and pinned status.

### 6.2 Everyday Commands

**Inspect before you act**

```bash
hermes curator status           # overview: per-skill usage, states, last run
hermes curator list-archived     # what is currently in .archive/
hermes curator run --dry-run     # preview a full pass --- report, no mutations
```

**Running a pass**

```bash
hermes curator run                # prune-only pass (free --- no model tokens)
hermes curator run --background   # fire-and-forget in a background thread
hermes curator run --consolidate  # force the LLM consolidation pass this run
```

**Managing individual skills**

```bash
hermes curator pin cve-triage     # never auto-transition this one
hermes curator unpin cve-triage
hermes curator archive old-skill  # manually archive (refuses pinned skills)
hermes curator restore old-skill  # bring an archived skill back to active
```

**Bulk cleanup**

```bash
hermes curator prune --dry-run          # preview: what would be archived
hermes curator prune --days 120         # bulk-archive unpinned skills idle 120+ days
hermes curator prune --days 90 --yes    # skip the confirmation prompt

# Default is 90 days. Falls back to created_at when the skill
# has never been used, so never-used skills can still be pruned.
```

**Safety net: backup and rollback**

```bash
hermes curator backup             # manual snapshot of ~/.hermes/skills/
hermes curator rollback --list    # list available snapshots
hermes curator rollback           # restore the newest snapshot
hermes curator rollback --id <timestamp>
hermes curator rollback -y        # skip confirmation
```

**Pausing entirely / slash equivalents**

```bash
hermes curator pause              # stop scheduled runs until resumed
hermes curator resume

# Slash commands mirror the CLI inside a session:
/curator status
/curator run --dry-run
```

### 6.3 Configuration

All curator settings live in `config.yaml` — not `.env`, since none of this is secret.

```yaml
# ~/.hermes/config.yaml
curator:
  enabled: true
  interval_hours: 168      # run weekly
  min_idle_hours: 2        # wait for a quiet period before running
  stale_after_days: 30     # active → stale
  archive_after_days: 90   # stale → archived
  prune_builtins: true     # may also archive unused bundled skills
  consolidate: false       # LLM consolidation pass OFF by default
```

**On cost:** the deterministic inactivity and prune sweep runs for free — zero model tokens. The "consolidate overlapping skills into umbrellas" pass uses the auxiliary model and is off by default because it costs tokens on every run and makes broad structural changes to your library. Opt in deliberately, or run it on demand with `--consolidate`.

### 6.4 A Practical Routine for Security Teams

1. **Pin your critical security skills immediately.** `hermes curator pin cve-triage`, `hermes curator pin threat-profile`. Pinned skills are exempt from auto-transitions AND from `skill_manage delete` — the agent cannot remove them. Patching still works, so they keep improving.
2. **Take a backup before any bulk operation.** `hermes curator backup` costs nothing and gives you a rollback point.
3. **Always dry-run first.** `hermes curator prune --dry-run` and `hermes curator run --dry-run` show exactly what would change before anything moves.
4. **Review status monthly.** `hermes curator status` shows which skills the agent actually uses. A skill with a high `patch_count` is one the agent keeps revising — worth reading to see what it is learning.
5. **Leave consolidate off unless you review the result.** Merging skills into umbrellas is a broad structural change to procedural memory. Run it manually with `--consolidate` when you have time to read the diff, not on a schedule.

> **🔒 Security lens**
> Pinning is a security control, not just housekeeping. `skill_manage(action="delete")` refuses pinned skills outright, so a pinned skill cannot be removed by the agent — whether through drift, a bad reflection pass, or manipulation. Pin the skills that encode your judgement, and the agent can improve them but never destroy them.

### 6.5 Upstream Drift Detection — A Supply Chain Control

Separately from the curator, Hermes tracks provenance for skills installed from the Hub so it can detect when the upstream copy changes. It compares the stored source identifier against the current upstream bundle content hash.

```bash
hermes skills check          # which installed hub skills changed upstream
hermes skills update         # reinstall only those with updates
hermes skills update <name>  # update one specific skill
```

> **🔒 Security lens**
> Treat `hermes skills check` as a supply chain integrity control and run it on a schedule. A skill you reviewed and approved three months ago can be silently modified upstream; this is how you find out. It is the agent-ecosystem equivalent of monitoring your dependencies for unexpected changes.

---

## Part 7 — The Security Lens: Governing the Loop

Everything above describes a genuinely elegant system. Now the part that matters most: an agent that writes its own instructions is an agent that can be taught the wrong thing — and unlike a prompt injection that fades when the session ends, a poisoned skill persists on disk and loads into every future session.

> "A skill file encoding a successful attack pattern is as dangerous as the attack itself. An agent learning from poisoned inputs will encode that learning and apply it later."

### 7.1 The Distinction That Defines the Threat Model

> **The agent cannot install skills. It can only write them.**
>
> This is the most important security fact in this module, and it is widely missed. The Skills Hub is exclusively user-operated. Agents CANNOT autonomously install, modify, or delete Hub skills. That is a deliberate design decision to stop an agent expanding its own capabilities without human oversight. What agents CAN do is create procedural memory skills through the `skill_manage` tool after completing complex workflows. So the threat model splits cleanly in two, and each half needs a different control:
>
> - **Marketplace risk** (a malicious third-party skill) — mitigated by the scanner and the trust model, and gated by a human running the install command.
> - **Self-authored risk** (the agent writes a poisoned skill from a manipulated interaction) — NOT covered by the install scanner at all, because nothing is being installed. This is why the write-approval gate in 7.4 is not optional for security work. It is the only control that addresses the second half.

### 7.2 The Install-Time Scanner

All hub-installed skills go through a security scanner before landing on disk. The install flow is: fetch → quarantine → scan → policy decision → install or block-and-audit. If the scan blocks an install, the quarantined copy is deleted and the event is written to an audit log.

**What the scanner looks for:** data exfiltration, prompt injection, destructive commands, supply-chain signals, secret exfiltration patterns (curl interpolating `$API_KEY`, `$TOKEN`, `$SECRET`), reads of credential stores (`~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.kube`, and Hermes's own `~/.hermes/.env`), persistence mechanisms, and obfuscation.

**Verdicts and the `--force` flag**

| Verdict | Behaviour |
|---|---|
| safe | Installs normally |
| caution / warn | Blocked by policy for community sources. Can be overridden with `--force` after you have read the report. |
| dangerous | HARD BLOCKED. `--force` does not override a dangerous verdict. There is no flag that bypasses this gate. |

```bash
# Read the SKILL.md, source, scripts and scan results FIRST
hermes skills inspect skills-sh/<author>/<skill>

# Install with default policy (blocks dangerous, prompts on warn)
hermes skills install skills-sh/<author>/<skill>

# Override a NON-dangerous finding, only after manual review
hermes skills install skills-sh/<author>/<skill> --force
```

> **🔒 Security lens**
> Reflexively passing `--force` defeats the entire gate. The discipline: inspect, read the SKILL.md body yourself, and only use `--force` when you understand the specific finding and accept it. Red flags in a skill body: unexpected destructive commands, instructions to upload files or environment variables to external endpoints, vague instructions that pull arbitrary URLs at runtime, and hidden meta-instructions in the description attempting to override the system prompt.

### 7.3 One More Collision Risk: Bundles

Hermes supports skill bundles — named groups of skills invoked together with a single slash command. Bundles take precedence over individual skills when slugs collide: if you have a bundle named `research` and also a skill named `research`, `/research` invokes the bundle. The documentation notes this is intentional, since you opted in by naming it.

```bash
hermes bundles create backend-dev \
  --skill github-code-review \
  --skill test-driven-development \
  --skill github-pr-workflow \
  -d "Backend feature work"

hermes bundles list
hermes bundles show backend-dev
hermes bundles delete backend-dev

# In a session: /bundles lists them, /<bundle-name> loads one
```

> **🔒 Security lens**
> Intentional or not, name collision plus precedence is a shadowing pattern worth knowing. If you standardise bundle names across a team, make sure nobody can quietly introduce a skill that a bundle silently overrides — or vice versa.

### 7.4 The Write-Approval Gate — The Control Most People Miss

Hermes ships an official mechanism to put a human in the learning loop. It is off by default, and almost nobody turns it on. Remember from 7.1: this is the ONLY control that addresses agent-authored skills, because nothing is being installed and the install scanner never runs.

```yaml
# ~/.hermes/config.yaml --- gate both stores
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200   # ~800 tokens
  user_char_limit: 1375     # ~500 tokens
  write_approval: true      # ← REQUIRE APPROVAL

skills:
  write_approval: true      # ← REQUIRE APPROVAL
  guard_agent_created: true # ← content scanner (see 7.5)
```

With `write_approval: true`, every `skill_manage` write — create, edit, patch, delete, write_file, remove_file — is staged instead of committed. Staged writes survive restarts under `~/.hermes/pending/skills/`. Critically, this applies whether the write came from a foreground turn or from the background review that runs after a session.

**Reviewing staged writes**

```bash
/skills pending          # list staged skill writes + one-line gist
/skills diff <id>        # full unified diff
/skills approve <id>     # apply it (or 'all')
/skills reject <id>      # drop it (or 'all')
/skills approval on      # turn the gate on (or 'off') and persist

/memory pending          # staged memory writes (auto ones tagged [auto])
/memory approve <id>     # apply one (or 'all')
/memory reject <id>      # drop one (or 'all')
/memory approval on      # turn the gate on (or 'off') and persist
```

The official documentation frames this as Hermes's consent-aware learning loop, and describes the memory gate as the answer to "the agent saved a wrong assumption about me." It explicitly recommends turning it on for secure environments, for small models that misjudge what they learned, or for anyone who simply wants eyes on the self-improvement loop.

> **🔒 Security lens**
> For any security-sensitive Hermes deployment, set both `memory.write_approval` and `skills.write_approval` to `true`. Yes, it slows the learning loop down. That is the trade: you exchange some autonomous improvement for the ability to see — and refuse — everything your agent decides to learn. Given that a poisoned skill persists indefinitely and executes automatically, that is a trade worth making.

### 7.5 The Guard Is Not the Gate

There is a second, independent setting that is easy to confuse with the first. `skills.guard_agent_created` is a content scanner — dangerous-pattern heuristics applied to agent-created skill writes. The official documentation is explicit that it is not an approval gate, and that the two are independent.

Run both. The guard catches known-bad patterns automatically. The gate lets you see everything, including the novel patterns the guard does not recognise. And remember the framing from the Hermes security policy: in-process scanners are heuristics operating on attacker-influenced strings. They are useful. They are not boundaries.

| Control | What it covers | What it misses |
|---|---|---|
| Install scanner | Third-party skills from the Hub | Anything the agent writes itself |
| `guard_agent_created` | Known-bad patterns in agent-written skills | Novel patterns it has no signature for |
| `write_approval` | Every agent write, foreground or background | Nothing — but it depends on you actually reading the diff |
| Pinning | Protects a skill from agent deletion | Does not stop patching — by design |

---

## Part 8 — Practical Controls for the Learning Loop

1. **Pin the skills that encode your judgement.** `hermes curator pin <skill>` makes them undeletable by the agent while still allowing improvement.
2. **Never install a third-party skill without `hermes skills inspect` first.** Read the SKILL.md body yourself. Do not reflexively reach for `--force`.
3. **Prefer trusted-tier sources** (`openai/skills`, `anthropics/skills`, `huggingface/skills`, `NVIDIA/skills`) over arbitrary community repos. If your team publishes vetted skills, add your own tap and install from there preferentially.
4. **Run `hermes skills check` on a schedule** to catch upstream drift in skills you already reviewed and approved.
5. **Put `~/.hermes/skills/` under Git** so every agent-authored change is diffable and reversible.
6. **Watch the directory with `inotifywait` or `fswatch`** so new skills announce themselves.
7. **Run `hermes curator` on a schedule** to consolidate and archive — an unaudited library grows past what anyone can review.
8. **Read `MEMORY.md` and `USER.md` periodically.** They are small by design (2,200 and 1,375 characters). There is no excuse for not reading them.
9. **Use profile isolation** to separate a research agent that ingests untrusted web content from an agent that touches anything you care about.

## Quick Command Reference

| Command | Purpose |
|---|---|
| `hermes skills browse / search` | Find skills in the Hub |
| `hermes skills inspect ID` | Preview a skill WITHOUT installing — always do this first |
| `hermes skills install ID` | Install (`--force` overrides caution/warn, never dangerous) |
| `hermes skills check / update` | Detect and apply upstream changes — supply chain control |
| `hermes skills tap add REPO` | Register your own vetted repo as a source |
| `hermes skills publish PATH` | Publish a skill to the registry |
| `hermes skills reset <name> --restore` | Restore a deleted bundled skill |
| `hermes bundles create / list / show` | Group skills under one slash command |
| `hermes curator status` | Per-skill usage, state, last run |
| `hermes curator run --dry-run` | Preview a maintenance pass, no changes |
| `hermes curator pin <skill>` | Protect a skill from auto-transition AND agent deletion |
| `hermes curator prune --days N` | Bulk-archive skills idle N+ days (default 90) |
| `hermes curator backup / rollback` | Snapshot and restore the whole skills directory |
| `hermes curator list-archived` | See what has been archived |
| `/skills pending \| diff \| approve` | Review staged agent-authored skill writes |
| `/memory pending \| approve \| reject` | Review staged memory writes |

## Key Takeaways

1. A skill is a Markdown runbook that executes — procedural memory, not a plugin or a tool. Installing one is closer to importing a system prompt than running `pip install`.
2. All active skills live in `~/.hermes/skills/<category>/<name>/SKILL.md`, with optional `scripts/`, `references/`, `templates/`, `examples/` and `assets/` subdirectories.
3. Bundled skills contain NO security category. A cybersecurity library comes from the official/trusted tiers or from the learning loop — you build it.
4. The Skills Hub is a package manager for agent procedures: browse, search, inspect, install, check, update, publish. Taps let you register your own vetted repo — but new taps get community trust by default.
5. Four trust levels govern skills: builtin, official, trusted, community — each with a different scanning and install policy.
6. The loop is Observe → Execute → Reflect → Crystallise → Reuse, triggered by four documented conditions including complex tasks of 5+ tool calls.
7. **The key distinction:** agents cannot install Hub skills — that is human-only by design. But agents CAN author their own skills via `skill_manage`, and the install scanner never sees those.
8. The install scanner covers marketplace risk: fetch → quarantine → scan → policy → install or block-and-audit. `--force` overrides caution and warn. It never overrides dangerous.
9. Self-authored risk is covered only by `write_approval`, `guard_agent_created`, and pinning — the first two are off by default.
10. The curator never deletes; max action is archive, and archives are recoverable. Pinning is a security control: pinned skills cannot be deleted by `skill_manage` but can still be patched.
