# 🏛️ El Templo Platform — Case Study

> A multi-app fitness platform for an Argentine calisthenics gym chain with 8 locations across Mar del Plata and Barcelona. Built by a 2-developer team using Claude Code and spec-driven AI workflows.

**Status:** Late-stage testing, pre-launch · **Role:** Contributing Full Stack Developer · **Team size:** 2 developers + CEO + data analyst

> ⚠️ This is a case study of a private client project. No source code is shared here to protect client IP. This document describes the system, architecture, technical decisions and my contributions.

---

## 📌 Quick Facts

| | |
|---|---|
| **Product type** | Multi-app SaaS platform (5 apps in monorepo) |
| **Client** | Private — multi-location calisthenics gym chain |
| **Scale** | 8 physical locations (7 Mar del Plata, 1 Barcelona) |
| **Team** | 2 developers (one senior, me as contributor) + CEO + data analyst |
| **Development approach** | AI-native: Claude Code + GSD (Get Shit Done) spec-driven framework |
| **Timeline** | v1.0 (Feb 2026) → v5.3 (active, Apr 2026) |
| **Total milestones shipped** | 5 major versions, 80+ phases |
| **Current state** | All apps in late-stage testing, mobile app awaiting Play Store review |

---

## 🎯 The Challenge

The client runs a calisthenics gym chain with a proprietary training methodology (SPOM) that defines how workouts are structured and progressed. Before this platform, coaches were building training programs manually, members had no unified way to track their progression, and customer service for new leads, trials and memberships ran entirely on human WhatsApp operators.

The goal: **build a full ecosystem that automates session generation, gives members a guided daily training experience, gives coaches tools to review and edit sessions at scale, and handles customer-facing WhatsApp conversations with AI — so the business can scale without linearly scaling headcount**.

---

## 🏗️ System Overview

The platform is a monorepo of 5 apps, each with a clear responsibility:

| App | Purpose | Stack | Deployment |
|---|---|---|---|
| **Member App** | PWA + native mobile app where members view their daily workouts, track sessions, and see progression | Vue 3, Quasar, Capacitor, TypeScript | `app.eltemplo.org` + Android/iOS |
| **Admin App** | Web app for coaches to review, approve and edit generated sessions, manage members, and handle business ops | Vue 3, Quasar, TypeScript | `admin.eltemplo.org` |
| **API** | Central backend serving both frontends and the bot; handles auth, session generation, business logic | Fastify 5, Drizzle ORM, MySQL 8, TypeScript | `api.eltemplo.org` |
| **Marketing Site** | Public-facing landing, franchise forms, blog | Nuxt 3 | Public domain |
| **WhatsApp AI Bot ("Mica")** | Conversational AI handling customer queries, trial bookings, class reservations, and human handoff | Fastify, WhatsApp Cloud API, OpenAI/Anthropic, Redis | Separate process on same EC2 |

---

## 🗺️ Architecture
```mermaid
flowchart TB
    subgraph Clients
        Member[Member AppVue + CapacitoriOS / Android / PWA]
        Admin[Admin AppVue + QuasarWeb]
        Web[Marketing SiteNuxt 3]
        WA[WhatsAppEnd User]
    end

    subgraph Backend[AWS EC2 + Nginx + PM2]
        API[Fastify APIapi.eltemplo.org]
        Bot[Mica BotFastify · Port 3001]
    end

    subgraph Data
        MySQL[(MySQL 8Permanent Records)]
        Redis[(RedisSessions · Locks · Context)]
        R2[(Cloudflare R2Exercise Videos)]
    end

    subgraph External[External Services]
        Meta[WhatsApp Cloud API]
        LLM[OpenAI / Anthropic]
        Sentry[Sentry]
    end

    Member --> API
    Admin --> API
    Web --> API
    WA  Meta
    Meta  Bot

    Bot -->|localhost HTTP| API
    Bot --> Redis
    Bot --> LLM

    API --> MySQL
    API --> Redis
    API --> R2
    API --> Sentry
    Bot --> Sentry
```

**Key architectural decisions:**

- **Modular monolith** over microservices — one Fastify API with explicit module boundaries (auth, sessions, admin, progression, SPOM). Easier to reason about, deploy, and debug at current scale.
- **Bot as separate process** — the WhatsApp bot runs as its own Fastify process alongside the API. If the bot crashes, the API stays up. If the API is unreachable, the bot degrades gracefully.
- **Redis for ephemeral state, MySQL for permanent records** — Redis handles conversation context (6h TTL), distributed locks for schedulers, and customer profiles (90d TTL). MySQL stores everything that must survive.
- **Model-agnostic AI abstraction** — an `AiProvider` interface lets us switch between OpenAI GPT-4o mini and Anthropic Claude Haiku via config, without touching business logic.
- **Cloudflare R2 with presigned URLs** for exercise videos — avoids routing large files through the API.

---

## 🧰 Tech Stack

**Frontend**
- Vue 3 (Composition API) · Quasar Framework · TypeScript
- Capacitor (member app: PWA + native iOS/Android)
- Pinia stores · Nuxt 3 (marketing site)

**Backend**
- Fastify 5 · Drizzle ORM · MySQL 8 · Redis
- TypeScript (strict, zero `any` policy)
- Pino structured logging

**AI / LLM**
- WhatsApp Cloud API (Meta, official)
- OpenAI GPT-4o mini / Anthropic Claude Haiku (model-agnostic)
- Function calling with custom tools (`check_schedule`, `book_class`, `register_trial`, `get_location`, `request_human`)
- RAG-style knowledge injection (12-section business knowledge base)

**Infrastructure & DevOps**
- AWS EC2 · Nginx reverse proxy · PM2 process manager
- Let's Encrypt SSL across 3 subdomains
- GitHub Actions CI/CD (build → test → rsync → migrate → restart → smoke test → auto-rollback)
- Sentry monitoring (API + frontend)
- Cloudflare R2 with presigned uploads

**AI-Assisted Development**
- Claude Code
- GSD (Get Shit Done) spec-driven framework
- Research → Plan → Execute → Verify cycle per phase

---

## 🔍 Feature Deep Dives

### 1. Algorithmic Session Generation (SPOM Engine)

The client has a proprietary training methodology called SPOM with **1,040 rules, 936 weekly rotator entries, and 1,870 exercises**. Coaches used to build programs manually per member — slow, inconsistent, not scalable.

**What we built:** A deterministic 9-stage generation pipeline that produces a full week of personalized sessions given a member's branch, level, and training history. The pipeline handles format compatibility, contraction distribution, intensity-based budgets, and per-block mobility exercises.

**Technical highlights:**
- Linear difficulty scale (1-12) validated against 19 coach-built reference weeks
- Per-block exercise count caps with format-specific parameters (EMOM intervals, AMRAP time caps, Complex rounds)
- Saved blocks for coach reuse
- Admin review workflow with bulk approve, coverage alerts, and a full audit trail of edits

---

### 2. WhatsApp AI Bot ("Mica")

The client wanted to automate customer-facing WhatsApp conversations — answering questions about pricing, schedules and locations, booking trial classes, registering new leads, and handing off to humans when needed.

**What we built:** A separate Fastify process connected to the WhatsApp Cloud API that uses function-calling LLMs to handle conversations. Mica has an Argentine Spanish persona and a 12-section business knowledge base covering pricing, schedules, FAQs, objection handling and sales playbooks.

**Technical highlights:**
- **Webhook pipeline** verifying Meta signature, deduplicating messages, and pushing to the AI layer
- **Model-agnostic AI provider** — swap OpenAI ↔ Anthropic via env var
- **Function calling tools:** `check_schedule`, `check_membership`, `get_location`, `book_class`, `register_trial`, `request_human` — all calling the API via localhost HTTP, with interactive WhatsApp button confirmation for destructive actions
- **Two-layer memory:** Redis session context (last 20 messages, 6h TTL) + customer profile (90d TTL, injury notes, preferences)
- **Client lifecycle state machine:** `LEAD → TRIAL → ACTIVE_MEMBER → INACTIVE_MEMBER → EXPIRED_MEMBER`, with state-adaptive sales objectives per stage
- **Proactive schedulers** using `node-cron` with Redis distributed locks for deduplication on process restart: class reminders N hours before booked sessions, trial follow-ups 24-48h after attendance
- **Admin conversation UI** with paginated chat list, unread badges, status filters, chat-bubble detail view and a **"Tomar control"** human takeover that pauses the bot and lets a human admin reply, then resumes

The bot shipped in 9 days across 7 phases — **~16,000 lines added, 23 requirements completed**.

---

### 3. Admin Session Management & Editing

Coaches needed a way to review and edit AI-generated sessions before approving them for members.

**What we built:** A full editing workflow in the admin app with session review queue, bulk approve, per-exercise swap dialogs, format and prescription changes, block reordering, and a complete edit audit trail.

**Technical highlights:**
- Full CRUD on sessions, blocks, exercises with optimistic UI
- pdfmake-based PDF export with format-specific parameter display
- Edit history stored in `session_edit_logs` for coach accountability
- Major refactor of the main editing service (1,232 → 350 LOC) decomposing a god-object into domain services
- Similar refactor on the member-side Day Player (900 → 350 LOC)

---

## 🤖 Development Approach

This platform was built with a **fundamentally different workflow** from traditional team software development.

**Team composition:** 2 developers (one senior, me as contributor), a CEO, and a data analyst acting as sales lead. The dev team drives delivery; business members define requirements and validate outcomes.

**Toolchain:** We work inside **Claude Code** using the **GSD (Get Shit Done) framework** — a spec-driven system that structures every phase into:

1. **Research** — understand the problem, read the existing code, identify affected modules
2. **Plan** — produce multiple candidate plans with tradeoffs before any code is written
3. **Execute** — implement with autonomous agents, generating atomic, well-documented commits
4. **Verify** — automated test runs + manual review before merging

**What this looks like in practice:**

- Each milestone (v1.0, v2.0, v5.0…) is broken into **phases** (e.g. Phase 82: Playbook Engine), and each phase into **plans** with explicit **requirement IDs** traceable to a `REQUIREMENTS.md` file.
- A single phase often produces **100+ atomic commits** tied to specific requirements.
- **Requirements, milestones, roadmap and architecture decisions are all versioned in markdown** inside the repo (`.planning/`), treated as first-class artifacts.
- Pre-commit hooks enforce formatting, linting and zero `any` types.
- CI/CD runs full type check, lint, security audit, integration tests and build on every push. Deploy pipeline includes automated backup/rollback on failure.

This approach lets a 2-developer team ship at a velocity that would traditionally require 4-6 people. The tradeoff is that the human developers are no longer typing most of the code — we are **directing, reviewing, validating and making architectural decisions**.

---

## 👤 My Contributions

Honest breakdown of my role:

- **I am the junior developer on a 2-dev team.** The senior dev leads architecture and drives the hardest technical decisions; I contribute across the stack and own specific pieces.
- **I direct Claude Code / GSD on features assigned to me**, review the generated plans, validate the output, debug issues, and integrate with the rest of the codebase.
- **I understand the system at a functional and architectural level** — how modules interact, why decisions were made, how data flows — and I'm continuously learning the deeper internals of parts I haven't touched directly.
- **Areas where I've contributed directly:** frontend pages and components in the member and admin apps, integration with the API, bug fixes and QA across the stack, participation in phase planning and review.

My role is best described as **AI-assisted contributing developer**: I multiply my output through AI tooling, but I own what I ship and understand what I'm shipping.

---

## 📅 Milestones Shipped

| Version | Date | What Shipped |
|---|---|---|
| **v1.0** Training Module | Feb 2026 | Auth, SPOM engine import (1040 rules), deterministic session generation, member weekly view + day player, progression system, brand identity |
| **v2.0** Admin App | Feb 2026 | Full admin app, session review & editing, PDF export, mobility exercises, SSL deployment across 3 subdomains, Sentry, Vitest integration tests, major refactors, per-member journeys, video integration, R2 upload, staging environment |
| **v3.0** Landing & Web | — | Nuxt 3 marketing site, franchise forms, blog, brand alignment |
| **v4.0 / v4.1** Ecosystem Foundation | — | Admin consolidation, attendance & scheduling, data migration |
| **v5.0** WhatsApp AI Bot | Mar 2026 | WhatsApp Cloud API integration, AI with function calling, Redis memory, action tools, proactive schedulers, admin conversations UI, human takeover |
| **v5.1** Production Readiness | Mar 2026 | Business knowledge base, production seeding, CI/CD pipeline for the bot, WhatsApp production setup |
| **v5.2** Mica Persona | Apr 2026 | Argentine Spanish persona, 12-section knowledge base, WhatsApp-only formatting, full conversation flow testing |
| **v5.3** Conversational Sales Engine | 🚧 Active | Playbook engine, lead discovery mode, state-adaptive prompts, avatar adaptation |

---

## 🧠 Technical Learnings

Things I've internalized working on this platform:

- **Function calling is fundamentally different from prompt-only LLM usage.** Designing tools (`book_class`, `register_trial`) as clean contracts the model can reason about is closer to API design than prompt engineering.
- **State machines belong in conversational AI.** Without an explicit client lifecycle state, the bot becomes inconsistent. With it, every response has a clear objective.
- **Redis distributed locks matter more than I expected.** A scheduler that fires twice because PM2 restarted is a real bug with real consequences (sending a customer the same WhatsApp twice). Dedup keys are not optional.
- **Modular monoliths beat microservices at small team size.** Moving between modules in a single codebase is faster than coordinating contracts across services. We'll split when we need to, not before.
- **Spec-driven AI development requires writing better specs, not less.** The `REQUIREMENTS.md` file is now the artifact I care about most — if the requirements are wrong, the code will be wrong, faster.
- **Zero `any` TypeScript is worth the friction.** Refactors that would have scared us are safe because the type system has our back.

---

## 📍 Current Status

- **v1.0 – v5.2:** Shipped to production infrastructure, in late-stage testing
- **v5.3:** In active development (Playbook Engine + Discovery Mode)
- **Mobile app:** Awaiting Play Store review · iOS TestFlight workflow ready · early test users on Android
- **Public launch:** Imminent pending Play Store approval and final QA pass

---

## 📫 Contact

**Matías Bordón** — AI-Native Full Stack Developer  
📍 Mar del Plata, Argentina · Remote, GMT-3  
🔗 [LinkedIn](https://linkedin.com/in/matias-bordon-160555273) · 💻 [GitHub](https://github.com/MatiasBordon1) · ✉️ matzaia2001@gmail.com

*Open to full stack and AI engineering roles at remote-first companies.*
