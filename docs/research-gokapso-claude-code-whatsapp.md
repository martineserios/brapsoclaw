# Research: gokapso/claude-code-whatsapp
**Date:** 2026-03-16
**Sources checked:** 25+
**Findings:** 14 HIGH, 10 MEDIUM, 4 LOW (across 8 scouts)

---

## Findings

### [CONFIRMED-INTERNAL] gokapso/claude-code-whatsapp Architecture — HIGH
- **Source:** [GitHub](https://github.com/gokapso/claude-code-whatsapp)
- **Affects:** [dim 36](~/enter_thebrana/brana-knowledge/dimensions/36-claw-ecosystem-chat-interface.md), ADR-019
- **Detail:** 26 stars, 1 fork, last updated Jan 2026. ~600-900 LOC across 13 files. Split architecture: Express webhook server (src/) + E2B sandbox (e2b-server/). Session keyed by phone number. Commands: /reset, /compact, /clear, /status.
- **Action:** Use as reference for brapsoclaw, but collapse split architecture (no E2B).

### [ANSWERS-INTERNAL] NanoClaw Baileys confined to 3 files — HIGH
- **Source:** [DeepWiki](https://deepwiki.com/gavrielc/nanoclaw), [GitHub](https://github.com/qwibitai/nanoclaw)
- **Affects:** brapsoclaw architecture decision
- **Detail:** Baileys touches only: src/index.ts (makeWASocket, messages.upsert, sock.sendMessage), src/auth.ts (QR), store/auth_info_baileys/. The ENTIRE container/agent-runner/, IPC system, group-queue, task-scheduler, db.ts are transport-agnostic. Swap scope: ~150-200 lines.
- **Action:** Fork NanoClaw. Modify 3 files, delete 2. Keep everything else.

### [ANSWERS-INTERNAL] NanoClaw is polling-based, not event-driven — HIGH
- **Source:** [nanoclaws.io](https://nanoclaws.io/blog/whatsapp-ai-bot-2026-complete-guide)
- **Affects:** brapsoclaw architecture
- **Detail:** startMessageLoop() polls getNewMessages() every 2000ms via Baileys. Kapso webhooks are push-based. This is a fundamental improvement, not just a transport swap — eliminates polling latency and reduces resource usage.
- **Action:** Replace polling loop with Kapso webhook handler (Express POST /webhook).

### [ANSWERS-INTERNAL] E2B NOT needed — Anthropic recommends containers — HIGH
- **Source:** [Anthropic Agent SDK docs](https://platform.claude.com/docs/en/agent-sdk/hosting)
- **Affects:** ADR-019, brapsoclaw cost model
- **Detail:** Official guidance: ephemeral/persistent Docker containers with Linux namespaces. E2B costs $150+/mo (Pro). Docker on a $30-50 VPS is cheaper. gVisor for stronger isolation.
- **Action:** Use Docker containers per Anthropic guidance. No E2B dependency.

### [CONFIRMED-INTERNAL] Kapso SDK webhook pattern — HIGH
- **Source:** [Kapso docs](https://docs.kapso.ai/docs/how-to/whatsapp/set-up-webhooks)
- **Detail:** normalizeWebhook() from @kapso/whatsapp-cloud-api/server. Events: message.received/sent/delivered/read/failed. HMAC-SHA256. Must respond within 5 seconds. Idempotency via X-Idempotency-Key.
- **Action:** Use normalizeWebhook() + verifySignature() in brapsoclaw webhook handler.

### [SECURITY] WhatsApp prompt injection — "Lethal Trifecta" — HIGH
- **Source:** [Docker blog](https://www.docker.com/blog/mcp-horror-stories-whatsapp-data-exfiltration-issue/), [Invariant Labs](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)
- **Detail:** brapsoclaw has all 3: sensitive data + untrusted content + exfiltration tools (send_message). Attack vectors: direct injection via messages, tool poisoning via MCP, dynamic tool replacement. send_message is both core feature and highest-risk surface.
- **Action:** Implement defense-in-depth: input sanitization, phone allowlist for send_message, credential proxy pattern, network isolation, tool call audit log.

### [SECURITY] MCP tool poisoning — HIGH
- **Source:** [Anthropic secure deployment](https://platform.claude.com/docs/en/agent-sdk/secure-deployment)
- **Detail:** Compromised MCP server can embed hidden instructions in tool descriptions. Only self-written or Anthropic-provided servers should be used. No automated vetting exists.
- **Action:** Audit all MCP servers. Review tool descriptions for hidden instructions.

### NanoClaw Docker deal + ecosystem — LOW
- **Source:** [TechCrunch](https://techcrunch.com/2026/03/13/the-wild-six-weeks-for-nanoclaws-creator-that-led-to-a-deal-with-docker/)
- **Detail:** Gavriel Cohen signed Docker deal 2026-03-13. Active development (last commit Mar 14). Forks: OpenLobster, opencode-nanoclaw. Competitors: ZeroClaw (Rust), NullClaw (Zig), NemoClaw (enterprise), PicoClaw (minimal).

---

## Contradictions with Internal Docs

### CONTRADICTION: "swap 2 files" oversimplified
- **Finding:** Deep dive shows NanoClaw needs 3 files modified + 2 deleted. gokapso has 6 source files with a completely different architecture (split process model).
- **Internal:** [dim 36](~/enter_thebrana/brana-knowledge/dimensions/36-claw-ecosystem-chat-interface.md) line 144: "Brana adaptation: swap 2 files (kapso.ts → CLI adapter)"
- **Impact:** Low — the direction is correct (Kapso swap is surgical), the number was optimistic. Actual scope: ~150-200 lines changed.
- **Action:** Update dim-36 line 144 to reflect accurate scope.

---

## Architecture Decision: NanoClaw Fork vs gokapso Reference

| Dimension | Fork NanoClaw | Adapt gokapso |
|-----------|---------------|---------------|
| LOC to change | ~150-200 lines (3 files) | ~400 lines (collapse E2B split) |
| What you keep | Container isolation, IPC, group memory, scheduler, Agent SDK | Kapso webhook handler, formatter, session map |
| What you lose | Nothing (Kapso replaces Baileys cleanly) | E2B dependency (remove), GitHub flow (optional) |
| Architecture | Push (webhooks) into existing pull loop | Already push-based |
| Isolation | Per-group Docker containers | Per-user E2B (replace with Docker) |
| Memory model | CLAUDE.md per group + SQLite | In-memory Map (add persistence) |

**Recommendation: Fork NanoClaw + cherry-pick from gokapso.** NanoClaw has the better foundation (container isolation, IPC, group memory, scheduler). gokapso has the better Kapso integration patterns (normalizeWebhook, formatter). Combine both.

---

## brapsoclaw Implementation Roadmap

### Phase 1: Transport Swap (Baileys → Kapso)
1. Fork NanoClaw
2. Replace src/index.ts: makeWASocket() → Express + Kapso webhook handler
3. Replace src/index.ts: sock.sendMessage() → kapso.sendMessage()
4. Replace src/ipc.ts: delivery call → kapso.sendMessage()
5. Delete src/auth.ts + store/auth_info_baileys/
6. Add src/formatter.ts (from gokapso — WhatsApp 4096 char batching)
7. Add .env: KAPSO_API_KEY, PHONE_NUMBER_ID, WEBHOOK_SECRET

### Phase 2: Architecture Improvements
8. Replace polling loop with webhook push (fundamental upgrade)
9. Add webhook signature verification (verifySignature)
10. Add idempotency tracking (X-Idempotency-Key)
11. Add 5-second response timeout handling (async + 202 Accepted)

### Phase 3: Security Hardening
12. Input sanitization layer (regex patterns for injection attempts)
13. send_message phone number allowlist
14. Credential proxy pattern (API keys outside agent boundary)
15. Container hardening (--cap-drop ALL, --network none, --read-only)
16. Tool call audit log (append-only)

### Phase 4: Enhanced Features
17. Multi-channel support (web widget, CLI — channel adapter pattern from ADR-019)
18. Tiered access (end user / client / operator — from ADR-019)
19. Persistent session storage (SQLite → Postgres)
20. Cost tracking per conversation

---

## New Sources Discovered
- [dzhng/claude-agent-server](https://github.com/dzhng/claude-agent-server) — E2B + Claude pattern (gokapso mirrors this)
- [Enriquefft/openclaw-kapso-whatsapp](https://github.com/Enriquefft/openclaw-kapso-whatsapp) — OpenClaw + Kapso integration
- [nanoclaws.io](https://nanoclaws.io/) — Official NanoClaw docs site
- [NanoClaw DeepWiki](https://deepwiki.com/gavrielc/nanoclaw) — Architecture diagrams

## Registry Updates Proposed
- Update gokapso-github: last_checked → 2026-03-16
- Update kapso-ai-docs: last_checked → 2026-03-16
- Add source: nanoclaws-io (type: docs, creator: nanoclaw, trust: promising)
- Add source: nanoclaw-deepwiki (type: reference, trust: promising)
