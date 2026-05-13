---
title: brapsoclaw — Solución Técnica
status: current
created: 2026-05-13
updated: 2026-05-13 (agentfs/Turso + mirage evaluations added)
task: t-28
---

# brapsoclaw — Solución Técnica

> Audiencia: vos en 6 meses. Todo lo que necesitás para entender qué se construyó, por qué, y cómo seguir.

---

## Qué es esto

**brapsoclaw** es el primer ladrillo de **brana-cloud**: un agente personal accesible desde WhatsApp, alimentado por Claude API, con acceso a memoria y proyectos a través de ruflo en la nube.

No es un framework open-source para terceros. Es tu herramienta personal para tener brana contigo en cualquier dispositivo.

**Origen:** Fork/rewrite de [NanoClaw](https://github.com/qwibitai/nanoclaw) que reemplaza Baileys (WhatsApp Web unofficial) con Kapso (WhatsApp Business API oficial).

---

## Problema

Tenés acceso a herramientas poderosas (Claude, brana, proyectos, memoria) pero solo desde la CLI local. Cuando estás en el celular o en otro dispositivo, no tenés contexto ni canal de entrada útil. brapsoclaw resuelve eso.

---

## Arquitectura

```
WhatsApp → Kapso (webhooks) → brapsoclaw (Express/TS) ──[202 Accepted]──→ Kapso
                                    ↓ (async)
                            HMAC-SHA256 verify
                                    ↓
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

### Flujo crítico: 202 Accepted + dead-letter

Kapso exige respuesta al webhook en ≤5s. El handler retorna `202 Accepted` de inmediato y procesa en background:

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

Sin dead-letter, cualquier error en el flujo async produce silencio total para el usuario.

### HMAC-SHA256 en webhooks

Cada request entrante de Kapso se verifica antes de procesar:

```typescript
function verifyHmac(req: Request, secret: string): boolean {
  const sig = req.headers['x-hub-signature-256'] as string;
  const expected = 'sha256=' + createHmac('sha256', secret)
    .update(req.rawBody).digest('hex');
  return timingSafeEqual(Buffer.from(sig), Buffer.from(expected));
}
```

`KAPSO_WEBHOOK_SECRET` en env vars. Falla → `403 Forbidden` sin procesar.

### Infraestructura cloud

```
Tu máquina local                     Cloud (Railway)
──────────────────                   ──────────────────────────────────
brana / Claude Code                  ┌─────────────────────────────┐
  └─ MCP (HTTP) ──────────────────▶  │  ruflo (HTTP MCP server)    │
                                     │  └─ PostgreSQL (Railway PG)  │
                                     └──────────────┬──────────────┘
                                                    │ MCP
                                     ┌──────────────▼──────────────┐
                                     │  brapsoclaw (Node.js/TS)    │
                                     │  └─ Claude API              │
                                     │  └─ Kapso webhooks          │
                                     └──────────────┬──────────────┘
                                                    │
WhatsApp ◀──── Kapso ◀──────────────────────────────┘
```

Una sola fuente de verdad: ruflo cloud + PostgreSQL. brana local y brapsoclaw comparten la misma memoria.

---

## Stack

| Componente | Tecnología | Notas |
|-----------|-----------|-------|
| Lenguaje | TypeScript (Node.js 22+, ESM, strict) | Kapso SDK es TS-first |
| Server | Express | Simple, probado |
| WhatsApp | `@kapso/whatsapp-cloud-api` | BSP oficial, push webhooks |
| AI | `@anthropic-ai/sdk` (Claude Sonnet 4.6) | API credits — subscription bloqueado |
| Session state | SQLite (`better-sqlite3`) dev → PostgreSQL prod | Usar Kysely o Drizzle como abstracción |
| Memoria long-term | Ruflo HTTP + PostgreSQL | Compartido con brana local |
| Deployment | Railway | ~$5-10/mes |
| Package manager | pnpm | |
| Tests | Vitest | Test-first |

### Por qué TypeScript y no Python

Python requería wrapper HTTP sobre Kapso SDK (no existe SDK oficial) y abría brecha de ecosistema. El gain en ML tooling no aplica — esto es conversacional, no ML.

### Por qué no LangGraph

Overhead de orquestación injustificado para un agente de un solo Claude call. Añade dependencias y complejidad sin beneficio.

### ORM/query builder — IMPORTANTE

No escribir SQL raw contra SQLite. La migración dev→prod es un driver swap con Kysely/Drizzle; sin abstracción es un rewrite. Esto está locked en ADR-001.

---

## Billing

**API credits requeridos incondicionalmente.** `ANTHROPIC_API_KEY` en env — dev, staging, y prod.

Anthropic bloqueó subscription quota para bots automatizados en Feb 2026 (ToS). No hay alternativa soportada. Ver `memory/field-note_subscription-mode-blocked.md`.

**Modelo default:** Claude Sonnet 4.6.

---

## Fases de construcción

| Fase | Qué construir | Estado |
|------|--------------|--------|
| **0 — MVP conversacional** | Express + Kapso + Claude API + persona.md estático + session SQLite | Pendiente (t-31 → t-39) |
| **1 — GitHub reads** | Tool `read_project()` via GitHub API | Futuro |
| **2 — Ruflo cloud** | Deploy ruflo en Railway + Postgres, migrar memoria local | Futuro |
| **3 — Conectar local** | Actualizar brana local para apuntar a ruflo cloud | Futuro |
| **4 — Memory tools** | `memory_search()`, `memory_store()` en brapsoclaw | Futuro (depende Fase 2+3) |
| **5 — Write tools** | `create_task()`, `append_log()` via GitHub | Futuro (depende Fase 1) |
| **6 — Web search** | Brave/Perplexity tool | Futuro |
| **7 — Canales + Calendar** | Web UI, Google Calendar, Telegram | Futuro lejano |

**Estado actual:** Diseño y documentación completa. Código por construir desde Fase 0.

### Tareas concretas de la Fase 0 (backlog)

| Task | Descripción |
|------|-------------|
| t-31 | Scaffold TypeScript — pnpm init, tsconfig strict+ESM, deps |
| t-32 | Webhook handler — HMAC verify, 202 Accepted, parse payload |
| t-33 | Session manager — Map<phone, Message[]> + SQLite persist |
| t-34 | Persona loader — lee `agent/persona.md` al arrancar |
| t-35 | Claude runner — tool loop con Anthropic SDK |
| t-36 | Command router — detecta `/reset`, `/status`, `/deep`, etc. |
| t-37 | WA formatter — convierte markdown de Claude a formato WhatsApp |
| t-38 | Config/env — KAPSO_API_KEY, KAPSO_WEBHOOK_SECRET, ANTHROPIC_API_KEY |
| t-39 | E2E test local — ngrok expone /webhook a internet |

---

## Ruflo Cloud — Path de migración

Ruflo ya soporta HTTP mode y PostgreSQL backend natively — no hay que construir sync desde cero.

**Pasos:**

1. `ruflo ruvector setup` → provisiona infra PostgreSQL en Railway
2. `ruflo ruvector import` → migra memoria local (`patterns-export.json`) a PostgreSQL
3. Deploy ruflo en Railway con `ruflo mcp start -t http -p 8080`
4. Actualizar settings de brana local: MCP stdio → MCP HTTP (apunta al endpoint Railway)
5. brapsoclaw se conecta al mismo endpoint como MCP client

**Namespaces en ruflo:**
- `pattern` — learnings de sesiones (/brana:close, /brana:retrospective) — IRREEMPLAZABLES
- `knowledge` — dimension docs (~590 secciones) — regenerables desde git
- `skills` — skill frontmatter — regenerables desde git
- `decisions` — ADRs

---

## Tools del agente (diseño — Fases 4-6)

| Tool | Hace | Fuente |
|------|------|--------|
| `memory_search(query)` | Busca en ruflo cloud | ruflo MCP |
| `memory_store(key, value)` | Guarda aprendizaje/nota | ruflo MCP |
| `read_project(name)` | Lee estado de un repo | GitHub API |
| `create_task(project, task)` | Crea tarea en tasks.json | GitHub API (commit) |
| `append_log(entry)` | Agrega a event-log | GitHub API (commit) |
| `search_web(query)` | Búsqueda web | Brave/Perplexity API |
| `get_calendar()` | Agenda del día | Google Calendar API |

MVP (Fase 0) arranca sin tools — solo conversación con persona context estático.

### Acceso a archivos remotos (Fases 4-6)

Para que brapsoclaw lea los `.md` de brana-knowledge desde cloud hay dos opciones evaluadas:

| Opción | Cuándo usar |
|--------|------------|
| **GitHub API** (`read_project()`) | Acceso a archivos específicos por path — ya está en el diseño, cero infra extra |
| **Mirage + S3** (`@struktoai/mirage-node`) | Si el agente necesita `grep`/`find`/navegación sobre los archivos — monta un S3 bucket como filesystem POSIX, el agente opera con comandos Unix estándar |

Mirage no reemplaza ruflo (búsqueda semántica) — son complementarios: ruflo para "encontrá lo relevante", mirage para "dame exactamente este archivo o buscá literalmente esta cadena".

---

## Persona Context

**Fase 0 (MVP):** Archivo `agent/persona.md` en el repo. Se carga al inicio de cada conversación via `getSystemPrompt()`.

**Futuro (Fase 4):** Al inicio de cada conversación, el agente ejecuta `memory_search("who is martin")` y construye el contexto desde ruflo. Se mantiene actualizado automáticamente.

Diseño: ver `feedback/two-layer-persona-context` en memory — Capa 1 (estático) + Capa 2 (ruflo lookup dinámico).

---

## UX — Interfaz WhatsApp

### Comandos

| Comando | Hace |
|---------|------|
| `/log [texto]` | Loguea en event-log del proyecto relevante |
| `/task [descripción]` | Crea tarea en el proyecto que corresponda |
| `/reset` | Limpia historial de conversación |
| `/status` | Estado del agente (modelo, sesión, tools activos) |
| `/projects` | Lista proyectos con estado |
| `/chunks` | Toggle chunking on/off/auto |
| `/deep` | Próxima respuesta usa Opus 4.7 (razonamiento profundo) |

### Idioma

Adapta al idioma del mensaje entrante. Sin preferencia fija.

### Formato WhatsApp

WhatsApp no es markdown estándar. El formatter convierte:

| Input (Claude) | Output (WhatsApp) |
|----------------|------------------|
| `## Header` | `*HEADER*` |
| `**bold**` | `*bold*` |
| Tablas | Listas de texto |
| `[texto](url)` | `texto: url` |
| Código inline | `` ```código``` `` |

### Chunking

Default: **auto** — chunking inteligente por párrafo si la respuesta supera ~800 chars.

- `/chunks on` → siempre múltiples mensajes
- `/chunks off` → siempre un solo mensaje
- `/chunks auto` → vuelve al default

Preferencia persistida en SQLite por número de teléfono (estado per-contact — no config global). Ver `memory/feedback_whatsapp-per-contact-formatting.md`.

---

## Costo estimado mensual

Supuesto: 30 conversaciones/día, ~800 tokens promedio (500 input + 300 output)

```
30 conv × 800 tokens × 30 días = 720K tokens/mes
Sonnet 4.6: $3/MTok input + $15/MTok output
= ~$1.50 input + ~$3.30 output = ~$4.80/mes Claude
```

| Componente | Costo estimado |
|-----------|---------------|
| Claude Sonnet 4.6 | ~$4.80/mes |
| Railway (server + Postgres) | ~$5-10/mes |
| Kapso | Free tier (suficiente para uso personal) |
| **Total** | **~$10-15/mes** |

---

## Preguntas abiertas

- [ ] Auth brapsoclaw → ruflo cloud: API key en header vs JWT
- [ ] Persona context: estructura exacta del `agent/persona.md`
- [ ] Historial de conversación: ventana deslizante vs compresión con `/compact`
- [ ] ORM/query builder definitivo: Kysely vs Drizzle (debe decidirse antes de t-33)
- [ ] **Evaluar agentfs/Turso en t-33** — agentfs da KV store per-contact + tool_calls audit-log insert-only gratis. Turso como backing eliminaría el riesgo SQLite→PostgreSQL de ADR-001 (mismo driver dev/prod, sync a cloud). Blocker: historial de conversación (Message[]) no encaja nativamente en KV — evaluar en el momento de construir t-33, no antes.

---

## Referencias

- `docs/decisions/ADR-001-stack-arquitectura-brana-cloud.md` — decisiones de arquitectura locked
- `docs/ideas/personal-assistant-kapso.md` — brainstorm original con diseño completo
- `.claude/CLAUDE.md` — convenciones del proyecto
- `memory/field-note_subscription-mode-blocked.md` — billing constraint
- `memory/feedback_library-swap-requires-abstraction.md` — ORM warning
- `memory/feedback_reject-mirror-architectures.md` — por qué no Supabase espejo
- `memory/feedback_whatsapp-per-contact-formatting.md` — per-contact formatting
