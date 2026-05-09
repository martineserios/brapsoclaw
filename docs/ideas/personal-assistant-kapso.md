---
title: brapsoclaw — brana-cloud personal assistant
status: draft
created: 2026-05-08
updated: 2026-05-08
---

# brapsoclaw — Sistema brana-cloud: asistente personal vía WhatsApp

> Brainstormed 2026-05-08. Diseño en progreso — no apurar backlog hasta que esté claro.

## Problema

Tengo acceso a herramientas poderosas (Claude, brana, proyectos, memoria) pero solo desde la CLI local.
Cuando estoy en el celular o en otro dispositivo, no tengo contexto ni canal de entrada útil.
Quiero que brana sea un sistema vivo, en la nube, accesible desde cualquier lugar.

## Qué es brapsoclaw en este contexto

Primer ladrillo de **brana-cloud**: un agente personal que vive en WhatsApp vía Kapso,
alimentado por Claude API, con acceso a mi memoria y proyectos a través de ruflo en la nube.
No es un framework open-source — es mi herramienta personal.

---

## Arquitectura — Diseño actual

### Hallazgo clave sobre ruflo (2026-05-08)

Ruflo ya soporta dos capacidades críticas para cloud:

1. **Modo HTTP:** `ruflo mcp start -t http -p 8080` — no es solo stdio local
2. **Backend PostgreSQL (ruvector):** `ruflo ruvector setup/import/init` — puede usar Postgres en vez de SQLite

Esto elimina la necesidad de un sync daemon. La solución correcta es **ruflo como servicio cloud**.

### Diagrama

```
Tu máquina local                     Cloud (Railway)
──────────────────                   ──────────────────────────────────
brana / Claude Code                  ┌─────────────────────────────┐
  └─ MCP (HTTP) ──────────────────▶  │  ruflo (HTTP MCP server)    │
                                     │  └─ PostgreSQL (Supabase /   │
                                     │     Railway Postgres)        │
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

**Una sola fuente de verdad: ruflo cloud + PostgreSQL.**
- Tu brana local escribe/lee memoria → ruflo cloud
- brapsoclaw lee/escribe memoria → mismo ruflo cloud
- Sin sync daemons. Sin conflictos. Sin lag de archivos.

---

## Memoria — Cómo funciona hoy y cómo cambia

### Estado actual (local)
```
~/.swarm/memory.db          ← ruflo SQLite (in-memory, flush cada 60s)
  Namespaces:
    pattern   → learnings de sesiones (/brana:close, /brana:retrospective) — IRREEMPLAZABLES
    knowledge → dimension docs (~590 secciones) — regenerables desde git
    skills    → skill frontmatter — regenerables desde git
    decisions → ADRs

system/state/patterns-export.json  ← JSON export semanal (git-tracked)
```

### Estado objetivo (cloud)
```
Cloud ruflo (HTTP) + PostgreSQL
  └─ mismos namespaces, mismo protocolo MCP
  └─ accesible desde local brana Y desde brapsoclaw
  └─ migración: ruflo ruvector import --input patterns-export.json
```

### Path de migración
1. `ruflo ruvector setup` → genera infra PostgreSQL
2. `ruflo ruvector import` → migra memory actual a Postgres
3. Deploy ruflo en Railway con backend Postgres
4. Actualizar settings local: MCP stdio → MCP HTTP (apunta a cloud ruflo)
5. brapsoclaw se conecta al mismo endpoint como MCP client

---

## Tools del agente (diseño)

| Tool | Hace | Fuente de datos |
|------|------|-----------------|
| `memory_search(query)` | Busca en ruflo cloud | ruflo MCP |
| `memory_store(key, value)` | Guarda aprendizaje/nota | ruflo MCP |
| `read_project(name)` | Lee estado de un repo | GitHub API |
| `create_task(project, task)` | Crea tarea en tasks.json | GitHub API (commit) |
| `append_log(entry)` | Agrega a event-log | GitHub API (commit) |
| `search_web(query)` | Búsqueda web | Brave/Perplexity API |
| `get_calendar()` | Agenda del día | Google Calendar API |

---

## Persona Context — Diseño pendiente

Qué sabe el agente sobre mí. Dos enfoques a decidir:

**Estático (MVP):** Archivo `persona.md` en el repo. Se carga al inicio de cada conversación.
Pros: simple, sin dependencias. Contras: se desactualiza.

**Dinámico (futuro):** Al inicio de cada conversación, el agente ejecuta `memory_search("who is martin")`
y construye el contexto desde ruflo. Se mantiene actualizado automáticamente.

El borrador del persona context está en la conversación del brainstorm — pendiente de escribir a archivo.

---

## Fases de construcción

| Fase | Qué construir | Dependencias |
|------|--------------|--------------|
| **0 — MVP conversacional** | Express + Kapso + Claude API + persona.md estático + session SQLite | Ninguna |
| **1 — GitHub reads** | Tool `read_project()` via GitHub API | Fase 0 |
| **2 — Ruflo cloud** | Deploy ruflo en Railway + Postgres, migrar memoria local | Fase 0 |
| **3 — Conectar local** | Actualizar brana local para apuntar a ruflo cloud | Fase 2 |
| **4 — Memory tools** | `memory_search()`, `memory_store()` en brapsoclaw | Fase 2+3 |
| **5 — Write tools** | `create_task()`, `append_log()` via GitHub | Fase 1 |
| **6 — Web search** | Brave/Perplexity tool | Fase 0 |
| **7 — Canales + Calendar** | Web UI, Google Calendar, Telegram | Futuro |

---

## Stack

- **Server:** Node.js 22+ + TypeScript + Express
- **WhatsApp:** `@kapso/whatsapp-cloud-api`
- **AI:** `@anthropic-ai/sdk` (Claude Sonnet 4.6 / Opus 4.7)
- **Session memory (MVP):** SQLite local (`better-sqlite3`)
- **Long-term memory:** ruflo cloud (HTTP MCP) + PostgreSQL
- **Deployment:** Railway (~$5-10/mes server + Postgres)
- **Context:** GitHub API + ruflo cloud

## Restricciones / hallazgos clave

- Subscription mode **BLOQUEADO** por Anthropic (Feb 2026). Requiere API credits.
- Kapso es infraestructura (delivery), no el cerebro.
- Ruflo ya soporta HTTP + PostgreSQL — no hay que buildear sync desde cero.
- Opción C (Supabase espejo) descartada: dos fuentes de verdad → conflictos inevitables.
- OpenClaw: security nightmare, no usar.

---

---

## UX — Diseño (decidido 2026-05-08)

### Comandos
Modelo híbrido: lenguaje natural para consultas + comandos cortos para acciones frecuentes.

| Comando | Hace |
|---------|------|
| `/log [texto]` | Loguea en event-log del proyecto relevante |
| `/task [descripción]` | Crea tarea en el proyecto que corresponda |
| `/reset` | Limpia historial de conversación |
| `/status` | Estado del agente (modelo, sesión, tools activos) |
| `/projects` | Lista todos los proyectos con estado |
| `/chunks` | Toggle chunking on/off/auto |
| `/deep` | Próxima respuesta usa Opus 4.7 (razonamiento profundo) |

### Idioma
Adapta al idioma del mensaje. Sin preferencia fija.

### Formato de respuestas (WhatsApp)
WhatsApp NO es markdown estándar. El formatter convierte:
- `## Header` → `*HEADER*` (bold)
- `**bold**` → `*bold*`
- Tablas → listas de texto
- Links `[texto](url)` → `texto: url`
- Código inline → ` ```código``` `

### Chunking (mensajes largos)
Default: **auto** — chunking inteligente por párrafo si la respuesta supera ~800 chars.
Controlable con `/chunks`:
- `/chunks on` → siempre múltiples mensajes
- `/chunks off` → siempre un solo mensaje
- `/chunks auto` → vuelve al default

Preferencia persistida en SQLite por número de teléfono.

### Async response pattern
Kapso requiere responder en 5 segundos. El agente:
1. Responde `202 Accepted` a Kapso inmediatamente
2. Procesa Claude API en background
3. Envía la respuesta cuando está lista (Kapso permite delayed send)

---

## Modelo

**Default: Claude Sonnet 4.6** — velocidad + costo óptimos para conversación.
**Opus 4.7 on-demand** — activado con `/deep` para razonamiento complejo.

### Costo estimado mensual
Supuesto: 30 conversaciones/día, ~800 tokens promedio (500 input + 300 output)

```
30 conv × 800 tokens × 30 días = 720K tokens/mes
Sonnet 4.6: $3/MTok input + $15/MTok output
= ~$1.50 input + ~$3.30 output = ~$4.80/mes
```

Total estimado: **~$10-15/mes** (Sonnet API + Railway + Kapso free tier)

---

## Preguntas de diseño abiertas

- [ ] Auth de brapsoclaw → ruflo cloud: ¿API key en header? ¿JWT?
- [ ] Persona context: estructura exacta del `persona.md` → pendiente de escribir
- [ ] Cómo escala el historial de conversación: ¿ventana deslizante? ¿compresión con /compact?
