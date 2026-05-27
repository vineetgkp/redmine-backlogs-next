# Architecture

`redmine-backlogs-next` is a standalone application that presents an Agile/Scrum view of a Redmine instance using only Redmine's REST API. It is deliberately decoupled from Redmine's internals.

## Components

```
┌──────────────┐      HTTPS/JSON      ┌──────────────┐   Redmine REST   ┌──────────────┐
│   web/ (SPA) │  ───────────────────▶│  proxy/      │ ────────────────▶│   Redmine    │
│ React + Vite │ ◀─────────────────── │ (aggregator) │ ◀──────────────── │   5.0+       │
│ + TypeScript │                      │ holds API key│                   │              │
└──────────────┘                      └──────────────┘                   └──────────────┘
```

### `web/` — front-end SPA
- React + Vite + TypeScript.
- Renders the sprint board, backlog, and burndown/velocity charts.
- Holds **no** Redmine credentials. Talks only to the proxy.

### `proxy/` — backend aggregator
- **Backend language: undecided** (Node/Express or PHP/Slim). The contract below is language-agnostic.
- Responsibilities:
  - **Holds the Redmine API key server-side** — the key is never shipped to the browser.
  - **Solves CORS** — Redmine has no usable CORS support, so the browser cannot call it directly.
  - **Aggregates & caches** — collapses the chatty, N+1 REST pattern (e.g. one call per version, per issue page) into a few task-shaped endpoints, with short-TTL caching.

### Why a proxy is mandatory
A pure browser-only SPA is not viable: (1) Redmine does not emit permissive CORS headers, and (2) the API key would be exposed client-side. The proxy is the smallest thing that fixes both.

## Redmine data-model mapping

| Backlogs concept | Redmine primitive               | API source |
|------------------|----------------------------------|------------|
| Sprint           | Version                          | `GET /projects/:id/versions.json` |
| Sprint contents  | Issues with `fixed_version_id`   | `GET /issues.json?fixed_version_id=…` |
| Backlog          | Issues with no `fixed_version_id`| `GET /issues.json?fixed_version_id=!*` (or status filter) |
| Board columns    | Issue statuses                   | `GET /issue_statuses.json` |
| Story points     | Issue custom field (integer)     | `custom_fields[]` on each issue |
| Burndown series  | Issue journals (status/SP changes over time) | `GET /issues/:id.json?include=journals` |
| Velocity         | Completed SP per closed Version  | derived: sum SP of closed issues per closed version |

Burndown requires the journal history that Redmine 5.0+ exposes reliably; it is the trickiest piece and is deferred to a later milestone.

## Proxy API surface (target)

Task-shaped, not a 1:1 mirror of Redmine:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/health` | liveness + Redmine reachability + auth check |
| `GET /api/sprints?project=:id` | versions for a project, shaped as sprints |
| `GET /api/sprints/:versionId/board` | issues for one sprint, grouped by status column, with SP |
| `GET /api/backlog?project=:id` | unscheduled, point-estimated stories, ordered |
| `PATCH /api/issues/:id` *(M3+)* | update status / version (drag-drop round-trip) |
| `GET /api/sprints/:versionId/burndown` *(M4)* | remaining-SP time series from journals |
| `GET /api/projects/:id/velocity` *(M4)* | completed SP per closed version |

## Configuration

Provided to the **proxy** via environment (`.env`, never committed):

| Var | Meaning |
|-----|---------|
| `REDMINE_URL` | base URL of the target Redmine, e.g. `http://localhost:3000` |
| `REDMINE_API_KEY` | server-side API key (My account → API access key) |
| `STORY_POINTS_FIELD_ID` | id of the integer custom field used for story points |
| `CACHE_TTL_SECONDS` | aggregation cache lifetime (optional) |
| `PORT` | proxy listen port |

The front-end only needs the proxy's base URL.

## Constraints & assumptions
- Target Redmine **5.0+** (for reliable journal/REST behaviour).
- REST API and a project API key must be enabled in Redmine (*Administration → Settings → API*).
- Story points live in a single integer custom field; its id is configured, not hard-coded.
- The app is **read-mostly** through M2; writes (M3) go exclusively through the proxy via REST — never direct DB access.
