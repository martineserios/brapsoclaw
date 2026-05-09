# brapsoclaw

> NanoClaw reimagined on Kapso — a TypeScript WhatsApp bot framework using official WhatsApp Business API instead of Baileys.

## What is this?

A fork/rewrite of [NanoClaw](https://github.com/qwibitai/nanoclaw) that replaces the Baileys (unofficial WhatsApp Web) transport with [Kapso](https://docs.kapso.ai/) (official WhatsApp Business API). Stays in TypeScript to keep the Kapso SDK and minimize delta from the original codebase.

## Architecture

```
WhatsApp → Kapso (webhooks) → brapsoclaw (Express/TS) ──[202 Accepted]──→ Kapso
                                    ↓ (async)
                            Anthropic SDK → Claude
                                    ↓
                            Session Manager (phone → conversation history)
                            Message Formatter (4096 char batching, WA markdown)
                            Input Sanitizer (injection guard, length limits)
                            Security Layer (phone allowlist, audit log)
                            Ruflo (long-term memory, HTTP mode, shared w/ brana)
                                    ↓
                            Kapso outbound API → WhatsApp reply
```

Kapso requires ≤5s webhook response. brapsoclaw returns 202 Accepted immediately, processes Claude response async, then delivers via Kapso outbound API.

### Key differences from NanoClaw

| Feature | NanoClaw | brapsoclaw |
|---------|----------|------------|
| WhatsApp transport | Baileys (unofficial, Web protocol) | Kapso (official Business API) |
| Message delivery | Polling every 2s | Push webhooks (no latency, no waste) |
| Message types | Text only | Text, media, buttons, templates, flows |
| Isolation | Docker containers per group | Docker + network isolation |
| Bot commands | None | /reset, /status, /compact |
| Security | None | Input sanitization + phone allowlist + audit log |

## Stack

- **Language:** TypeScript (Node.js 22+, ESM)
- **Server:** Express
- **AI:** Anthropic SDK (`@anthropic-ai/sdk`) — Claude API credits
- **WhatsApp:** `@kapso/whatsapp-cloud-api` (webhooks + send)
- **State:** SQLite (`better-sqlite3`) dev → PostgreSQL prod
- **Memory:** Ruflo (`-t http -p 8080`, ruvector PostgreSQL) — shared long-term memory with local brana
- **Deployment:** Docker / Railway

## Billing mode

**API credits required unconditionally.** Set `ANTHROPIC_API_KEY` in env.

Subscription mode is prohibited for automated bots per Anthropic ToS (Feb 2026) — including dev and test environments. There is no supported alternative billing path.

## Client

- **Owner:** martineserios (personal project)
- **Type:** Open-source product
- **Ecosystem:** Part of brana portfolio

## Conventions

- Conventional commits: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`
- Tests: Vitest, test-first
- TypeScript strict mode, ESM modules
- `pnpm` for package management
- No `any` types — use `unknown` + type guards at boundaries
