# brapsoclaw — Errata & Corrections

Spec mismatches found during sessions. Resolved via `/brana:maintain-specs`.

## Summary Table

| ID | Severity | Status | Title |
|----|----------|--------|-------|
| E2026-05-08-1 | Medium | resolved | CLAUDE.md billing mode allows subscription for dev — blocked since Feb 2026 |
| E2026-05-08-2 | Medium | resolved | CLAUDE.md stack table omits ruflo as production dependency |
| E2026-05-08-3 | Low | resolved | CLAUDE.md architecture diagram missing 202 Accepted async pattern |

---

## E2026-05-08-1: CLAUDE.md billing mode allows subscription for dev

**Severity:** Medium  
**Status:** pending (fix: t-29)  
**Discovery:** 2026-05-08 brainstorm session — validated via brana-knowledge dim-36 (claw-ecosystem)  
**Affected files:** `.claude/CLAUDE.md` — Billing mode table

**Spec says:** Billing mode table states subscription mode is OK for "Local dev/testing only"  
**Reality:** Anthropic blocked subscription quota for automated bots Feb 2026, ToS violation — even dev/test contexts prohibited. API credits (`ANTHROPIC_API_KEY`) required unconditionally.  
**Fix:** Replace billing table row with: "API credits required. Subscription mode prohibited for automated bots per Anthropic ToS (Feb 2026), including dev."

---

## E2026-05-08-2: CLAUDE.md stack table omits ruflo as production dependency

**Severity:** Medium  
**Status:** pending (fix: t-29)  
**Discovery:** 2026-05-08 brainstorm session — brana-cloud architecture decision  
**Affected files:** `.claude/CLAUDE.md` — Stack section

**Spec says:** Stack lists Express, Anthropic SDK, Kapso, SQLite/PostgreSQL — ruflo absent  
**Reality:** Ruflo cloud (HTTP mode `-t http -p 8080`, PostgreSQL via ruvector) is the shared long-term memory layer for both local brana and brapsoclaw  
**Fix:** Add ruflo to stack table; add "Memory layer" subsection under Architecture with the cloud topology

---

## E2026-05-08-3: CLAUDE.md architecture diagram missing 202 Accepted async pattern

**Severity:** Low  
**Status:** pending (fix: t-29)  
**Discovery:** 2026-05-08 UX design — Kapso 5-second webhook response window  
**Affected files:** `.claude/CLAUDE.md` — Architecture diagram

**Spec says:** Diagram shows linear: Kapso → brapsoclaw → Claude API  
**Reality:** Kapso requires ≤5s webhook response. Pattern: immediate 202 Accepted back to Kapso, Claude processes async in background, then sends reply via Kapso outbound API  
**Fix:** Update diagram to show two-arrow flow: sync 202 ack + async delayed send
