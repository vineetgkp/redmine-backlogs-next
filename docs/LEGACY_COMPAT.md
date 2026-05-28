# Legacy `redmine_backlogs` compatibility

This document records how `redmine-backlogs-next` will render data from existing installations of the legacy [redmine_backlogs](https://github.com/backlogs/redmine_backlogs) plugin (now archived). It is the result of a POC-level investigation against the legacy source and the Redmine 5.0 REST API; every claim is citable to a file path or doc URL.

## Why this matters

The new app is architecturally **REST-only** (see [ARCHITECTURE.md](ARCHITECTURE.md)). Stock Redmine REST cannot see plugin-private tables. So before any UI work, we have to know *exactly* which legacy data is reachable, which is not, and what to do about the gap.

## Feature surface of the legacy plugin

From `app/models/` and `app/controllers/` in the legacy repo:

- **Stories** (`rb_story.rb`) and **Tasks** (`rb_task.rb`) — STI subclasses of `Issue`.
- **Sprints** (`rb_sprint.rb`) — STI subclass of `Version`.
- **Releases** + **release multiviews** (`rb_release.rb`, `rb_release_multiview.rb`).
- **Sprint burndown** (`rb_sprint_burndown.rb`) and **per-issue history cache** (`rb_issue_history.rb`).
- **Velocity / scrum stats** (`rb_stats.rb`).
- **Per-project Backlogs settings** (`rb_project_settings.rb`).
- UI surface: master backlog, sprint taskboard, printable cards, per-sprint wiki page.

## Data-model inventory — where each piece physically lives

Storage classes:
- **(A) Stock Redmine primitive** — reachable by REST.
- **(B) Plugin-owned table or raw column add** — NOT reachable by stock REST.
- **(C) Computed** at read time.

| Concern | Storage | Evidence (legacy repo) |
|---|---|---|
| Story (issue + `story_points` + `position`) | **(A)** `issues` table, STI `RbStory < Issue`; extra columns `story_points`, `position`, `remaining_hours` added | `app/models/rb_story.rb`; `db/migrate/011_create_stories_tasks_sprints_and_burndown.rb`, `026_add_story_positions.rb` |
| Task | **(A)** `issues` table, STI `RbTask < Issue`, ordered via nested set | `app/models/rb_task.rb`; `db/migrate/015_order_tasks_using_tree.rb` |
| Sprint | **(A)** `versions` table, STI `RbSprint < Version`; extra columns `sprint_start_date`, `wiki_page_title`, `sharing` | `app/models/rb_sprint.rb`; migration `011` |
| Sprint→Issue link | **(A)** standard `issues.fixed_version_id` | exposed via REST `issues.json` |
| Story→Task hierarchy | **(A)** standard `issues.parent_id` | REST exposes `parent_issue_id` |
| Sprint burndown daily series | **(B)** `rb_sprint_burndown` (`version_id`, serialized `stories`, `burndown`) | `db/migrate/038_rb_add_history.rb` |
| Per-issue history cache | **(B)** `rb_issue_history` (serialized day snapshots) | migration `038`; `app/models/rb_issue_history.rb` |
| Releases | **(B)** `releases` table + `issues.release_id` column | `db/migrate/019_add_release_tables.rb`, `040_add_release_id_to_issues.rb`, `043_add_releases_planned_velocity.rb` |
| Release burndown days | **(B)** `release_burndown_days` | migration `019` |
| Release day-level issue cache | **(B)** `issue_release_day_cache` | `db/migrate/048_add_issues_release_day_cache.rb` |
| Release multiview | **(B)** `release_multiviews` | `db/migrate/049_add_release_multiview.rb` |
| Per-project Backlogs settings | **(B)** `rb_project_settings` | `db/migrate/037_add_rb_project_settings.rb` |
| Velocity / stats | **(C)** computed from sprint + issue + history | `app/models/rb_stats.rb` |

## REST coverage gap

**Reachable today via stock Redmine 5.0 REST:** stories, tasks, parent/child hierarchy, sprint↔version assignment, status, tracker, journals (via `?include=journals`), and any value that was authored as a real Custom Field. References: [Rest_Issues](https://www.redmine.org/projects/redmine/wiki/Rest_Issues), [Rest_Versions](https://www.redmine.org/projects/redmine/wiki/Rest_Versions).

**NOT reachable via stock REST:**

- `issues.position` — story/task ordering, written as a raw integer column. Not exposed by `issues.json`.
- `versions.sprint_start_date` — `versions.json` exposes `due_date` (= `effective_date`) but not Backlogs' added start-date column.
- `issues.story_points` / `remaining_hours` raw columns — invisible unless they were *also* configured as Custom Fields.
- `releases`, `release_burndown_days`, `release_multiviews`, `issue_release_day_cache`, `rb_sprint_burndown`, `rb_issue_history`, `rb_project_settings` — entirely plugin-owned, zero REST surface.

## Options evaluated

### Option 1 — One-time migration script (recommended primary)
Read the legacy Redmine DB once; write recoverable values into stock primitives:
- `issues.position` → integer Custom Field (`cf_backlog_position`).
- `issues.story_points`, `remaining_hours` → integer CFs.
- `versions.sprint_start_date` → date CF on Version.
- `releases` → Versions (with a naming convention or shared parent), copying `planned_velocity` to a CF.

**Pros:** clean API-only architecture afterwards; survives Redmine upgrades; no Redmine-side install in steady state. **Cons:** one-way; lossy for the burndown *caches* (recoverable from journals — see below); CF-id management across multiple projects.

### Option 2 — Read-only compat Redmine plugin (fallback only)
Ship a tiny Rails plugin exposing `rb_*` tables verbatim as JSON (`/rb_compat/sprints.json`, `/issue_positions.json`). Proxy consumes it alongside stock REST.

**Pros:** zero data movement; preserves caches exactly; ~150 LOC. **Cons:** re-introduces the deploy-side plugin we are trying to retire; needs re-test on every Redmine upgrade; still requires a write path for new edits.

### Option 3 — Proxy with direct read-only DB credentials (rejected)
**Pros:** fastest prototype. **Cons:** violates the API-only architectural promise; couples the proxy to Redmine's internal schema; silently breaks on any Redmine column change; doubles the credential blast radius.

## Recommendation — Hybrid, Option 1 primary

1. Migrate the recoverable primitives once (Option 1) so the new SPA is pure REST.
2. **Re-derive burndown on the fly** from `journals` (stock REST exposes them via `?include=journals`) — this is what `rb_sprint_burndown.rb` itself does internally, so the math is recoverable; the daily cache is not load-bearing for correctness, only for speed.
3. Ship a read-only compat plugin (Option 2) **only** if a specific user has unrecoverable Backlogs-specific data that must render verbatim during cutover. Remove it post-migration.

## Status of the legacy plugin (verified against the upstream repo)

- **Last real code commit: 2018-09-01** (`gh api repos/backlogs/redmine_backlogs/commits/master`). The `pushed_at` of 2023-08-17 is a branch-rename housekeeping push, not new code.
- **Not officially archived**, but effectively abandoned (168 open issues, no releases since `v1.0.6`).
- **Supported Redmine versions: ~2.3 through early 4.x.** Evidence in the upstream repo: `init.rb` branches on `Rails::VERSION::MAJOR < 3` and has a `Redmine::VERSION::MAJOR == 2 && MINOR >= 3` gate; `Gemfile` pins `rails ~>3.0`, `nokogiri < 1.6.0` (for Redmine 2.3), `rspec ~> 2.11`. It will **not** install on Redmine 5.0.

## Consequence: migration is inherently cross-version

The migration is not a single-Redmine operation. It moves data from one Redmine to a different, newer one:

```
┌──────────────────────────────┐                ┌─────────────────────────────┐
│ SOURCE                       │                │ TARGET                      │
│ Redmine 3.4 or 4.2           │   migration    │ Redmine 5.0+                │
│ + legacy redmine_backlogs    │ ─────────────▶ │ + pre-created Custom Fields │
│ + rb_* tables                │  (one-shot)    │ + redmine-backlogs-next     │
└──────────────────────────────┘                └─────────────────────────────┘
```

This is actually a better model — anyone retiring legacy `redmine_backlogs` is also upgrading Redmine — and it sharpens the migration script's contract: read-only on the source DB, write-only over REST to the target.

## POC plan (M0.5, ~2–3 days, gated on Docker)

**Prerequisite:** Docker Desktop installed on the host. The POC is not runnable until that's in place.

**The two-container experiment:**

1. **Source container** — Redmine 3.4 (Ruby 2.3) or 4.2 (Ruby 2.6/2.7) with the legacy `redmine_backlogs` plugin installed, MySQL 5.7 backing store, seeded with: 1 project, 5 stories with distinct `position` and `story_points`, 1 sprint with a populated `rb_sprint_burndown` row, optional 1 release.
2. **Target container** — stock Redmine 5.x with pre-created Custom Fields: `cf_backlog_position` (int), `cf_story_points` (int), `cf_remaining_hours` (int), `cf_sprint_start_date` (date on Version).
3. **Migration script** `scripts/migrate_legacy.{py|js}` — opens the source MySQL read-only; for the sample project, reads `issues.position`, `issues.story_points`, `issues.remaining_hours`, `versions.sprint_start_date`; writes each via `PUT /issues/{id}.json` / `PUT /versions/{id}.json` against the **target** Redmine. Language: Node/TS or Python — not Ruby (we do not depend on the legacy Rails stack to migrate off it).
4. **Round-trip verification (ordering & points).** Against the target: `GET /issues.json?project_id=X&include=custom_fields&sort=cf_<position_id>` returns stories in the legacy order with story points intact.
5. **Burndown re-derivation.** For the sprint, `GET /issues/{id}.json?include=journals` for every issue in `fixed_version_id=<sprint>`, reconstruct remaining-SP by day, and compare numerically against the source's `rb_sprint_burndown.burndown` series. Tolerance: exact for completed days; minor drift acceptable for the open day.

**Success criterion:** the SPA can render the sample project's sprint board *and* burndown from the **target** Redmine using only the REST API, with story ordering and points identical to the legacy UI, and burndown matching the legacy cache (within tolerance).

**Failure modes that would change the recommendation:**
- `position` cannot be reliably written/read as a sortable Custom Field → fall back to Option 2 for ordering.
- Source journals are insufficient to reconstruct burndown (e.g. SP changes were never journalled in that era of Redmine) → fall back to Option 2 for burndown.
- Legacy plugin does not run on a Ruby/Rails combination buildable today on linux/amd64 → narrow the supported source range further and document.

## Out of scope for this POC

- Write-path for *new* edits (drag-drop) — that is M3, after the read-only board ships.
- Multi-project migration ergonomics — addressed if and when the single-project POC passes.
- Printable cards, sprint wiki pages — deferred; not on the critical path.
