# MVP & Roadmap

Built as **vertical slices** — each milestone is independently shippable and demonstrable, rather than building all layers horizontally before anything works.

## Guiding principle
Prove the riskiest unknowns first (auth + data-model mapping, then journals/burndown), and resist cloning all of the original Backlogs plugin at once.

## Milestones

### M0 — Scaffold (~½ day)
- Repo structure: `web/`, `proxy/`, `docs/`.
- Docs: README, [ARCHITECTURE.md](ARCHITECTURE.md), this MVP spec. ✅ (docs done; code skeleton pending stack decision)
- `docker-compose.yml` for a local dev Redmine.
- Config scaffolding: `.env.example` with `REDMINE_URL`, `REDMINE_API_KEY`, `STORY_POINTS_FIELD_ID`.

### M0.5 — Legacy compat POC (~2–3 days, needs Docker)
Prove that data from existing legacy `redmine_backlogs` installs can be rendered through the new REST-only architecture. Plan and rationale: [LEGACY_COMPAT.md](LEGACY_COMPAT.md). Hybrid approach: one-time **cross-version** migration (source = Redmine 3.x/4.x + legacy plugin; target = Redmine 5.0+ stock) of recoverable raw columns (`position`, `story_points`, `remaining_hours`, `sprint_start_date`, releases) into stock Custom Fields, plus on-the-fly burndown re-derivation from issue journals. Two-container Docker setup; legacy plugin frozen at the Redmine 4.x era (last code commit 2018). **Gates M1.**

### M1 — Proxy ↔ Redmine (~1 day)
- Proxy connects to the Redmine REST API with the server-side key.
- Fetches projects / versions / issues.
- Smoke endpoints: `GET /api/health`, `GET /api/sprints`.
- **De-risks the hardest part first**: auth + the sprint→version / SP→custom-field mapping.

### M2 — Read-only Sprint Board
- Versions rendered as sprints.
- Issues grouped into columns by status.
- Story points shown on each card.
- **First visible win.**

> **v0.1 cut line = M0–M2** — a working read-only board. Ship it, gather feedback, then continue.

### M3 — Write / interactive
- Drag-drop a card → update issue status and/or version, round-tripped to Redmine via the proxy (`PATCH /api/issues/:id`).

### M4 — Burndown & velocity
- Remaining-SP time series computed from issue journals (`GET /api/sprints/:id/burndown`).
- Velocity = completed SP per closed version.
- Trickiest milestone; depends on Redmine 5.0+ journal history.

## Out of scope (for now)
- Full feature parity with the legacy `redmine_backlogs` plugin (wikis, task boards beyond the sprint board, printable cards).
- Multi-Redmine federation.
- Auth/SSO for the app itself beyond a single configured API key.

## Open decisions
- **Proxy backend language** — Node/Express (TS, one language across the stack, shared types) vs PHP (Slim/Laravel). Front-end is React + Vite + TypeScript regardless. *Decision deferred; docs are stack-agnostic.*
