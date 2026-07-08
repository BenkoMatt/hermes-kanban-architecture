# Hermes Kanban Architecture

A multi-profile, Kanban-orchestrated agent architecture built on [Hermes Agent](https://hermes-agent.nousresearch.com) by Nous Research. One orchestrator profile delegates tasks to specialized worker profiles via an automated Kanban board — tasks are routed by matching keywords against profile descriptions.

## How It Works

```
Father Matt chats in `daily`
        │
        ▼
   Caddy (orchestrator)
   creates Kanban task
        │
        ▼
   Auto-Decomposer
   reads task + matches
   against profile descriptions
        │
        ▼
   Task assigned to
   best-matching worker profile
        │
        ▼
   Worker executes autonomously
   using its specialized skills
        │
        ▼
   Result returns to board
   → reported back to Matt in `daily`
```

The dispatcher runs every 60 seconds in the gateway. No manual profile switching needed — just talk to the orchestrator and it routes to the right specialist.

## Fleet Overview

### Orchestrator

| Profile | Role | Description |
|---------|------|-------------|
| **daily** | Orchestrator / PM | Coordinates Kanban dispatch, manages all other profiles, handles system health, cron jobs, and cross-profile operations. Has the full skill catalog. |

### Workers (Kanban-routable)

| Profile | Specialty | Routes What |
|---------|-----------|-------------|
| **creative** | Creative content generation | ASCII art, diagrams, image generation, p5.js, pixel art, manim animations, web design prototyping |
| **personal** | Personal assistant | Email, smart home, media, gaming servers, social media, day-to-day personal tasks |
| **platform** | Infrastructure / DevOps | VPS operations, Docker, s6-overlay supervision, Hermes dashboard, environment setup, webhooks |
| **security** | Cybersecurity | Security audits, VPS hardening, threat assessment, GitHub token management, posture monitoring |
| **webdev** | Web development / QA | FastAPI web apps, static sites, GitHub Pages DNS, dogfood QA testing, code quality gates |
| **research** | Research / analysis | Web research, competitive intelligence, market analysis, literature reviews, Firecrawl data gathering |
| **jlp** | Jenna Lynn Photography | Photography website (prod + staging), gallery updates, Instagram, business operations |
| **finance** | Budget / financial | Budget statements, expense tracking, financial analysis, BudgetSage operations |
| **mmd** | Matt's Mystery Dates | Date night generator — plans surprise three-part dates (event + dinner + dessert) using web research and learned preferences |

### Isolated (NOT routable)

| Profile | Role | Why Isolated |
|---------|------|-------------|
| **work** | Sensitive professional work data | No Honcho memory, no Kanban, no delegation, no shared skills. Standalone island — no bleed between work and personal. |

## Shared Knowledge Layer

All connected profiles (10 of 11) access shared deep-reference skills via `external_dirs`:

| Skill | Version | Reference Files | Covers |
|-------|---------|-----------------|--------|
| `hermes-agent` | v2.2.0 | 20 | Hermes Agent internals — context files, memory providers, hooks, security model, provider resilience, code execution, persistent goals |
| `honcho-deep-reference` | v1.0.0 | 12 | Honcho v3 memory system — architecture, reasoning, peer representations, design patterns, dreaming, CLI/SDK, Hermes integration |
| `firecrawl-mastery` | v1.0.0 | 10 | Firecrawl v2 — scrape/search/crawl/map, interact, agent extraction, document parsing, PII redaction, monitoring, research index, CLI, MCP, errors |

These skills load on-demand only when a profile's task requires them — no context bloat.

## Key Design Principles

1. **One orchestrator, many specialists** — Matt talks to `daily`; the decomposer routes to the right worker automatically.
2. **Lean profiles** — each profile keeps only skills relevant to its specialty. 259+ unused skills were trimmed across worker profiles.
3. **Isolation for sensitive data** — the `work` profile is a standalone island with no cross-profile connections.
4. **Shared knowledge, not shared context** — deep-reference skills are available to all connected profiles but only load when needed.
5. **Descriptions drive routing** — the Kanban auto-decomposer matches task keywords against profile descriptions to pick the right worker.
6. **Scalable** — the system runs on a single Contabo VPS with 7.8GB RAM and comfortably scales to 15+ profiles.

## Infrastructure

- **Platform:** Hermes Agent by Nous Research
- **VPS:** Contabo (Ubuntu 24.04, 7.8GB RAM, 145GB disk)
- **Model:** GLM-5.2 via Ollama Cloud
- **VPN:** Tailscale
- **Memory:** Honcho v3 (hybrid mode) for connected profiles; built-in memory for isolated `work` profile
- **Dispatcher:** Runs in gateway, 60-second poll interval

## Setup Overview

Each profile is created with:
```
hermes profile create <name> --description "..."
```

Then configured with:
1. Model assignment (config.yaml)
2. Shared skills access (`external_dirs` pointing to shared-skills directory)
3. Skill trimming (remove unused bundled skills)
4. Toolset configuration (Kanban, delegation, etc.)

The isolated `work` profile additionally removes:
- Kanban toolset
- Delegation toolset
- Session search toolset
- `external_dirs` (shared skills)
- Honcho memory provider

## Repository

This repo documents the architecture for reference and sharing. The live system runs on the VPS described above.

---

Built by [Caddy](https://github.com/BenkoMatt) — an AI executive assistant running on Hermes Agent.