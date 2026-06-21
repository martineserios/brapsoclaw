# ADR-002: Two-Tier `dev → main` Branch Strategy

**Date:** 2026-06-21
**Status:** accepted
**Deciders:** Martín
**Relates to:** brana ADR-060 (branch strategy for autonomous agents), thebrana `docs/guide/workflows/branching.md`, proyecto-anita ADR-057 (sibling adoption), thebrana t-2189 (cross-repo rollout)

## Context

`ventures/brapsoclaw` adopted brana's two-tier branch model during the t-2189 cross-repo rollout
(2026-06-21). Previously every feature merged straight to `main` (production) with no
integration buffer, and the branch-naming convention was undocumented in this repo.

This ADR records the decision **for this repo**. ADR-060 deliberately does not mandate the
topology portfolio-wide — each repo decides — and this repo opts in.

## Decision

### Branch naming

`{epic-slug}/{work-type}/t-{NNN}-{description-slug}`
work-type ∈ `feat` `fix` `chore` `docs` `test` `refactor` `review`.

### Two tiers

| Branch | Role | How it advances |
|--------|------|-----------------|
| feature (`{epic}/{type}/t-NNN-slug`) | one unit of work | branched **off `dev`**; worktrees, short-lived |
| **`dev`** | integration buffer — nothing here is live | features merge in (`--no-ff`); no deploy |
| **`main`** | **production** | advances **only at ship**: `dev→main` fast-forward, then deploy |

`main` lagging `dev` is the safety buffer, not drift to close eagerly.

### Rules

- **Never commit to `main` directly, never merge a feature into `main`.** Features integrate to `dev`.
- **Ship** (periodic, human-gated):
  ```bash
  git checkout main && git merge --ff-only dev
  # <your production deploy command>                          # production deploy
  git push origin main dev && git checkout dev
  ```
  If `--ff-only` is rejected, `main` was touched directly — stop and investigate, don't force.
- **Deploy is a ship-time action only** — never from `dev`.

## Consequences

- `main` is always a coherent, deployed-and-known-good snapshot; integration risk is absorbed on `dev`.
- Aligns this repo with brana practice; the shared `/brana:build` CLOSE step integrates to `dev` unmodified.
- Cost: one extra `dev→main` promotion per ship. The shared skills' ship step references thebrana's
  `bootstrap.sh`; here substitute `# <your production deploy command>`.

## Migration

- `dev` branch created from `main` HEAD (2026-06-21); `CLAUDE.md` carries the branching pointer (on `dev`).
- Legacy flat-named branches grandfathered — renamed opportunistically, not en masse.
