# Hermes Security Course Resource

A hands-on guide to using Hermes — a persistent, self-improving AI agent — safely and effectively for security work.

Hermes's defining feature is its closed learning loop: it captures how it solves problems and writes reusable skills, so it gets faster and more tailored to you the longer you use it. That same persistence and autonomy is also what makes it a real attack surface. This resource covers both sides — how to lock Hermes down, and how to get genuine leverage out of it once it's secured.

## Contents

### Modules

- [**Giving Hermes a Personality and Understanding Its Memory Files**](modules/hermes-personality-and-memory.md) — How to give Hermes a durable identity with `SOUL.md`, tell it who you are with `USER.md`, and understand the difference between `SOUL.md`, `MEMORY.md`, `USER.md`, and `AGENTS.md`, including their character limits and when changes take effect.
- [**Hermes Skills and the Self-Improvement Loop**](modules/hermes-skills-and-learning-loop.md) — What skills are, how the Skills Hub and trust tiers work, how the self-improvement loop (Observe → Execute → Reflect → Crystallise → Reuse) actually operates, and how to govern it with the Curator and write-approval gates.
- [**Hermes as a Security Force Multiplier**](modules/hermes-security-force-multiplier.md) — Day-one use cases (briefings, CVE triage, vendor monitoring), acting safely through Zapier MCP, and how the learning loop turns repetitive security work into a compounding personal analyst over 90 days.

### Labs

- [**Securing Hermes, Step by Step**](labs/securing-hermes-step-by-step.md) — A hands-on hardening lab covering VPS setup, Telegram gateway authorization, command execution controls, container isolation, credential and file protection, network controls, and ongoing monitoring.

## Suggested Order

1. Start with the [securing lab](labs/securing-hermes-step-by-step.md) to get a hardened Hermes instance running safely.
2. Read [Giving Hermes a Personality and Understanding Its Memory Files](modules/hermes-personality-and-memory.md) to give your hardened instance an identity and understand how it remembers things.
3. Read [Hermes Skills and the Self-Improvement Loop](modules/hermes-skills-and-learning-loop.md) to understand how the agent learns and how to govern that process.
4. Finish with [Hermes as a Security Force Multiplier](modules/hermes-security-force-multiplier.md) to put it to work on real security workflows.

## A Note on Scope

Every workflow in this resource is designed to run against public information and your own personal productivity tools. Do not connect Hermes to corporate systems, production infrastructure, identity providers, or sensitive data — treat it as a capable but untrusted assistant that lives outside your corporate perimeter.
