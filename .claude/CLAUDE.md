# brapsoclaw

> NanoClaw reimagined on Kapso — a TypeScript WhatsApp bot framework using official WhatsApp Business API instead of Baileys.

## What is this?

A fork/rewrite of [NanoClaw](https://github.com/qwibitai/nanoclaw) that replaces the Baileys (unofficial WhatsApp Web) transport with [Kapso](https://docs.kapso.ai/) (official WhatsApp Business API). Stays in TypeScript to keep the Kapso SDK and minimize delta from the original codebase.

## Architecture

```
WhatsApp → Kapso (webhooks) → brapsoclaw (Express/TS) → Anthropic SDK → Claude
                                    ↓
                            Session Manager (phone → conversation history)
                            Message Formatter (4096 char batching, WA markdown)
                            Input Sanitizer (injection guard, length limits)
                            Security Layer (phone allowlist, audit log)
```

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
- **Deployment:** Docker / Railway

## Billing mode

Two modes available at runtime via env:

| Mode | How | Cost | When |
|------|-----|------|------|
| API credits | `ANTHROPIC_API_KEY` set | Per-token billing | Production |
| Subscription | No API key, `claude auth login` done | Subscription quota | Local dev/testing only |

**Do not run subscription mode in production.** ToS gray area — personal quota shared with your own Claude usage. API credits are the correct choice for any always-on deployment.

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
