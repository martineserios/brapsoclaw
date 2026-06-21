<!-- brana-branching-v1 -->
## Branching — two-tier `dev → main` (brana ADR-060)

Branch naming: `{epic-slug}/{work-type}/t-{NNN}-{description-slug}`
(work-type ∈ `feat` `fix` `chore` `docs` `test` `refactor` `review`).
Example: `backend/feat/t-1109-chess-api-ingestion`.

- Feature branches are cut **off `dev`** and merge back to `dev` (`--no-ff`); worktrees, short-lived.
- **`dev`** = integration buffer (nothing live). **`main`** = production.
- **Never commit to `main` directly / never merge a feature into `main`.** `main` lagging `dev` is the safety buffer.
- **Ship** (periodic, human-gated):
  ```bash
  git checkout main && git merge --ff-only dev
  # <your production deploy command> # ship deploy — fill in
  git push origin main dev && git checkout dev
  ```
  Deploy only at ship, never from `dev`.
