# brapsoclaw

> NanoClaw reimagined on Kapso — a robust WhatsApp bot framework using official WhatsApp Business API instead of Baileys.

## What is this?

A fork/rewrite of [NanoClaw](https://github.com/qwibitai/nanoclaw) that replaces the Baileys (unofficial WhatsApp Web) transport with [Kapso](https://docs.kapso.ai/) (official WhatsApp Business API). More robust, production-ready, and compatible with business WhatsApp accounts.

## Architecture

```
WhatsApp → Kapso (webhooks) → brapsoclaw (FastAPI) → Claude API (Agent SDK)
                                    ↓
                            Session Manager (per-conversation memory)
                            Tool Router (MCP, skills, functions)
                            Persona System (configurable agents)
```

### Key differences from NanoClaw

| Feature | NanoClaw | brapsoclaw |
|---------|----------|------------|
| WhatsApp transport | Baileys (unofficial, Web protocol) | Kapso (official Business API) |
| Message types | Text only | Text, media, buttons, templates, flows |
| Isolation | Docker containers | Kapso-managed + optional sandboxing |
| Team inbox | None | Kapso team inbox integration |
| Workflows | Baked-in cron | Kapso workflow nodes + external triggers |
| Deployment | Self-hosted only | Kapso cloud + self-hosted hybrid |

## Stack

- **Language:** Python (FastAPI)
- **AI:** Claude API via Anthropic SDK / Agent SDK
- **WhatsApp:** Kapso webhooks + SDK
- **State:** SQLite (dev) → PostgreSQL (prod)
- **Deployment:** Railway / Docker

## Client

- **Owner:** martineserios (personal project)
- **Type:** Open-source product
- **Ecosystem:** Part of brana portfolio

## Conventions

- Conventional commits: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`
- Tests: pytest, test-first
- Python: 3.12+, uv for package management
- Type hints everywhere
