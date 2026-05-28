# redmine-backlogs-next

Agile/Scrum companion for [Redmine](https://www.redmine.org/) built entirely on its REST API — sprint board, backlog & burndown. A ground-up, API-decoupled spiritual successor to the original [redmine_backlogs](https://github.com/backlogs/redmine_backlogs) plugin, with **no coupling to Rails internals**.

## Why

The original `redmine_backlogs` plugin hooked deep into Redmine's Rails models and views. Every Redmine upgrade risked breaking it, and it could only run co-located with the Redmine install. `redmine-backlogs-next` instead talks to Redmine purely over its [public REST API](https://www.redmine.org/projects/redmine/wiki/Rest_api), so it:

- Runs as a **separate application** — deploy it anywhere, point it at any Redmine 5.0+.
- Survives Redmine upgrades — it depends on a stable API surface, not internal classes.
- Stays read-mostly and safe — it never touches the database directly.

## What it does

- **Sprint board** — Redmine *Versions* rendered as sprints; issues grouped into columns by status, with story points.
- **Backlog** — ordered, point-estimated list of stories not yet assigned to a sprint.
- **Burndown & velocity** — remaining story points over time (from issue journals) and completed points per closed sprint.

## How it maps onto Redmine

| Backlogs concept | Redmine primitive          |
|------------------|-----------------------------|
| Sprint           | Version                     |
| Story points     | Issue custom field (integer)|
| Burndown series  | Issue journals (history)    |
| Velocity         | Completed SP per closed Version |

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full mapping and component design, [docs/MVP.md](docs/MVP.md) for the milestone plan, and [docs/LEGACY_COMPAT.md](docs/LEGACY_COMPAT.md) for how data from the original `redmine_backlogs` plugin is migrated and rendered.

## Status

Early. Docs-first; the proxy backend language and code skeleton are not yet committed. See the roadmap in [docs/MVP.md](docs/MVP.md).

## Quickstart (dev Redmine)

You need a Redmine to develop against:

```bash
docker run -d -p 3000:3000 redmine
```

Then, in the Redmine UI:

1. Create a project and a few issues.
2. Add a **Story Points** custom field (integer, applied to issues).
3. Create 2–3 **Versions** — these become your sprints.
4. Generate an API key: *My account → API access key*.

That instance + key is the target the application develops against. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for required config.

## License

[MIT](LICENSE).
