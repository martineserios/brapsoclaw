---
id: ADR-001
title: Stack y arquitectura brana-cloud (brapsoclaw Phase 1)
status: accepted
created: 2026-05-09
updated: 2026-05-09
deciders: martineserios
tags: [stack, architecture, security, async]
---

# ADR-001: Stack y arquitectura brana-cloud

## Contexto

brapsoclaw es el primer ladrillo de brana-cloud: un agente personal accesible desde WhatsApp, alimentado por Claude API, con acceso a memoria y proyectos a través de ruflo cloud. El objetivo es que brana sea un sistema vivo accesible desde cualquier dispositivo.

Las restricciones principales al momento de la decisión:
- Kapso exige respuesta al webhook en ≤5s.
- Subscription mode de Anthropic está bloqueado para bots automatizados desde Feb 2026 (ToS).
- Ruflo ya soporta HTTP mode + PostgreSQL natively — no hay que construir sync desde cero.
- El challenger pre-ADR identificó dos gaps críticos de producción que el spec original no cubría: dead-letter en el flujo async y verificación HMAC del webhook entrante.

## Decisiones

### 1. Lenguaje y runtime: TypeScript + Node.js 22+

**Decisión:** TypeScript ESM, strict mode, `pnpm`.

**Por qué:** El Kapso SDK es TypeScript-first (`@kapso/whatsapp-cloud-api`). Mantenerse en TS minimiza el delta respecto al codebase original (NanoClaw). Node.js 22+ tiene soporte nativo ESM, fetch, y performance suficiente para el volumen esperado (~30 conv/día).

**Alternativa descartada — Python:** Requería wrapper HTTP sobre Kapso SDK (no existe SDK Python oficial) y abría una brecha de ecosistema (tests, tipos, deploy). El gain en ML tooling no aplica al caso de uso.

**Alternativa descartada — LangGraph:** Overhead de orquestación injustificado para un agente conversacional de un solo Claude call. Añade dependencias, complejidad de state machine, y cost de mantenimiento sin beneficio para este caso.

### 2. Transport WhatsApp: Kapso (oficial) en lugar de Baileys (unofficial)

**Decisión:** `@kapso/whatsapp-cloud-api` — webhooks entrantes + send outbound.

**Por qué:** Baileys usa el protocolo WhatsApp Web no documentado — Anthropic ToS y Meta ToS prohíben su uso en bots de producción. Kapso es BSP oficial con WhatsApp Business API. Push webhooks eliminan el polling de 2s de NanoClaw.

### 3. Patrón async 202 Accepted — con dead-letter (t-54)

**Decisión:** El handler del webhook retorna `202 Accepted` inmediatamente. El procesamiento Claude + envío de respuesta ocurre en background. En caso de fallo de Claude API o Kapso outbound: 1 retry automático; si falla definitivamente, enviar mensaje de error plain-text al usuario vía Kapso outbound.

**Por qué el dead-letter es requerido:** Sin él, cualquier error en el flujo async (rate limit, timeout de Claude, fallo de red a Kapso) produce un silencio total para el usuario. El gap fue identificado en el challenger pre-ADR-001 (t-54). No implementarlo significa mensajes perdidos sin trazabilidad.

**Implementación mínima:**
```typescript
async function processAsync(phone: string, text: string) {
  try {
    const reply = await callClaude(text);
    await sendKapso(phone, reply);
  } catch {
    await retryOnce(() => callClaude(text).then(r => sendKapso(phone, r)))
      .catch(() => sendKapso(phone, "⚠️ No pude procesar tu mensaje. Intentá de nuevo."));
  }
}
```

### 4. Verificación HMAC-SHA256 en webhooks entrantes (t-55)

**Decisión:** Verificar la firma `X-Hub-Signature-256` de cada request entrante de Kapso antes de procesar el payload.

**Por qué es requerido:** Sin verificación HMAC, cualquier actor externo puede enviar requests al webhook endpoint e inyectar mensajes falsos. El gap fue identificado en el challenger pre-ADR-001 (t-55). Esto no es hardening opcional — es el boundary de trust entre internet y el agente.

**Implementación mínima:**
```typescript
function verifyHmac(req: Request, secret: string): boolean {
  const sig = req.headers['x-hub-signature-256'] as string;
  const expected = 'sha256=' + createHmac('sha256', secret)
    .update(req.rawBody).digest('hex');
  return timingSafeEqual(Buffer.from(sig), Buffer.from(expected));
}
```

Secreto configurado en `KAPSO_WEBHOOK_SECRET` env var. Si falla, responder `403 Forbidden` sin procesar.

### 5. Billing: API credits only

**Decisión:** `ANTHROPIC_API_KEY` requerida en todas las instancias (dev, staging, prod). Subscription mode prohibido.

**Por qué:** Anthropic bloqueó subscription quota para bots automatizados en Feb 2026 — ToS violation incluso en dev. No hay alternativa soportada. Ver field note `memory/field-note_subscription-mode-blocked.md`.

**Modelo default:** Claude Sonnet 4.6. Costo estimado: ~$4.80/mes a 30 conv/día × 800 tokens.

### 6. Estado conversacional: SQLite (dev) → PostgreSQL (prod)

**Decisión:** `better-sqlite3` en desarrollo. PostgreSQL (Railway o Supabase) en producción.

**Por qué:** SQLite es zero-config para dev local. La migración a PostgreSQL es un driver swap — requiere nombrar el ORM/query builder desde el día 1 para evitar un rewrite en Phase 2. Usar `Kysely` o `Drizzle` como abstracción.

**Riesgo registrado:** Si se escribe SQL raw contra SQLite y se asume compatibilidad, la migración a PostgreSQL es un rewrite, no un swap. Ver feedback `memory/feedback_library-swap-requires-abstraction.md`.

### 7. Memoria de largo plazo: Ruflo cloud (HTTP mode + PostgreSQL)

**Decisión:** Ruflo desplegado en Railway con backend PostgreSQL (ruvector). Accesible vía MCP HTTP desde brapsoclaw y desde brana local.

**Por qué:** Ruflo ya soporta `ruflo mcp start -t http -p 8080` y `ruflo ruvector setup/import`. Elimina la necesidad de un sync daemon. Una sola fuente de verdad compartida entre brana local y brapsoclaw.

**Alternativa descartada — Supabase espejo:** Dos writers sobre schemas solapados → conflictos inevitables. Ver feedback `memory/feedback_reject-mirror-architectures.md`.

**Path de migración:**
1. `ruflo ruvector setup` → infra PostgreSQL
2. `ruflo ruvector import` → migra memoria local a Postgres
3. Deploy ruflo en Railway
4. Actualizar brana local: MCP stdio → MCP HTTP (apunta a cloud)
5. brapsoclaw conecta al mismo endpoint

---

## Preguntas abiertas

- [ ] Auth brapsoclaw → ruflo cloud: API key en header vs JWT
- [ ] Persona context: estructura exacta del `persona.md` (ver `agent/persona.md`)
- [ ] Historial de conversación: ventana deslizante vs compresión con `/compact`
- [ ] ORM/query builder para el swap SQLite → PostgreSQL (Kysely vs Drizzle)

---

## Consecuencias

- **t-54** (dead-letter async) y **t-55** (HMAC verification) son prerequisitos de producción. El ADR los documenta como decisiones de arquitectura, no como hardening posterior.
- El ADR no puede considerarse "signed off" hasta que t-54 y t-55 estén implementados y en tests.
- Personas futuras que lean CLAUDE.md deben ver el Status column en la tabla de diferencias (E2026-05-09-1 aplicada).

---

## Referencias

- `docs/ideas/personal-assistant-kapso.md` — brainstorm original
- `docs/corrections.md` — errata E2026-05-08-1/2/3, E2026-05-09-1
- `memory/field-note_subscription-mode-blocked.md`
- `memory/feedback_reject-mirror-architectures.md`
- `memory/feedback_library-swap-requires-abstraction.md`
- Backlog: t-54, t-55, t-27
