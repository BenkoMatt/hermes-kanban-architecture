# Hermes Kanban Architecture

A multi-profile, Kanban-orchestrated agent architecture built on [Hermes Agent](https://hermes-agent.nousresearch.com) by Nous Research. One orchestrator profile delegates tasks to specialized worker profiles via an automated Kanban board — tasks are routed by matching keywords against profile descriptions.

## How a Task Flows Through the System

```
┌─────────────────────────────────────────────────────────────┐
│                        MATT (you)                     │
│              chats in `daily` profile                        │
│                                                             │
│  "Route this through Kanban: build a room designer app"     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                    DAILY (Orchestrator/PM)                    │
│                                                             │
│  Caddy receives your request, creates a Kanban task card     │
│  with: title, body (full spec), assignee, goal mode          │
│                                                             │
│  hermes kanban create "Build room designer app" \            │
│    --body "..." --assignee webdev --goal                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   KANBAN BOARD (SQLite)                       │
│                                                             │
│  Task ID: t_83a21dda   Status: ready                        │
│  Assignee: webdev      Created: 14:28                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   DISPATCHER (Gateway, 60s poll)             │
│                                                             │
│  Every 60 seconds, the dispatcher checks for ready tasks.    │
│  It claims the task, spawns the worker profile in an         │
│  isolated workspace, and starts a timer.                     │
│                                                             │
│  → Claimed t_83a21dda for webdev                            │
│  → Spawned worker process (PID 686162)                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   WEBDEV (Worker Profile)                     │
│                                                             │
│  Spins up in isolated workspace:                             │
│  /root/.hermes/kanban/workspaces/t_83a21dda/                 │
│                                                             │
│  Reads task body → plans → executes autonomously:            │
│    • Analyzes reference app (1,624 lines)                   │
│    • Writes new app (79KB, 2,460 lines)                     │
│    • Deploys via HTTP server + Tailscale Serve               │
│    • Verifies zero JS errors in browser                     │
│                                                             │
│  Goal mode: a judge checks after each turn — keeps going    │
│  until the deliverable meets all requirements (up to 30     │
│  turns max).                                                │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   TASK COMPLETION                             │
│                                                             │
│  Worker calls kanban_complete with summary + artifacts       │
│  Task status: ready → running → done                        │
│  Duration: ~7 minutes                                       │
│  Artifacts: index.html (79KB)                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   BACK TO DAILY (Report)                      │
│                                                             │
│  Caddy reads the completed task, verifies the artifact,     │
│  and reports the result back to Matt.                 │
└──────────────────────────────────────────────────────────────┘
```

## The Fleet — Who Routes Where

```
                        ┌──────────────┐
                        │    daily     │
                        │ (Orchestrator│
                        │  /PM/Hub)    │
                        └──────┬───────┘
                               │
                    creates tasks & delegates
                               │
          ┌────────┬──────────┼──────────┬────────┬─────────┐
          │        │          │          │        │         │
          ▼        ▼          ▼          ▼        ▼         ▼
    ┌─────────┐┌────────┐┌─────────┐┌────────┐┌────────┐┌──────────┐
    │creative ││personal││platform ││security││webdev  ││ research │
    │         ││        ││         ││        ││        ││          │
    │ ASCII   ││ email  ││ VPS ops ││ audits ││ FastAPI││ web      │
    │ diagrams││ smart  ││ Docker  ││ harden ││ static ││ research │
    │ images  ││ home   ││ s6-ovly ││ tokens ││ GitHub ││ Firecrawl│
    │ p5.js   ││ gaming ││ dashboard││threats ││ QA/dog ││ market   │
    │ pixel   ││ social ││ webhooks ││monitor ││  food  ││ analysis │
    └─────────┘└────────┘└─────────┘└────────┘└────────┘└──────────┘

          ┌────────┐  ┌──────────┐  ┌─────────────────────┐
          │  jlp   │  │ finance  │  │        mmd          │
          │        │  │          │  │  Matt's Mystery     │
          │ JLP    │  │ budget   │  │  Dates              │
          │ website│  │ expenses │  │                     │
          │ gallery│  │ BudgetSg │  │ event+dinner+       │
          │ IG     │  │ analysis │  │ dessert planner     │
          └────────┘  └──────────┘  └─────────────────────┘

                                                    ┌─────────┐
                                                    │  work   │
                                                    │ 🔒 ISOL │
                                                    │ ATED    │
                                                    │         │
                                                    │ NO Kan  │
                                                    │ ban     │
                                                    │ NO Hon  │
                                                    │ cho     │
                                                    │ NO shar │
                                                    │ ed skills│
                                                    └─────────┘
```

## Kanban Task Lifecycle

```
    ┌──────────┐     dispatcher      ┌──────────┐     worker      ┌──────────┐
    │  ready   │ ──── claims ──────▶ │ running  │ ── completes ─▶ │   done   │
    │          │     & spawns        │          │                │          │
    └──────────┘                     └────┬─────┘                └──────────┘
                                          │
                                          │ if worker fails
                                          │ (max 2 retries)
                                          ▼
                                    ┌──────────┐
                                    │ blocked  │
                                    │ (review) │
                                    └──────────┘
```

## Profile Descriptions (Routing Rules)

### Orchestrator

| Profile | Description |
|---------|-------------|
| **daily** | Primary orchestrator and project manager. Coordinates Kanban dispatch, manages all other profiles, handles system health, cron jobs, and cross-profile operations. Has the full skill catalog. Route coordination, planning, and cross-cutting tasks here. |

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

## Trigger Words

| Trigger | Behavior |
|---------|----------|
| "Delegate this to [profile]" | Routes to a specific worker |
| "Route this through Kanban" | Decomposer picks the worker |
| "Queue a task for [profile]" | Assigns to a specific worker |
| "Send this to the board" | Decomposer picks the worker |
| *(no trigger words)* | Caddy handles directly in daily |

## Goal Mode

| Mode | Behavior |
|------|----------|
| **Normal** | Worker runs 1 turn → done |
| **Goal** (`--goal`) | Worker runs up to N turns, judge checks after each, keeps going until deliverable is verified complete (max 30 turns default) |

Use goal mode for complex builds and multi-step deliverables.

## Shared Knowledge Layer

All 10 connected profiles access shared deep-reference skills via `external_dirs`:

| Skill | Version | Reference Files | Covers |
|-------|---------|-----------------|--------|
| `hermes-agent` | v2.2.0 | 20 | Hermes Agent internals — context files, memory providers, hooks, security model, provider resilience, code execution, persistent goals |
| `honcho-deep-reference` | v1.0.0 | 12 | Honcho v3 memory system — architecture, reasoning, peer representations, design patterns, dreaming, CLI/SDK, Hermes integration |
| `firecrawl-mastery` | v1.0.0 | 10 | Firecrawl v2 — scrape/search/crawl/map, interact, agent extraction, document parsing, PII redaction, monitoring, research index, CLI, MCP, errors |
| `brave-search-mastery` | v1.0.0 | 7 | Brave Search API — LLM Context, Web Search, News Search, Answers, Goggles, search operators |
| `formspree-mastery` | v1.0.0 | 10 | Formspree — claim URLs (AI agent flow), HTML/AJAX/React forms, 25+ integrations, CLI, API, spam protection, form rules, webhooks |

Skills load on-demand only when a profile's task requires them — no context bloat. The isolated `work` profile has no access to shared skills.

## Key Design Principles

1. **One orchestrator, many specialists** — Matt talks to `daily`; the decomposer routes to the right worker automatically.
2. **Lean profiles** — each profile keeps only skills relevant to its specialty. 259+ unused skills trimmed across worker profiles.
3. **Isolation for sensitive data** — the `work` profile is a standalone island with no cross-profile connections.
4. **Shared knowledge, not shared context** — deep-reference skills are available to all connected profiles but only load when needed.
5. **Descriptions drive routing** — the Kanban auto-decomposer matches task keywords against profile descriptions to pick the right worker.
6. **Scalable** — the system runs on a single VPS with 7.8GB RAM and comfortably scales to 15+ profiles.

## Real Test Results (t_83a21dda)

```
  14:28:00  Task created          "Build customizable room designer web app"
  14:28:00  Dispatcher claimed    → webdev
  14:28:00  Worker spawned        PID 686162
  14:29:00  Analyzed reference    Read existing app (1,624 lines)
  14:30:00  Building              Writing new app with all 7 features
  14:34:00  Deployed              HTTP server + Tailscale Serve :8461
  14:35:00  Verified              Zero JS errors, all features functional
  14:35:00  Task marked done      7 minutes total, 74 messages, 34 tool calls

  Result: Room Designer Pro — 79KB single-file app with 12 furniture types,
          dual unit display, custom walls, height fields, professional dark UI
```

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

---

Built by [Caddy](https://github.com/BenkoMatt) — an AI executive assistant running on Hermes Agent.